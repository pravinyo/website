---
title: "Code Smell Series: Naming smells - Part 1"
author: pravin_tripathi
date: 2022-06-15 00:00:00 +0530
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

> Are you sure your code is free from naming smell?
{: .prompt-warning }

You may think about ignoring it and adding to backlog due to week smell, but if that tech debt is not fixed and delayed, **it will create more headache then peace of mind while working**. It is the responsibility of developers to make sure that the code they are writing should **_not include naming Jargon_**, whether it is the application name, API endpoint, module, class, method, variables, etc.

> Until now, I have encountered this problem many times, but majority of them were small, had less impact and were fixed within a week. I recall 2 different instances which troubled me more.
{: .prompt-info }

### Recently 
I encountered naming smell, while performing application migration from one platform (in-house old service platform) to another platform (in-house new advanced service platform). There was one API that was exposed to the back-office team for pushing the latest data multiple times a day. 

> **_The API accepts a list of currency exchange details that is further processed (aggregation, validation, transformation, and more) and then stored in the DB to be used by other applications._**

### Problem 1:

Here, **naming of a few fields in currency exchange was not clear**, something like, `fxdCrncyCode` and `varCrncyCode`. 

Someone who is reading this name for the first time will not come to know what it could mean. **_Is it some business term?_** 

> **_We could guess `Crncy` could roughly mean currency given we are accepting currency exchange detail. But, what could fxd and var mean here?_**

I tried asking few developers who joined project before me, but they don’t know as that service was `last time updated a year ago`, and _folks who developed is no longer working in the project_. So next, I thought to check the **_API contract testing code_** where I  was expecting to find the test for that field and get actual meaning behind that, **but sadly, I didn’t get anything there**. 

At last, I had to read the complete API implementation to understand what was happening?, Where data was saved?, and there I saw it. 

> Fxd mean fixed and var mean Variable. 
{: .prompt-info }

#### Now you will say, Pravin

* **You could have guessed it does roughly means same variable name.**  
  But hold on,  
    - Why do I have to guess the name?  
    - Why not written clearly in the first place?  
    - What did they achieved by saving **two char from fixed** and **five char from Variable**?  

  One thing is for sure _I had to spend two days_ just `searching`, `asking`, `reading the code`, and also confirm the same in the downstream application.  

  The sad part is, the developer who implemented is no longer in the project, and **no developer on the team had the full context about it**.

* **You could have checked the API documentation and come to know everything about it.**  
  That’s true. But, I was not able to find the documentation page in confluence.

* **You could have checked details from the source that pushes the data to the API.**   
  Yes, I could have, but that might have cost me another 2-3 days as the team who uses the API is not actively aware much about the fields.  
  API integrated with them three years ago, so not everyone in that team had full context. Also, the team was busy with many tasks at that time. Given the tight deadline, that could be the last option for me.

#### Now What?
If that developer had written the proper name, which was a simple task, It might have saved my precious time.

**It is easy to criticise others code**. `What would you have done different in that developers place?` Lets discuss the possible solution to above problem.

> Some more context: After some time, I realised that developer used the same name coming from the source system in Request Payload.
{: .prompt-info }

**_You may say, from the source, we are getting data in that format, so what could be done?_**
> Are you sure there is no option other than using **fxdCrncyCode** and **varCrncyCode** that is unclear?
{: .prompt-warning }

**I think that is just an excuse**. Below are the few approaches that, You could follow to avoid above problem.
#### Approach 1:
Create DTO object which accept the source data and translate to domain specific name like,

```kotlin
data class ExchangeRateDTO (
  val fxdCrncyCode: String,
  val varCrncyCode: String 
  ) {
  fun toDomain(): ExchangeRate {
    return ExchangeRate(
      fixedCurrencyCode = this.fxdCrncyCode,
      variableCurrencyCode = this.varCrncyCode
    )
  }
}
```

> But this creates an extra class to manage. 
{: .prompt-tip }

#### Approach 2:
Much simpler way, if you are using Jackson like library for deserializing the data back to object,
Use `@JsonProperty` in the data class field name to map the name to different name.

```kotlin
data class ExchangeRate( 
  @JsonProperty("fxdCrncyCode") val fixedCurrencyCode: String,
  @JsonProperty("varCrncyCode") val VariableCurrencyCode: String
)
```

> In the API contract, Jackson will see that field name and set that against the proper name.
{: .prompt-tip }

#### Approach 3:
**Ask the source to send the correct field name**. That could have been a good approach if done initially. But in my case, that contract was already in use, and the team sending the data was lazy to pick this tech debt. 

### Problem 2:
This API had the wrong endpoint. It was like `../card-rate`, but in reality, it was not accepting card rate but currency exchange rate. `So, Why it was not fixed later?`

Due to these everywhere in the code base, wrong names was used, and  people who read the code thought, in a business context, it could mean card rate.  

But, the business people are not referring that name. so **_it is another naming smell_ in the API enpoint**.

> Naming smell - Part 2, is on second instance which troubled me.
[Link to a page]({{ site.url }}{{ site.baseurl }}{% post_url 2022-06-19-code-smell-series-file-name %})
{: .prompt-info }

> You have something to share, go ahead and add in comments 😉 » Happy Learning!!
{: .prompt-tip }