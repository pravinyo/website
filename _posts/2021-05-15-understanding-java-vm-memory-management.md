---
title: "Understanding Java VM for Memory Management"
author: pravin_tripathi
date: 2021-05-15 00:00:00 +0530
readtime: true
categories: [Blogging, Article]
tags: [java, docker-compose, kafka, performance-tuning, jvm, backend]
img_path: /assets/img/understanding-java-vm-memory-management/
image:
  path: header.jpg
  width: 1000   # in pixels
  height: 400   # in pixels
  alt: Photo by www.oracle.com
---

I had created a demo application in my free time which uses pub/sub for sharing messages between 2 java microservice application using Kafka as middleware. Later on, `I starter experimenting on effect of different garbage collection algorithm using some popular tools.`

After trying out different algorithm, `I am surprised that same code could get faster if right garbage collection algorithm is configured` for the application.

I have created an article to document same thing.

> **Full article was published on LinkedIn,** [link to LinkedIn blog article][article-link]
{: .prompt-info }

&nbsp;
> You have something to share, go ahead and add in comments 😉 » Happy Learning!!
{: .prompt-tip }

[article-link]: https://www.linkedin.com/pulse/understanding-java-vm-memory-management-pravin-tripathi/
