---
title: "Part 5: Database Engineering Fundamentals: How Shopify’s engineering improved database writes?"
author: pravin_tripathi
date: 2025-03-04 04:00:00 +0530
readtime: true
img_path: /assets/img/database-engineering-fundamental-part-5/
categories: [Blogging, Article]
mermaid: true
permalink: /database-engineering-fundamental-part-5/how-shopify-engineering-improved-database-writes/
parent: /database-engineering-fundamental-part-5/
tags: [softwareengineering, backenddevelopment]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---
# How Shopify’s engineering improved database writes by 50% with ULID

[https://www.youtube.com/watch?v=f53-Iw_5ucA](https://www.youtube.com/watch?v=f53-Iw_5ucA)

# Use Idempotency Keys

Distributed systems use unreliable networks, even if the networks look reliable most of the time. At Shopify’s scale, a **once in a million** chance of something unreliable occurring during payment processing means it’s happening many times a day. If this is a payments API call that timed out, we want to retry the request, but do so safely. Double charging a customer's card isn’t just annoying for the card holder, it also opens up the merchant for a potential chargeback if they don’t notice the double charge and refund it. A double refund isn’t good for the merchant's business either.

In short, we want a payment or refund to happen exactly once despite the occasional hiccups that could lead to sending an API request more than once. Our centralized payment service can track attempts, which consists of at least one or more (retried) identical API requests, by sending an idempotency key that’s unique for each one. The idempotency key looks up the steps the attempt completed (such as creating a local database record of the transaction) and makes sure we send only a single request to our financial partners. If any of these steps fail and a retried request with the same idempotency key is received, recovery steps are run to recreate the same state before continuing. [**Building Resilient GraphQL APIs Using Idempotency**](https://shopify.engineering/building-resilient-graphql-apis-using-idempotency) describes how our idempotency mechanism works in more detail.

An idempotency key needs to be unique for the time we want the request to be retryable, typically 24 hours or less. We prefer using an Universally Unique Lexicographically Sortable Identifier ([**ULID**](https://github.com/ulid/spec)) for these idempotency keys instead of a random version 4 UUID. ULIDs contain a 48-bit timestamp followed by 80 bits of random data. The timestamp allows ULIDs to be sorted, which works much better with the b-tree data structure databases use for indexing. In one high-throughput system at Shopify we’ve seen a 50 percent decrease in INSERT statement duration by switching from UUIDv4 to ULID for idempotency keys.

https://shopify.engineering/building-resilient-payment-systems

[Back to Parent Page]({{ page.parent }})