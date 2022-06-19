---
title: "Code Smell Series: Naming smells - Part 2"
author: pravin_tripathi
date: 2022-06-19 00:00:00 +0530
readtime: true
img_path: /assets/img/code-smell-series/naming/
categories: [Blogging, CodeSmellSeries]
tags: [coding, smells]
image:
  path: header_funny.jpg
  width: 1000   # in pixels
  height: 400   # in pixels
  alt: Image by Kira Pham from Pixabay
---

> ...Ahem...Why I still see non-sense naming?
{: .prompt-warning }

I recall encountering a few naming smells in the legacy system, during migration activity. There I saw `three files with similar name in the production infrastructure code`. 

## Context 
As per new business requirement, I had to make some enhancements to the existing service to **use different authorization tokens based on environments**. Config/Code level changes were pretty straight forward.

I encountered an actual problem when I wanted to deploy my changes to a non-dev environment (like release assurance and Prod). There, I needed to add some additional changes to `Jenkins credentials`, deployment `pipeline code`, and `helm chart`. 

## The Problem
```
Prod.yaml
Prod-new.yaml
Prod-new-new.yaml
```

I saw the above three files for both `release assurance` and `production` environment. In that legacy system, no active development was happening. When I checked git logs for the latest changes, **I failed to figure out which file I needed to change**. Only one of those configurations is in use in the deployment pipeline.

## Which one is it?
![funny image][funny-image-1]

After checking the commit, I saw `all three was updated same time`. It is weird. **_Why does someone updated all three files when only one is in used?_**

**You may say, It could be to have a backup of the configuration**. As git is been used for a long time, I don’t see any reason for the backup. If you want to revert to 1 month older configurations, you can easily do it. 

Ok, **I already spent more than 20 mins**, and I was thinking `prod.yaml` should be the one, but to double confirm, _I checked with the infra folks for what file I have to update_, and there I came to know I have to update `prod-new-new.yaml`.  
![funny image][funny-image-2]

## What? Why this weird naming format?
> I don’t understand this naming convention. This naming is adding confusion instead of bringing clarity.
{: .prompt-danger }

I asked the same question, but they were saying, **Pravin that been there for a long time, and what can we do?** I don’t understand How come that change at that time got approved.

&nbsp;
> You have something to share, go ahead and add in comments 😉 » Happy Learning!!
{: .prompt-tip }


[funny-image-1]: https://media.giphy.com/media/EHtxY9W5udG8Dty7a6/giphy.gif
[funny-image-2]: https://media.giphy.com/media/7v735rSZA1Szm/giphy.gif