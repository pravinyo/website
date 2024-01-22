---
title: "Understanding the impact of inaccurate User Acceptance Testing Environment"
author: pravin_tripathi
date: 2024-01-21 00:00:00 +0530
readtime: true
img_path: /assets/img/understanding-the-impact-of-inaccurate-user-acceptance-testing-environment/
categories: [Blogging, Article]
tags: [backenddevelopment, design, uat]
image:
  path: header.jpg
  width: 1000   # in pixels
  height: 400   # in pixels
  alt: Generated using Bing AI
---

User Acceptance Testing (UAT) is a crucial stage in the software development lifecycle. This process provides an opportunity for end-users to validate the functionality, usability, and compatibility of a system before it is officially launched or delivered. However, the UAT phase can often be marked by unexpected complications, with systems frequently exhibiting incorrect responses or incomplete functioning. These issues can lead to significant delays, increased developmental costs, and potential damage to the client's reputation. In this article, we will delve deeper into understanding these challenges and explore effective strategies to address them.

## Identifying the Challenges
The journey to the UAT phase often begins with a system that seems to function perfectly in the development environment. The software, in this stage, appears to meet all specified requirements and performs as expected. However, when the system is introduced to the UAT environment, the smooth sailing often turns into a stormy ride. The system may begin to return incorrect responses or demonstrate incomplete or inconsistent functionality. These unexpected issues can significantly prolong the project timeline, inflate development costs, and potentially harm the vendor's reputation.

