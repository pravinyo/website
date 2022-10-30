---
title: Do not trust others' code
author: pravin_tripathi
date: 2022-10-31 00:00:00 +0800
categories: [Blogging, Article]
tags: [design, backenddevelopment]
published: false
img_path: /assets/img/do-not-trust-others-code/
readtime: true
image:
  path: header.jpg
  width: 1000   # in pixels
  height: 400   # in pixels
  alt: Image by Dimitris Vetsikas from Pixabay
---

In this article, I shared one instance of poor API implementation with numerous bugs and my experience with it.

## Background Context
As per the current business process, If someone wants a loan against security, they have to physically visit one of the branches and wait in a long queue to start the process for the loan. 

The person who handles that request manually checks documents and later creates customers in the system once a loan is approved. Now, business wants to go digital and automate this manual process. Once the loan is approved digitally, the customer gets created in the same backend system (Legacy). This legacy system is core for most of the process and is difficult to change in a few years. So now you understand what problem we are going to discuss here.
 
Client creation API is used to add a new customer once the loan against security is approved. This API already exists. In the orchestrated loan application journey, I have to consume that API once the loan is approved. 
 
## The Problem
I am not able to create new clients in the legacy system. I read the documentation and reviewed the sample request numerous times but failed to create a new client via API.

 
Let’s begin,
 
## Part 1: Status code misuse and misleading error message
My initial feeling by looking at the API documentation was scary as it has more than 130 fields for the request in an excel sheet. Whenever I send a request for client creation, I get the below error,

Status code: `500 INTERNAL SERVER ERROR`
```json 
{
           "Message": "An error has occurred."
}
```

After looking at the above response, I thought it did authentication successfully, and when it hits the service, something has gone wrong with the request, and because of that it is giving the above error.
 
But, API requests have more than 130 fields in sample requests, and It is unclear from the response what is the actual problem. Even after trying different approaches , I failed to create a client, so I contacted a person from the integration team who successfully added a new client to the system, but that person used UI to create the client. Gauche, this is a pain. It's like hitting the wall. 
 
One and half days have been wasted mailing, waiting, calling, reading documentation, and trying different approaches. I still have no clue why I am getting this error. 
 
Based on a hunch, I thought to check the request header and found the issue. In the request header, username and password field were added in the wrong way. After resolving, I was able to get a different error but yeah some progress. 
 
> Instead of 401, they used 500 (INTERNAL SERVER ERROR ) for authentication failure. Is it the ignorance or laziness of the developer?   
> This issue took me nearly two days. And, it is not the end.
{: .prompt-info }   

![angry cat](angry-kitten-angry-cat.gif){: w="300" h="300"}

## Part 2: API is badly implemented and poorly documented
Now, I can see different errors and resolve most of the issues by referring to the documentation and the database(I got access via UI to check).
 
After resolving most of the validation errors, I got the below error.
```json
{
           …
           "status": {
            "Status": "Fail",
            …
            "Remarks": "Exception: Column ‘customerid’ does not belong to table Table."
           }
}
```

I may be sending customerId as an extra field, so I checked the payload again, but to my surprise, there is no field like customerId. As per the documentation, I do not see any problem with the request, so what is wrong here? 😫
 
Hold on, why is the error saying the customer id column in the table is not present? Something strange is about the implementation of the API. It is revealing something which should not be. I wonder what database query is used to insert new client details.
 
Since I can view the database, based on some trial and error with different combinations of values, I realized that the branch code (varchar) field with integer value works, but for string, it fails with the above error. But why is it even accepting that value if it should be an integer? why is it not part of the validation?
 
It is not the only problem with this field. When I used values from the branch code column(Integer) already in the database, as per my understanding, it should work, but it is failing with the above error again. I do not understand this behavior of the API implementation.
 
After some trial and error, I used the value from branchID instead of the branch code column in the database. It worked, but to my surprise, the documentation says branch code, but it is working with branch ID. `I lost my trust in the API implementation.`
 
