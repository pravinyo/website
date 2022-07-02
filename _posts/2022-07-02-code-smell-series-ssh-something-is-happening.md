---
title: "Code Smell Series: Shh…something is happening there"
author: pravin_tripathi
date: 2022-07-02 00:00:00 +0530
readtime: true
img_path: /assets/img/code-smell-series/ssh-something-is-happening/
categories: [Blogging, CodeSmellSeries]
tags: [coding, smells]
image:
  path: header.jpg
  width: 1000   # in pixels
  height: 400   # in pixels
  alt: Image by Michal Jarmoluk from Pixabay 
---

> ...Shh...let me tell you...something is happening in that code.
{: .prompt-warning }

Last month, I encountered one problem in the code. That code act as a decision-maker to decide the next step in the application, based on the decision from the human(Team Dispatcher), which is an async task and takes time. 

## The Problem
The problem was that decision-maker implementation was complex and was doing many things. Things were happening behind every line of code, Which is difficult to understand even if you read that code multiple times. 

Let’s jump to the code,

```kotlin
class DispatcherDecisionMaker(
    private val onApprove: Step,
    private val onReject: Step,
    private val onPendingSettlement: Step
) {
    fun onExecute(
        dispatcherHumanTaskResponse: DispatcherHumanTaskResponse? = null,
        fetchPaymentResponse: FetchPaymentResponse? = null
    ): Step {
        return when (fetchPaymentResponse == null) {
            true -> getDispatcherResponse(dispatcherHumanTaskResponse)
            false -> when (fetchPaymentResponse.isPaymentSettled) {
                false -> onPendingSettlement
                else -> getDispatcherResponse(dispatcherHumanTaskResponse)
            }
        }
    }

    private fun getDispatcherResponse(dispatcherHumanTaskResponse: DispatcherHumanTaskResponse?) =
        when (dispatcherHumanTaskResponse == null) {
            true -> onApprove
            false -> when (dispatcherHumanTaskResponse.decision) {
                CaseDecision.APPROVED -> onApprove
                else -> onReject
            }
        }
}
```

> Even today, After reading the above code and knowing the flow, I still need to check steps and other implementations to understand the code. I encourage you to once read the above code and then think about what you understood? 
{: .prompt-info }

![bored funny image][funny-image-3]

## Understanding
If you are unable to understand the above code, you are not alone. Let me brief you on what checks the above code does?  
* Is `payment settlement` toggle/flow enabled?
* Was there any response from the fetch `payment status` API call?
* If there was any response, check for `payment settlement` status and decide whether to wait for settlement to complete or move on to `dispatcher(Team) decision`?
* Is `dispatcher human task` toggle/flow enabled?
* If it is not enabled, `default to as approved`. Otherwise, check the **decision made by the Team Dispatcher and decide on the next step**.  

![push all thing funny image][funny-image-1]{: w="400" h="600"}

I was surprised that too many things were happening and have been taken care of by the above piece of code. But reading the code again now, I still feel confused as it is implicit knowledge, and the **other developer might not have that proper context, so he/she again needs to read the last step from the code itself**.

From the code, we see the **class is handling two responsibilities** `Payment decision` and `Dispatcher decision`. Also, there is `naming smell` problem in **getDispatcherResponse()**. 
> getDispatcherResponse() method doesn't return the dispatcher response but returns the next step to execute based on the human decision from Dispatcher Team.
{: .prompt-danger }

`fetchPaymentResponse == null`, this code check whether payment settlement flow/toggle is enabled or not. Seriously, **I can't interpret this by reading this class alone. That developer will have a hard time understanding his code after a few years.**

What is the use of `else` in line number 15?, Why not `true`? it **makes more sense than writing `else` there**.  
```diff
- else -> getDispatcherResponse(dispatcherHumanTaskResponse)
+ true -> getDispatcherResponse(dispatcherHumanTaskResponse)
```
{: .nolineno}

`dispatcherHumanTaskResponse == null`,  this code checks whether Dispatcher Human Task flow/toggle is enabled or not. Again, this is hard to understand from the code.  


> I am scared of the day if that change got pushed into production, and after a few months, some strange bug appears. I can't imagine that scenario for the developer who will be debugging it as everything is happening as an event.  
{: .prompt-warning }

![scary funny image][funny-image-2]

## Refactoring
Now, Let's see refactored version of the above implementation.

