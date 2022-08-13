---
title: "Tale of API Documentation"
author: pravin_tripathi
date: 2022-08-13 00:00:00 +0530
readtime: true
img_path: /assets/img/tale-of-api-documentation/
categories: ["Blogging", "Article"]
tags: [design, backenddevelopment, softwareengineering]
image:
  path: header.jpg
  width: 1000   # in pixels
  height: 400   # in pixels
  alt: Photo by Ben Kolde on Unsplash
---

**What do we mean by API in simple terms?** _An interface through which clients interact with the services._ 

**Why am I asking about API?** _There is a reason I brought up this topic. I have read a lot of API documentation internal and external(vendor API) to the projects, and the majority of them are poorly maintained. Sometimes, it is hard to understand what data to send for the working of the API._

> Why don't developers give importance to `writing API documentation`?

> Why is it still hard to understand the API even after reading the documentation provided by the team who developed that API?

> Let me share the problems I faced while working with other APIs. This problem is something I have experienced in many projects internally and with vendor-provided solutions.  
{: .prompt-info }

## Instance 1: Account Details API
This API is used to get account details based on `customer ID (Required)` and `account id (Optional)`.
- API says the customer ID field is mandatory, but if not provided still works. &#128529;
- If only an account ID was provided, which is optional, it can still return details based on the optional field. &#128529;
- Secondly, it also `returns other resources` which have `matching account IDs` but different customer IDs. 
  > It was not documented for the API.
- Even if it returns details for multiple resources, there is `no way to identify` which resource belongs to which customer ID as there is `no field ID` in returned resource details. 
  > Seriously, Which to use? How am I going to identify it? It makes no sense to me. &#128532;

## Instance 2: Suspension Details API
Legacy API is used to get the **suspension details** for `customer id`.
- API is named `GET BASE_URL/suspension-details`, but it returns account details. 
- Due to bad naming, it created more confusion during migration. It should have been named `GET "BASE_URL/account-details`.  
  > Why did this thing get missed? &#128533;

## Instance 3: Money Transfer API
An API used to `send money` from an `internal account` to an `external account` (customer account)
- API has `some fields marked as mandatory` in the documentation, but it can `accepts empty/null values`.  
  > It doesn't make sense. What is the point of mandatory fields?
- Details for **debtor** and **creditor accounts** have all the fields marked as `optional`. In reality, as per the business, that API needs `mandatory details`, but technical implementation says different things.  
  > I had to make a few calls with the client team and check the usage in other products to understand the API. 
- In the name of `adding encryption` features in the API, many fields are `renamed`, new fields were `added`, and some were `removed`.  
  > I don’t understand why this is not communicated to the other team/business, as this leads to the wrong information communicated with developers. _Later, wrong estimations are given by developers._  
  > This is a serious issue.

## Instance 4: Product Rate API
An API used to update the product rate. This rates get updated multiple times in a day.
- The `data model` name used for serializing the request payload looks like the `TOCard`, but what does `TO` mean here? No documentation was available, and When I checked the Jira story, there were also TO terms used.  
  > So, what does "TO" mean? I was clueless.
- In business terms, there was not any TO-related jargon used.
- The team that built that API is no longer available. During migration, new developers picked it up, but they had no full context.  
 > Going directly to the product owner could be a good approach, but I wanted to know why that developer thought `TOCard` name is sufficient to `understand the payload data`. 
- Once data is updated, it returns the response which has a message like, `Successfully updated TOCard details`
 > Later I realized after talking with a few old developers that "TO" in TOCard may mean` Treasury Officer`, as there is one such department. But, it is `again a guess` by that developer.  
 I do not understand what that developer achieved by saving a few characters in the model name. It is a useless name as it is not directly helping me understand the API.


## Instance 5: Event Producer
Interface does not have to be REST API. It could be a Pub-sub type interface, where the requests in the form of a message are consumed, and a response is sent in the form of a message/event.

Here, Incomplete implementation of producer pushed in production that wasn't consumed by any product. So, it did not affect any product, but What is the meaning of that code? Why was that code not reverted? It gave me the understanding that since it is in prod, It must be working. 

After I integrated my changes to consume that message, my code won't get it. After debugging the framework, I found that the header in the message had one mandatory field missing. After fixing it worked. 

Seriously, for someone new to the framework, this is a nightmare as he could not know framework level understanding in a short time. Better to implement it correctly or avoid writing it in the first place.


## Instance 6: 2 Factor Authentication API 
An API is used to send OTP pins to the customers, and another API is used to validate OTP pins.

Here validating OTP requires a sent OTP pin and function ID(used to identify message format specific to the product).

### Problem:
OTP Validation API suddenly stopped working for the product I was working on, which caused smoke failure in the lower environment. It blocked us from pushing any change to the higher environment like QA.

### Before:
The default value used for function ID in OTP generation and validation. This change was working fine. The front end pulls the function ID from DB based on product code. 

### Now:
After eight months, the frontend code had two different values for the product code (for example, FOREX and FOREX_CARD). 

The frontend code was quite complex to check which config was used while making the OTP generation and validation call, as two different configs were present with different product codes.

Later checking the DB, I found that the product code got updated from FOREX to FOREX_CARD a few days ago. In the OTP validation API, I found that if the function ID is missing in the request body, it takes the default value.

Next, talking with the integration team, I learned that the function ID has to be the same in OTP generation and validation calls for a product code which was currently a different value in the API call. 

Later after 3-4 hr debugging, I realized that the frontend was using an older name (FOREX), not the new name(FOREX_CARD) to get the function ID. By looking at the frontend code, it was difficult for me to understand those things, but with the help of frontend folks, I found the config.

Here is the question in my mind,
- Why were there two different names for product code in the frontend code?
- Why is older code not removed?

Anyway,
Later front-end code was refactored and removed code related to older names. Afterward, It started working.

It is the code and unclear understanding of the API usage due to poor documentation that caused the issue. 

The function ID has to be the same in OTP generation, and validation calls for a product code => This was not documented in tools like Jira, confluence, etc. Later talking with the integration team, I came to know. 

## Learning
I will not blame the team, but the practice that was followed by the developer who implemented that API in the product should have mentioned necessary details in the story so that others could quickly come to know.

It is better to have a clear understanding of API before making any changes to the product. Small changes could go unnoticed, and no one will come to know the actual cause of failure unless the one who implemented that change is debugging the issue, which in my case was not available.

## Conclusion
Documentation is there to help other developers transition to your API smoothly. It is also better to review your documentation with other peers if you are not clear about what to include and add. This documentation saves a lot of hours and unnecessary meetings and calls for small things. It is better to add implicit details in the documentation not directly mentioned in the code but is required to use the API correctly.

Avoid using meaningless small names or names like TOCard, which won’t help anyone understand your API. If those terms are business defined better to add a description explaining in 1-2 lines.

Double check the details regarding what fields are optional and mandatory, as miss information could cause the story to spill over.