![what](https://media.giphy.com/media/pPhyAv5t9V8djyRFJH/giphy.gif)
 
## Part 3: I somehow managed to make the API work, but the default value, as per the documentation, is breaking the API call.
It cost me approx. 2.5 days for making the first successful API call. It gave me mixed feelings of enjoyment and sadness. Next, I thought of playing with this API with some different values, which I will send to this API once I start the story implementation. I had some default values to use for my use case and thought to send that to the API and check whether I can successfully create a new client or not.   
 
The response I got looks like below,
```json
{
           …
           "status": {
            "Status": "Fail",
            …
            "Remarks": "Exception: Error converting data type varchar to numeric."
           }
}
```

From the above error, I thought maybe I edited the wrong field, but no, all field values look fine. So what could have gone wrong?   
The sad part about this error message is I do not know which among the multiple error fields (10 fields I edited) has the issue. I had to undo the changes and try to send a new call after each field change.  

> What a great API design and developer experience!, I was laughing with my pair. 
{: .prompt-info }
 
Later I found the problem I was giving groupName as NA (before groupName was Diamond),  ok it is the name, so I should be receiving an invalid name or something like that, but why is the API internally trying to convert that to the numeric value for NA? 
 
Seriously, I have no idea what to say. Internally I was crying and thinking about running away.
 
groupName field in the request now reverted to the previous value. Let’s see if I can send the request. Yes, I can create a new client in the system again. This time I felt true happiness in my last four days.
 
 
 ![imposter](https://media.giphy.com/media/j4fbBhYgu8mNEHkQ4w/giphy.gif){: w="300" h="300"}
 
 
## Part 4: frugally removed unnecessary/optional fields from the request payload but now a weird validation error. 
 
Since API requests have numerous optional fields, I tried to remove which are unnecessary, as per my use case. I removed most of the fields except for one field(pool account). If I remove it and send the request, here is the response,
```json 
{
           …
           "status": {
            "Status": "Fail",
            …
            "Remarks": " TaxStatus is required"
           }
}
```

Maybe I accidentally deleted the tax status field, so I checked the payload, but it is present. Why am I getting the above error? So I tried to add back the pool account and sent the same request again, and it worked.

> What is happening here? Is this another problem in the API? Big Yes.
{: .prompt-info }
 
The documentation says It is an optional field and can be blank. So I tried to remove it like other optional fields. I understand that sometimes we developers miss updating the documentation, which is ok, but why is it giving me the wrong error message? It should have said pool account required and not misleading by saying tax status.
 
 ![cleanup](https://media.giphy.com/media/oCGjUWkj8vz9u/giphy.gif)
 
## Part 5: Finally, I can send the request as per my use case, but my happiness did not last for more than 10 mins.
 
I got the feedback to update the data via UI after creating a new client in the system as branch folks will be using that same system to update the data. It is fair, and I thought it would be a piece of cake as I only have to update one field in the UI and try to save the data as branch folks do.
 
As I clicked the save button, I got numerous validation error messages.
> Yikes, what is going on here?...
> I was hoping the API would save the data in the system with all the proper validation. Is it too much to expect? 
{: .prompt-info }

Jokes apart, I manually had to resolve all those validation errors in the UI by adding the missing values and later adding the same in the API.
 
During fixing those validation issues, I encountered one mandatory field called department, which is required but was missing. So I checked API documentation for where to pass in the request, but I did not find anything about it. So I checked sample requests but no luck.
 
> Only one question I had at that time, Why did the system even allow the saving of the new client data? If validation was missing, shouldn't the API be responsible for validation?
{: .prompt-info }

![short term happiness](https://media.giphy.com/media/xT77XP5M4Iax5DVx96/giphy.gif)
 

## What could be improved?
- `Use proper documentation tool`: 
    The documentation which I got is nothing but excel sheets and some sample request payload in the text file. Seriously, we have a lot of good tools like swagger UI, Postman, Redoc, etc.; but still, why are we ignoring them and using word files, excel files, confluence, etc.?   

    I think this decision is made by the tech team, not by business, so I feel this is the ignorance of one who is leading the team. 

- `Use proper testing`: 
    Everyone knows about unit testing, integration testing, and end-to-end testing, but sometimes we do not write a valid test. We get the feeling of 80%+ test coverage but does that even matter if your code cannot do one thing properly? In my case, if testing was fairly done, I might not have spent three days sending one successful request to your API.   

- `Validate content`: 
    If you are making any code changes, ensure to validate the documentation content. In my case, it was an excel sheet with 130+ fields for request.    

- `Give meaningful error response`: 
    It is better to focus on the error handling part of the API. Sadly, This is something I have observed developers give less importance to. If validation fails, return a proper reason for failure instead of one generic error. I do not have access to your code, and you should not expect me to find time to read your code before accessing your API.   

- `Do perform other business operations on data`: 
    In my case, client creation API by name suggests I should create and save in a database. But as a developer, I also need to ensure that other business operations on the stored data are working. For here, after saving, when I tried to update the data via UI, which branch folk will do, I got many validation errors that got missed in the API implementation. When bypassing the manual process, try to validate other business flows on data.
 

## Conclusion
 Client creation API is core in journey automation. From five short stories about the same API, you could have realized the pain of using a poorly implemented API. I have experienced a lot of pain with poorly implemented APIs and poor documentation, but this experience is different from others and unique.