```kotlin
class DispatcherDecisionMaker(
    private val onApprove: Step,
    private val onReject: Step,
    private val onPendingSettlement: Step
) {
    fun onExecute(
        dispatcherHumanTaskResponse: DispatcherHumanTaskResponse? = null,
        fetchPaymentResponse: FetchPaymentResponse? = null
    ): Step {
        return when (isPaymentSettlementEnabled(fetchPaymentResponse)) {
            false -> nextStep(dispatcherHumanTaskResponse)
            true -> when (isPaymentSettled(fetchPaymentResponse)) {
                false -> onPendingSettlement
                true -> nextStep(dispatcherHumanTaskResponse)
            }
        }
    }

    private fun isPaymentSettlementEnabled(fetchPaymentResponse: FetchPaymentResponse?): Boolean {
        return fetchPaymentResponse != null
    }

    private fun isPaymentSettled(fetchPaymentResponse: FetchPaymentResponse?): Boolean {
        return fetchPaymentResponse!!.isPaymentSettled
    }

    private fun nextStep(dispatcherHumanTaskResponse: DispatcherHumanTaskResponse?) =
        when (isDispatcherFlowDisabled(dispatcherHumanTaskResponse)) {
            true -> onApprove
            false -> when (dispatcherDecision(dispatcherHumanTaskResponse)) {
                CaseDecision.APPROVED -> onApprove
                else -> onReject
            }
        }

    private fun isDispatcherFlowDisabled(dispatcherHumanTaskResponse: DispatcherHumanTaskResponse?): Boolean {
        return dispatcherHumanTaskResponse == null
    }

    private fun dispatcherDecision(dispatcherHumanTaskResponse: DispatcherHumanTaskResponse?): CaseDecision {
        return dispatcherHumanTaskResponse!!.decision
    }
}
```
## Further Refactoring
Above code is still doing multiple things. It can be further simplified into 2 separate class,  
Implementation of `Dispatcher Decision Maker`:
```kotlin
class DispatcherDecisionMaker (
    private val onApprove: Step,
    private val onReject: Step
) {
    fun onExecute(dispatcherHumanTaskResponse: DispatcherHumanTaskResponse? = null): Step {
        return when (isDispatcherFlowDisabled (dispatcherHumanTaskResponse)) {
            true -> onApprove
            false -> when (dispatcherDecision (dispatcherHumanTaskResponse)) {
                CaseDecision.APPROVED -> onApprove
                else -> onReject
            }
        }
    }

    private fun isDispatcherFlowDisabled (dispatcherHumanTaskResponse: DispatcherHumanTaskResponse?): Boolean {
        return dispatcherHumanTaskResponse == null
    }

    private fun dispatcherDecision (dispatcherHumanTaskResponse: DispatcherHumanTaskResponse?): CaseDecision {
        return dispatcherHumanTaskResponse!!.decision
    }
}
```

Implementation of `Payment Status Decision Maker`:
```kotlin
class PaymentStatusDecisionMaker(
    private val onPendingSettlementFlow: Step,
    private val onSuccess: Step,
    private val onSkip: Step
){
    fun onExecute(
        fetchPaymentResponse: FetchPaymentResponse? = null
    ): Step {
        return when (isPaymentSettlementEnabled (fetchPaymentResponse)) {
            true -> when (isPaymentSettled(fetchPaymentResponse)) {
                true -> onSuccess
                false -> onPendingSettlementFlow
            }
            false -> onSkip
        }
    }

    private fun isPaymentSettlementEnabled (fetchPaymentResponse: FetchPaymentResponse?): Boolean {
        return fetchPaymentResponse != null
    }

    private fun isPaymentSettled(fetchPaymentResponse: FetchPaymentResponse?): Boolean {
        return fetchPaymentResponse!!.isPaymentSettled
    }
}
```

It looks much better as each class is doing one thing.
> Even though it required to create another class but at the cost of clean responsibility and better readability, Which has more advantages than disadvantages. Also, it increases reusability.
{: .prompt-tip }

## Conclusion:
Initially, we saw that code was doing more things that can’t be easily interpreted by just reading it. To get the full context of the whole code, I needed to read the previous flow, and then I had to figure out what was happening, which was a clear signal of code smell. Be careful next time.

&nbsp;
> You have something to share, go ahead and add in comments 😉 » Happy Learning!!
{: .prompt-tip }

[funny-image-1]: https://media.giphy.com/media/M8vBiv9mgpHDGpqL9y/giphy-downsized-large.gif
[funny-image-2]: https://media.giphy.com/media/3iBbm6ilVZkC6RmjhL/giphy.gif
[funny-image-3]: https://media.giphy.com/media/kkxQdM0zvG2xCmmNh5/giphy-downsized-large.gif