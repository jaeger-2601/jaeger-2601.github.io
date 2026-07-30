---
title: "The Hidden Race in Our Data Export Pipeline"
date: 2026-02-22T10:00:00-04:00
tagline: "How a subtle concurrency bug in our export pipeline led to data corruption and what we changed to fix it."
classes: wide
images:
  path: /assets/images/unsplash-image-2.jpg
header:
  overlay_image: /assets/images/unsplash-image-2.jpg
  caption: "Photo credit: [**Jonathan Chng**](https://unsplash.com/@jon_chng)"
  overlay_filter: 0.5
  actions:
    - label: "More Info"
      url: "https://docs.confluent.io/kafka/design/consumer-design.html?utm_source=chatgpt.com#consumer-groups-and-group-ids"
categories:
  - Distributed Systems
tags:
  - Distributed Systems
  - Kafka
  - Concurrency
  - System Design
  - Redis
---

It started with a file that was slightly larger than it should have been. There were no crashes and no alerts, just an export that didn't look quite right. When we opened it, we found duplicated sections and content that had been partially repeated, and at first it felt like nothing more than bad data or a serialization issue, but the deeper we dug, the stranger it became.

Our platform supports exporting large datasets on demand. A client requests an export, we generate a job identifier, process it asynchronously, and write the final file to a network-attached storage path, and once that's done we update the status and return the file location. The pipeline was built on a Redis-based worker model, with jobs pushed to a queue and workers polling and processing them in the background, which is simple and practical and had been stable enough under normal load.

But after correlating worker logs and job timestamps in production, a pattern emerged: the same job was being processed more than once. The system wasn't crashing so much as it was racing, and what we'd uncovered was a subtle race condition hiding inside what looked like a perfectly safe Redis worker model.

---

### How the Race Happened in the Redis Worker Model

![Figure 1: Two workers observing the same job before acknowledgment](/assets/images/component_diagram_v2.svg){: .align-center}

The export process runs through several logical stages. A worker fetches data from internal services, transforms and aggregates the dataset, streams the output into a file, and finally updates the job status to mark it complete. Each stage worked correctly on its own, so the failure was never inside a single stage but at the boundary between worker executions.

Under production load, two scenarios were enough to trigger the issue. A job could time out and get retried while the original execution was still running, or a worker could crash after writing part of the file but before it managed to update the job status.

The Redis worker implementation followed a pattern where a worker would read a job identifier off the queue, begin processing it, and only remove it from the queue once the work was done. That gap between reading and acknowledging left a window where a second worker could observe and pick up the same job, and if the first worker crashed before removing it, the job would just sit there visible in the queue for someone else to grab.

![Figure 2: Visualize overlapping execution windows.](/assets/images/timing_window_v2.svg){: .align-center}

Because both executions were writing to the same file path, their writes interleaved. The filesystem allowed the concurrent writes without complaint, and from its perspective nothing illegal had happened, but from the application's perspective the output came out corrupted, full of duplicated or truncated data.

The core issue came down to a lack of atomic ownership. Observing a job had quietly become equivalent to claiming it, and the system never actually enforced that exclusivity.

---

### Moving to Kafka for Deterministic Ownership

![Figure 3: Partition-level ownership in Kafka consumer groups](/assets/images/kafka_ownership_diagram.svg){: .align-center}

To get rid of the issue for good, we redesigned the pipeline around Kafka consumer groups. Jobs are now published to a Kafka topic and consumed by workers belonging to a consumer group, and because Kafka guarantees that each partition is assigned to exactly one consumer at a time, message delivery itself becomes the atomic claim operation. Workers no longer have to compete through polling and the visibility gaps that came with it.

When a worker starts processing a job, it updates the job state in persistent storage to RUNNING along with some metadata: the worker identifier, a start timestamp, and the execution attempt number. That metadata turned out to be critical for crash detection and recovery later on.

If a worker crashes mid-execution, Kafka will eventually trigger a rebalance and hand the partition to another worker. When that new worker receives the same job message, it doesn't just blindly execute it again. It checks the persisted job state first, and if the job is marked RUNNING but the associated worker hasn't sent a heartbeat within the expected interval, the system treats that execution as abandoned.

We built a lightweight heartbeat mechanism where workers periodically update a timestamp while they're processing long-running exports, and once that timestamp goes stale past a configured threshold, the job becomes eligible for recovery. The new worker then transitions the job into a RETRY state through a conditional update and starts a fresh execution attempt.

That combination is what keeps duplicate active executions from happening while still letting the system recover automatically from crashes.

---

### Dead Letter Queue and Intelligent Failure Handling

![Figure 4: Dead Letter Queue and Intelligent Failure Handling](/assets/images/heartbeat_recovery_diagram.svg){: .align-center}

We also added a Dead Letter Queue to handle the failures that were never going to recover on their own. If a job fails repeatedly past a configured retry threshold, it stops getting reprocessed indefinitely and instead gets published to a dedicated DLQ topic along with diagnostic metadata like the failure reason and the attempt count.

That keeps poison messages from blocking partitions or triggering infinite retry loops, and it lets operators go inspect the failed exports on their own without touching healthy traffic.

Because job state is persisted independently of message delivery, workers can figure out whether a given message represents a fresh execution, a retry after a crash, or a previously failed attempt, rather than just assuming that redelivery means it's safe to re-execute. The system checks ownership and state before it does anything with real side effects.

---

### Safe File Writing Under Recovery

Even with deterministic ownership and crash detection in place, we still redesigned how files get written so we could rule out partial corruption entirely.

Workers now write to a temporary file that includes a unique execution identifier instead of writing straight to the final network path, so if a worker crashes mid-write, that temporary file stays isolated and doesn't touch any exports that had already completed. When a new worker picks up the recovery of a job, it either resumes from a clean state or overwrites the temporary artifact left behind by the abandoned execution.

Only once the export finishes successfully does the temporary file get atomically renamed to its final path. Since the rename is atomic at the filesystem level, the final file only becomes visible once it's fully written, never in some half-finished state.

Between the atomic message ownership, the persistent execution state, the heartbeat-based crash detection, the DLQ isolation, and the atomic file writes, we managed to close off the hidden race that had been living in the original Redis model, and the export pipeline has held up under real production failure conditions ever since.