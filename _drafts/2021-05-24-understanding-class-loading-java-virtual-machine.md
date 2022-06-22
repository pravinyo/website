---
title: "Understanding Class Loading in Java Virtual Machine"
author: pravin_tripathi
date: 2021-05-24 00:00:00 +0530
readtime: true
categories: [Blogging, Article]
tags: [java, security, backenddevelopment]
---

I had a use case where I wanted to `update the implementation` of the algorithm based on the trigger `without restarting the app`. This app is written using java(spring) and it exposes API to be consumed. Here, **whenever the configuration is changed in the repository, the app is notified to reload the code**. `Java allows us to use our implementation of the class loader` and using this, I implemented the required functionality. 

This article is created to explain the class loader and how we can use it to create our custom implementation and perform hot deployment without any app downtime.

> **Full article was published on LinkedIn,** [link to LinkedIn blog article][article-link]
{: .prompt-info }

&nbsp;
> You have something to share, go ahead and add in comments 😉 » Happy Learning!!
{: .prompt-tip }

[article-link]: https://www.linkedin.com/pulse/understanding-class-loading-java-virtual-machine-pravin-tripathi/
