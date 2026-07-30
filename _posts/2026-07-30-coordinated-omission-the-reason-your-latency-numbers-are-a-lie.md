---
title: "Coordinated Omission: The Reason Your Latency Numbers Are a Lie"
date: 2026-07-29T10:00:00-04:00
tagline: "How a default load testing pattern made our latency numbers look better than they were, and what we did to fix it."
classes: wide
images:
  path: /assets/images/unsplash-image-3.jpg
header:
  overlay_image: /assets/images/unsplash-image-3.jpg
  caption: "Photo credit: [**Itadaki**](https://unsplash.com/@itadakidesu)"
  overlay_filter: 0.5
  actions: 
    - label: "More Info"
      url: "https://qconsf.com/sf2012/dl/qcon-sanfran-2012/slides/GilTene_HowNotToMeasureLatency.pdf"
categories:
  - Performance Engineering
tags:
  - Locust
  - Load Testing
  - Latency
  - Scalability
  - Performance Engineering
---

Coordinated omission is a measurement problem that shows up in a lot of load testing setups, and it has a habit of making latency numbers look much better than they actually are. The rough idea is that when your service slows down, your load generator stops sending requests and waits, so all the requests that would have piled up duriing the slow period never got recorded. We end up measuring the system on its good behavior and missing most of the bad.

Our team ran into this issue the hard way. We had a load test telling our p95 was around 350ms but actual production data something closer to 1.2 seconds under similar load. We spent a decent amount of time assuming that something in the service had regressed in production environment and dug around in the code for a while without finding anything that could explain a gap that large. It took us longer to realize the problem was with how our load testing was counting latency rather than the service itself.

---
For some context on our load testing setup, we were running Locust in distributed mode using Locust Swarm, with a master process and worker processes that we could scale up or down depending on the load we wanted to generate. Generating maximum load mattered because our API sees close to 100k requests per minute at peak. On top of the volume, we have a fairly strict SLA on latency, so the tail latency was not something we can hand wave away. A typical load test task looks something like this:

```python
from locust import HttpUser, task

class ApiUser(HttpUser):

    @task
    def get_item(self):
        self.client.get("/items/42")
```

Each simulated user runs its task, waits for the response, and then goes again. That loop is the whole problem. A user does not fire its next request until the previous one has come back. So, if the the server stalls for two seconds, that user is just sitting there, blocked, and not generating any load during the exact window you most want to observe.