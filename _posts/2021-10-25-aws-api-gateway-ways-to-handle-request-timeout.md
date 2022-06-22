---
title: "AWS API Gateway — Ways to handle Request Timeout"
author: pravin_tripathi
date: 2021-10-25 00:00:00 +0530
readtime: true
img_path: /assets/img/aws-api-gateway-ways-to-handle-request-timeout/
categories: [Blogging, Article]
tags: [aws, softwarearchitecture, backenddevelopment]
image:
  path: header.png
  width: 1000   # in pixels
  height: 400   # in pixels
  alt: Image by Pravin Tripathi (pravin.dev) - for better image view, switch theme to light mode
---

AWS cloud services are improving with time, and I am enjoying using their services. We all know that using 3rd part services in our application adds additional constraints to our services. Compared to benefits, those restrictions may sometimes be manageable. 

## The Problem:
I have come across a problem where API request timeout is restricted due to vendor (AWS in my case) and can’t increase after max timeout value (30 sec). Daily multiple batches ran by different API consumers that experienced timeout for some requests amounting to 3–4% of the total request sent.

I have discussed above issue in depth in below article 😉.

> **Full article was published on Medium,** [link to Medium blog article][medium-article-link]
{: .prompt-info }

&nbsp;
> You have something to share, go ahead and add in comments 😉 » Happy Learning!!
{: .prompt-tip }

[medium-article-link]: https://medium.com/@pravinyo/aws-api-gateway-ways-to-handle-request-timeout-7ac37aeb5f2b
