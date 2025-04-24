---
title: "Understaning RAFT - distributed consensus protocol"
author: pravin_tripathi
date: 2023-05-07 00:00:00 +0530
readtime: true
media_subpath: /assets/img/understaning-raft-distributed-consensus-protocol/
categories: ["Presentation", "system-design"]
tags: [design, backenddevelopment, softwareengineering]
image:
  path: header.jpg
  width: 1000   # in pixels
  height: 400   # in pixels
  alt: Photo by Chastagner Thierry on Unsplash
file_document_path: "/assets/document/presentation/understanding-raft-distributed-consensus-protocol/RAFT - Distributed consensus protocol.pdf"
toc: false
---

## RAFT - Distributed Consensus Protocol

In this presentation, I talked about the RAFT consensus algorithm which is commonly used in large different systems to achieve consensus in a distributed environment. Many systems uses a custom implementation of the RAFT algorithm that is understandable and easy to implement compared to the Paxos algorithm for distributed consensus protocol.

We will start from the basic understanding of the algorithm and later see an issue with this algorithm. We will also talk on recent incident that caused an internet outage for many people and businesses.

## Deck
{% include presentation.html %}

## Diagram
![All Diagram](raft-presentation-diagram.svg){: width="700" height="400" }
_download and view svg file in separate viewer._

## Further readings
### Blogs
- [A Byzantine failure in the real world](https://blog.cloudflare.com/a-byzantine-failure-in-the-real-world/)
- [Consistent Core](https://martinfowler.com/articles/patterns-of-distributed-systems/consistent-core.html)
- [Generation Clock](https://martinfowler.com/articles/patterns-of-distributed-systems/generation.html)
- [Emergent leader](https://martinfowler.com/articles/patterns-of-distributed-systems/emergent-leader.html)
- [Heartbeat](https://martinfowler.com/articles/patterns-of-distributed-systems/heartbeat.html)

### Video:
- ["Raft - The Understandable Distributed Protocol" by Ben Johnson (2013)
](https://www.youtube.com/watch?v=ro2fU8_mr2w)