## Problems due to inaccurate setup
Let me share my recent experience with one of the stock depository services (Let's call them D).

The D’s system plays a vital role in the financial market, and one of the products that I am using is designed to facilitate the pledging of shares. This process involves an investor pledging their shares to a lender as collateral for a loan. Once the pledge is created, the D’s system is supposed to send a callback response confirming the successful pledging of shares. However, issues arise due to the faulty state management, session management, security management design of the system in UAT. In Production environment, all works fine. 

![robot](robot.jpg)
_generated using bing ai_

## Faulty state management of the transaction in UAT
Let's first start the discussion with state management of the transaction. It is of serious concern, if system is poorly handling state management. Lets me share one instance where I have faced problem due to this. In D’s system, every time a client’s customer wants to start the pledging flow, one transaction gets created which is used to identify the request in D’s internal system. Clients (Bank or NBFC) have to persist necessary details to query the state of the transaction at a later point in time to know progress as flow is performed on D’s portal and the only way to know is via callback response.

A transaction can be in one of the following states,
1. customer login to the web portal
2. customer pledged the shares (using their Demat account)
3. Depository Participants processed the shares for pledging
4. Shares got transferred to Clients (Demat account)

### Failure response is not returned in status API, but callback response returns the status in UAT
Transaction status returned in callback response and transaction status API is not consistent. If some error happens, the callback response will provide the details of failure but let's say if we call transaction status API, it will return the last successful status but not the failure details. This inconsistency forced clients to implement some state management at their end with additional logic. This is not good.

## Some cases cannot be tested in UAT for that requires a Prod account to test
There were some cases that I wanted to try out on the UAT environment which is supposed to be a replica of the prod environment, but when I heard that those cases I could not test in UAT, I asked question **Why it cannot be tested in UAT**. well, I didn’t get any proper response. `The impact of this is I have to test in Prod and if all cases are successful then great, otherwise I have to fix it.` This is just causing more delay for product release and causing inconvenience to the clients. If the system is well tested and proper testing is been employed with proper data, I don’t have to wait until Prod deployment to test the functionality. This simply explains the engineering practice by D’s tech team is bad and they have to do something. Since they have a good number of active Demat accounts, Banks and NBFCs have to work on some solutions to mitigate various problems. Also, It is not easy for newcomers to build a new depository service business as a barrier to entering this type of market is not easy, So seems like they don’t care now. But if other existing competitors come up with robust and better systems, D will be in trouble.

## Faulty session management in UAT
As per the documentation, D’s system should timeout the session if the transaction is in one of the below states,

1. Got redirected the client’s app to D’s portal for login but no action for 10 mins.
2. Post login, If the customer didn’t pledge shares within 15 min then the session will time out.

In UAT, the second point cannot be tested and their team says some development is going on so it is disabled. This is a very bad management of the environment by D’s tech team. Ideally, they should not be using UAT as it is used by the external client to test their system and use that behavior as a reference for prod release. In one of the tests, I can go beyond 35 minutes in a single session post-login. There are various complex things clients could be doing and based on session timeout clients may want to restart the transaction as they want to lend the loan to the customer. This simply proves that D’s UAT is not in good shape.

> Due to the above reasons, my team has to rely on documentation to proceed with the code changes. It is frustrating as we cannot test our system using their UAT system. Also, the UAT system is not at all close to the production environment which gives less confidence to me whether my code will work properly or not. This is a kind of nightmare for anyone who is chasing a tight deadline.
{: .prompt-warning }

## Faulty security check in UAT
This is something no one wants to ignore being in the finance industry and you are building an app that your clients going to use for financial transactions.

![fight](fight.jpg)
_generated using bing ai_

### A digital signature is asked for but not validated
Digital signature verification is a good security practice. Let me share an instance where inconsistency with this functionality in two different environments caused more confusion. According to the documentation, the server checks whether data has been tampered with for requests coming from the client. It creates a new digital signature of the data and compares it with the one sent by the client. But in UAT, this is not working, and the request works with a wrong digital signature. This is a serious problem as I don’t know whether the code changes added by me will work in production or not; And whether the digital signature generated is accepted by the server. If in production, the request fails due to an issue in digital signature it will again cause a delay to the product release.

### API contract asks for more details than needed to get transaction status.
It is always a good practice to ask only necessary information to return data to the consumer of the API. In my case, The API that I am going to discuss has more details like digital signature, requestor name, requestor ID, transaction ID, and transaction initiation time.

I found that, if the requestor name and transaction ID are provided then it could return the data. If only 2 information is requested, it is a waste to provide other information. The creation of a digital signature is computationally expensive, and if not required, it should be removed from the contract.

One of the biggest risks that I see with this API is that, it works if we provide wrong data for digital signature, requestor ID, and transaction initiation time but correct data for requestor name and transaction ID. It exposes another brute force attack which could leak information for other clients in UAT.

## Navigating the Challenges
![direction](direction.jpg)
_generated using bing ai_
Given the potential implications of these challenges, it's crucial to navigate them effectively. One of the most effective strategies is to ensure that the UAT environment closely mirrors the actual production environment. This involves aligning the system configurations, using realistic test data, and simulating actual user behavior and load.

Regular synchronization between the UAT and production environments can also play a critical role in the early detection and resolution of potential issues. By continually aligning these two environments, discrepancies can be identified and addressed before they escalate into larger problems.

Another key part of navigating these challenges involves adopting a comprehensive and thorough testing strategy. This strategy should encompass various types of testing, including unit testing, integration testing, system testing, and acceptance testing, among others. Each phase of testing should be designed to identify specific types of issues, thereby enhancing the system's overall robustness and reliability.

## Conclusion

The impact of inaccurate User Acceptance Testing (UAT) environments on software development projects, as illustrated through the experiences with a stock depository service (referred to as D), highlights significant challenges and potential consequences. The identified issues in the faulty state management, session management, and security checks during the UAT phase can lead to delays, increased development costs, and erode confidence in the system's reliability.

The challenges stem from discrepancies between the UAT and production environments, where unexpected behaviors arise in the UAT environment, jeopardizing the accuracy of testing outcomes. The issues with transaction status inconsistencies, untestable scenarios, session timeout discrepancies, and security vulnerabilities underscore the need for a more robust UAT environment.

To address these challenges, it is crucial for organizations to ensure that their UAT environment closely mirrors the production environment. Regular synchronization, realistic test data, and simulation of actual user behavior can contribute to early issue detection and resolution. Additionally, adopting a comprehensive testing strategy that includes various testing phases is essential for enhancing the system's robustness.

The experiences shared serve as a reminder of the importance of maintaining the integrity of UAT processes. Successful navigation of these challenges requires a commitment to aligning environments, implementing thorough testing practices, and fostering a collaborative approach between development teams and end-users. Ultimately, a well-structured and accurate UAT environment is imperative for ensuring the successful and timely release of high-quality software products.