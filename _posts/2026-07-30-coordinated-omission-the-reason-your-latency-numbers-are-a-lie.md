---
title: "Coordinated Omission: The Reason Your Latency Numbers Are a Lie"
date: 2026-07-29T10:00:00-04:00
tagline: "How a default load testing pattern made our latency numbers look better than they were, and what we did to fix it."
classes: wide
images:
  path: /assets/images/unsplash-image-4.jpg
header:
  overlay_image: /assets/images/unsplash-image-4.jpg
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

If you've ever worked with Kafka in production, you’ve probably dealt with consumer group rebalancing at some point. Most of the time, it’s just background noise. But every now and then, it turns into a full-blown operational headache, especially when your processes have a task that is long.

We ran into this exact scenario when building out a feature that exports large datasets into files on demand. At first glance, it sounded simple: a consumer reads a message, gathers some data, writes it to a file, and stores it on a network path for download. The issue? Some of these tasks could run for close to thirty minutes.

## Where Things Went Wrong
In our Kafka setup, each export request lands on a topic. A group of consumers picks them up and starts processing. Nothing special, until a task drags on longer than expected. Kafka expects each consumer to poll the broker at regular intervals. By default, that interval (`max.poll.interval.ms`) is set to five minutes.