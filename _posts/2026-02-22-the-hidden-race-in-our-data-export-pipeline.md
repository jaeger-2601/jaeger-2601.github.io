---
title: "The Hidden Race in Our Data Export Pipeline"
date: 2026-02-22T10:00:00-04:00
tagline: "How a subtle concurrency bug in our export pipeline led to inconsistent outputs — and what we changed to fix it."
classes: wide
images:
  path: /assets/images/unsplash-image-2.jpg
header:
  overlay_image: /assets/images/unsplash-image-2.jpg
  caption: "Photo credit: [**Jonathan Chng**](https://unsplash.com/@jon_chng)"
  overlay_filter: 0.5
categories:
  - Distributed Systems
tags:
  - Distributed Systems
  - Kafka
  - Concurrency
  - System Design
  - Redis
---

## The Hidden Race in Our Data Export Pipeline

Our platform supports exporting large datasets into files on demand. The flow is asynchronous. A client requests an export, we generate a job identifier, process the request in the background, and store the resulting file on a network attached storage path. Once the job completes, we update the status and return the file location to the client.

The original implementation of this pipeline was built on a Redis based worker model. Jobs were pushed into a Redis queue and a pool of workers would poll the queue, pick up job identifiers, and generate the corresponding files. The design looked simple and practical, and it worked well under normal load.

Over time we started noticing a strange issue in production. Some exported files were larger than expected and in certain cases the size was more than double what it should have been. When we opened the files we saw duplicated sections and partially repeated content. There were no crashes visible in service logs and no obvious error signals. The problem only became clear after tracing job execution across workers and correlating timestamps. What we discovered was that a race condition existed inside the Redis worker model itself.

---

### How the Race Happened in the Redis Worker Model

The export process consists of multiple logical stages. A worker fetches data from internal services, transforms and aggregates the dataset, streams the output into a file, and finally updates the job status to mark it as complete. Each stage functions correctly in isolation. The failure did not originate inside a single stage but at the boundary between worker executions.

Under production load two scenarios triggered the issue. A job could time out and be retried while the original execution was still running. In another case a worker could crash after writing part of the file but before updating the job status.

The Redis worker implementation followed a pattern where a worker would read a job identifier from the queue, begin processing it, and only remove it from the queue after completion. The gap between reading and acknowledging created a window where another worker could observe and process the same job. If the first worker crashed before removing the job, it would remain visible in the queue and another worker would pick it up.

Because both executions targeted the same file path, their writes interleaved. The filesystem allowed concurrent writes and from its perspective nothing illegal happened. From the application perspective however the output became corrupted, containing duplicated or truncated data.

The core issue was the lack of atomic ownership. Observing a job was treated as equivalent to claiming a job, but the system never enforced exclusivity.

---

### Moving to Kafka for Deterministic Ownership

To eliminate the issue we redesigned the pipeline using Kafka consumer groups. Jobs are now published to a Kafka topic and consumed by workers that belong to a consumer group. Kafka guarantees that each partition is assigned to exactly one consumer within the group at a time. Message delivery becomes the atomic claim operation. Workers no longer compete through polling and visibility gaps.

When a worker begins processing a job, it updates the job state in persistent storage to RUNNING along with metadata such as worker identifier, start timestamp, and execution attempt number. This metadata becomes critical for crash detection and recovery.

If a worker crashes mid-execution, Kafka will eventually trigger a rebalance and reassign the partition to another worker. When the new worker receives the same job message, it does not blindly execute it. Instead, it first checks the persisted job state. If the job is marked as RUNNING but the associated worker has not heartbeated within an expected interval, the system considers that execution abandoned.

We implemented a lightweight heartbeat mechanism where workers periodically update a timestamp while processing long running exports. If that timestamp becomes stale beyond a configured threshold, the job is eligible for recovery. The new worker then safely transitions the job to a RETRY state using a conditional update and begins a new execution attempt.

This prevents duplicate active executions while still allowing automatic recovery from crashes.

---

### Dead Letter Queue and Intelligent Failure Handling

We also introduced a Dead Letter Queue to handle irrecoverable failures. If a job fails repeatedly beyond a configured retry threshold, it is no longer reprocessed indefinitely. Instead, the job is published to a dedicated DLQ topic along with diagnostic metadata including the failure reason and attempt count.

This prevents poison messages from blocking partitions or causing infinite retry loops. It also allows operators to inspect failed exports separately without impacting healthy traffic.

Because job state is persisted independently of message delivery, workers can intelligently determine whether a message represents a fresh execution, a retry after crash, or a previously failed attempt. The system no longer assumes that message redelivery implies safe re-execution. It verifies ownership and state before performing side effects.

---

### Safe File Writing Under Recovery

Even with deterministic ownership and crash detection, we redesigned the file writing strategy to eliminate partial corruption risks.

Workers write to a temporary file that includes a unique execution identifier rather than writing directly to the final network path. If a worker crashes mid-write, the temporary file remains isolated and does not affect previously completed exports. When a new worker recovers the job, it either resumes from a clean state or overwrites the temporary artifact associated with the abandoned execution.

Only after the export completes successfully is the temporary file atomically renamed to the final path. The rename operation is atomic at the filesystem level, ensuring that the final file becomes visible only when fully written.

This combination of atomic message ownership, persistent execution state, heartbeat based crash detection, DLQ isolation, and atomic file writes eliminated the hidden race that existed in the original Redis model and made the export pipeline resilient under real production failure conditions.