---
title: "The Hidden Race in Our Data Export Pipeline"
date: 2026-03-22T10:00:00-04:00
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

Our platform supports exporting large datasets into files on demand. The flow is asynchronous. A client requests an export, we generate a job identifier, process the request in the background, and store the resulting file on a network attached storage path. Once the job completes, we update the status and return the file location to the client.

The original implementation of this pipeline was built on a Redis based worker model. Jobs were pushed into a Redis queue and a pool of workers would poll the queue, pick up job identifiers, and generate the corresponding files. The design looked simple and practical, and it worked well under normal load.

Over time we started noticing a strange issue in production. Some exported files were larger than expected and in certain cases the size was more than double what it should have been. When we opened the files we saw duplicated sections and partially repeated content. There were no crashes visible in service logs and no obvious error signals. The problem only became clear after tracing job execution across workers and correlating timestamps. What we discovered was that a race condition existed inside the Redis worker model itself.

---

### How the Race Happened in the Redis Worker Model

The export process consists of multiple logical stages. A worker fetches data from internal services, transforms and aggregates the dataset, streams the output into a file, and finally updates the job status to mark it as complete. Each stage functions correctly in isolation. The failure did not originate inside a single stage but at the boundary between worker executions.

Under production load two scenarios triggered the issue. A job could time out and be retried while the original execution was still running. In another case a worker could crash after writing part of the file but before updating the job status.

The Redis worker implementation followed a common pattern where a worker would peek at the queue, retrieve a job identifier, start processing it, and once finished remove it from the queue. The problem lies in the gap between peeking and claiming.

Between the moment a worker reads a job and the moment it removes it from the queue, another worker can also peek at the same job. Both workers then begin processing the same job concurrently. Because both executions target the same file path, their writes interleave. The filesystem allows concurrent writes, so from its perspective nothing illegal happens. From the application perspective however the output becomes corrupted, containing duplicated or truncated data.

This behavior was difficult to reproduce in controlled testing because it required precise timing and overlapping retries. Under light load everything appeared correct. Under real production traffic with failures and retries, the race surfaced.

Although the system appeared distributed and safe at first glance, it lacked an atomic ownership guarantee. The queue allowed visibility into work but did not guarantee exclusive ownership. We treated seeing a job as equivalent to claiming a job, but those are fundamentally different semantics. Retries could start while the original execution was still running. Crashes could leave partially completed work behind. Multiple workers could overlap on the same job. Ownership was implicit and coordinated by convention instead of enforced by the system, and that gap created the race condition.

---

### Moving to Kafka for Stronger Ownership

To eliminate the issue we redesigned the pipeline using Kafka consumer groups. Instead of workers polling and peeking at a shared queue, jobs are published to a Kafka topic and consumed by workers that belong to a consumer group.

Kafka guarantees that each partition is assigned to exactly one consumer within the group at a time. A message is delivered to one consumer, and if that consumer fails the partition is reassigned. At no point are two workers processing the same partition simultaneously.

This model provides stronger ownership semantics. It removes the ambiguity that existed in the Redis model because message delivery itself becomes the atomic claim operation. Workers no longer compete for the same job through polling and peeking. The system enforces exclusivity at the infrastructure level.

---

### Making File Writes Safe

Even with stronger ownership at the queue layer, we also improved how files were written to eliminate partial corruption risks.

Workers now write to a temporary file instead of writing directly to the final network path. The temporary file includes a unique execution identifier and the full generation process happens there. Only after the job completes successfully is the temporary file atomically renamed to the final path.

The rename operation is atomic at the filesystem level. This guarantees that even if a worker crashes during execution or a retry happens mid-process, it cannot corrupt the final output file. We also introduced cleanup logic to remove stale temporary files left behind by failed executions so that storage does not accumulate orphaned artifacts over time.

---

### Outcome and Lessons

After migrating from the Redis worker model to Kafka-based processing and enforcing atomic file writes, the race condition disappeared.

More importantly, the pipeline became resilient to retries, worker crashes, and overlapping execution attempts. What initially looked like a simple corruption bug exposed a deeper architectural gap around ownership and state transitions.

The key lesson was that a queue that allows workers to observe jobs does not automatically guarantee exclusive execution. True correctness in distributed job pipelines requires atomic ownership semantics at the moment a job is claimed. The bug was not caused by Kafka. Kafka solved it. The root issue was relying on a non-atomic claim mechanism in the Redis model.

By replacing implicit ownership with explicit system-level guarantees and isolating file writes from live mutations, we transformed a fragile pipeline into one that behaves predictably under failure.