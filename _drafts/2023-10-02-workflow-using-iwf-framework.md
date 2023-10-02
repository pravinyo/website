---
title: "REST API Based Workflow Design Using iWF Framework"
author: pravin_tripathi
date: 2023-10-02 00:00:00 +0530
readtime: true
img_path: /assets/img/iwf-workflow/
categories: [Blogging, Article]
tags: [backenddevelopment, design, java, softwareengineering]
image:
  path: header.jpg
  width: 1000   # in pixels
  height: 400   # in pixels
  alt: Generated using Bing AI
---

# Using the iWF DSL framework to write workflow on the top of the Temporal platform

In this article, We are going to discuss Workflow and design a simple use case using the iWF framework (with Temporal Server).

## Part 1: Basics Concepts
Before directly jumping to code, Let's see some concepts about workflow and Temporal Server.

### Runtime platform
Provide the ecosystem to run your applications and take care of the `durability, availability, and scalability` of the application. Both Cadence(from Uber) and Temporal share the same behavior as Temporal is forked from Cadence. **Worker Processes are hosted by you and execute your code**. The communication within the Cluster uses **gRPC**. Cadence/Temporal service is responsible for keeping workflow state and associated durable timers. It maintains internal queues (called task lists) which are used to dispatch tasks to external workers. Workflow execution is resumable, recoverable, and reactive.

- [Cadence Doc](https://cadenceworkflow.io/docs/get-started/)
- [Temporal Doc](https://docs.temporal.io/temporal)
- [iWF Project](https://github.com/indeedeng/iwf/wiki/Basic-concepts-overview)

![Temporal service](temporal-service.png)
_Temporal System Overview for workflow execution_

### What is Workflow?
The term Workflow frequently denotes either a Workflow Type, a Workflow Definition, or a Workflow Execution.

- Workflows are sequences of tasks/steps that are executed in a specific order.
- It is based on the principle of separation of concerns.
- It focuses on the design and implementation of business processes as workflows.
- **Workflow Definition**: A Workflow Definition is the code that defines the constraints of a Workflow Execution. A Workflow Definition is often also referred to as a Workflow Function.
- **Deterministic constraints**: A critical aspect of developing Workflow Definitions is ensuring they exhibit certain deterministic traits – that is, making sure that the same Commands are emitted in the same sequence, whenever a corresponding Workflow Function Execution (instance of the Function Definition) is re-executed.
- **Handling unreliable Worker Processes**: Workflow Function Executions are completely oblivious to the Worker Process in terms of failures or downtime.
- _Event Loop:_  
  ![img_1.png](workflow-execution.png)
- _Workflow execution states:_
  ![img.png](workflow-execution-state.png)

### What is a Workflow Engine?
- A workflow engine facilitates the flow of information, tasks, and events.
- The workflow engine is responsible for managing the execution of workflows.
- Workflow engines may also be referred to as Workflow Orchestration Engines
- The other components of the system are responsible for performing the specific tasks that make up the workflows


### What is the activities or workflow state?
- An Activity is a normal function or method that executes a single, well-defined action (either short or long-running), such as calling another service, transcoding a media file, or sending an email message.
- Workflow code orchestrates the execution of Activities, persisting the results. If an Activity Function Execution fails, any future execution starts from the initial state
- Activity Functions are executed by Worker Processes
- Workflow State is used in the domain of the iWF framework which is the same as Activities in Cadence or Temporal.


### Event handling
Workflows can be signaled about an external event. A signal is always point-to-point destined to a specific workflow instance. Signals are always processed in the order in which they are received.
- Human Tasks
- Process Execution Alteration
- Synchronization  
Example: there is a requirement that all messages for a single user are processed sequentially but the underlying messaging infrastructure can deliver them in parallel. The Cadence solution would be to have a workflow per user and signal it when an event is received. Then the workflow would buffer all signals in an internal data structure and then call an activity for every signal received.

### Visibility
- View, Filter, and Search for Workflow Executions
  - https://docs.temporal.io/visibility#list-filter-examples
  - https://docs.temporal.io/visibility#search-attribute
- Query Workflow state

## Part 2: Temporal Server Design
Both Cadence and Temporal provide a platform to execute our workflow function which is nothing but business logic.

![img.png](cadence-service.png)
### What are the components of the Cadence/Temporal server?
The server consists of four independently scalable services:
- **Frontend gateway:** for rate limiting, routing, and authorizing.
- **History service:**maintains data (workflow mutable state, event and history storage, task queues, and timers).
- **Matching service:** hosts Task Queues for dispatching.
- **Worker Service:** Worker Service: for internal background Workflows (replication queue, system Workflows).
- [Read more...](https://docs.temporal.io/clusters)

## Part 3: iWF Framework Design (Temporal as Backend)
iWF is the framework that is developed to simply run the workflow and harness the full potential of the Cadence/Temporal Server.

### High-Level Design
An iWF application is composed of several iWF workflow workers. **These workers host REST APIs as “worker APIs” for the server to call**. This callback pattern is similar to AWS Step Functions invoking Lambdas if you are familiar with it.  

An application also performs actions on workflow executions, such as starting, stopping, signaling, and retrieving results by calling iWF service APIs “service APIs”.  

The service APIs are provided by the “API service” in the iWF server. Internally, this API service communicates with the Cadence/Temporal service as its backend.  
![HLD for iWF](iwf-architecture.png)


### Low-Level Design
Users define their workflow code with a new SDK “iWF SDK” and the code is running in workers that talk to the iWF interpreter engine.  

The user workflow code defines a list of WorkflowState and kicks off a workflow execution. At any workflow state, the interpreter will call back the user workflow code to invoke some APIs (waitUntil or execute). Calling the waitUntil API will return some command requests. When the command requests are finished, the interpreter will then call the user workflow code to invoke the “execute” API to return a decision.  

The decision will decide how to complete or transition to other workflow states. At any API, workflow code can mutate the data/search attributes or publish to internal channels.  
![LLD for iWF](iwf-workflow-execution.png)


### RPC: Interact with workflow via API
Using RPC annotation is one of the ways to interact with the workflow from external sources like REST API, and Kafka event. It can access persistence, internal channels, and state execution. 
![communication](communication.png)  
[RPC vs Signal](https://github.com/indeedeng/iwf/wiki/RPC#signal-channel-vs-rpc)  
Both RPC and Signal are the two ways to communicate from an external system with the workflow execution.
  - RPC is a synchronous API call - [Definition](https://github.com/indeedeng/iwf-java-sdk/blob/main/src/main/java/io/iworkflow/core/RpcDefinitions.java)
  - The signal channel is an Asynchronous API.

Some recommend, as a best practice, to use RPC with an Internal channel to asynchronously call the workflow. It is basically to replace the Signal API.  
`RPC + Internal Channel => Signal Channel`

> Internal-Channel and Signal Channel are both message queues
{: .prompt-info }
  

### iWF Approach to Determinism and Versioning
There are some problems with the history replay for the workflow which causes non-determinism issues due to events like workflow state deletion or business logic changes, etc.

- iWF framework recommends using the flag to control the code execution as versioning is removed.
- Since there is no versioning, the non-determinism issue will not happen.
- Read more: [IWF doc](https://github.com/indeedeng/iwf/wiki/Compare-with-Cadence-Temporal#determinism-and-versioning)


### Example of Atomicity using RPC for sending message, state transition, and saving data in DB.
```java
public class UserSignupWorkflow implements ObjectWorkflow {
  ...

    // Atomically read/write/send message in RPC
    @RPC
    public String verify(Context context, Persistence persistence, Communication communication) {
        String status = persistence.getDataAttribute(DA_Status, String.class);
        if (status.equals("verified")) {
            return "already verified";
        }
        persistence.setDataAttribute(DA_Status, "verified");
        communication.publishInternalChannel(VERIFY_CHANNEL, null);
        return "done";
    }
    ...
}
```

## Part 4: Simple workflow example using iWF
Below is the workflow diagram of the KYC application based on Aadhaar.
![diagram](kyc-workflow.png)

### Step 1: Write Workflow definition
```java
public class AadhaarKycWorkflow implements ObjectWorkflow {
    private final List<StateDef> stateDefs;

    public AadhaarKycWorkflow(Client client) {
        this.stateDefs = List.of(
                StateDef.startingState(new GenerateAadhaarOtpStep()),
                StateDef.nonStartingState(new ValidateAadhaarOtpStep()),
                StateDef.nonStartingState(new SaveAadhaarDetailsStep(client))
        );
    }

    @Override
    public List<StateDef> getWorkflowStates() {
        return stateDefs;
    }

    @Override
    public List<PersistenceFieldDef> getPersistenceSchema() {
        return List.of(
                SearchAttributeDef.create(SearchAttributeValueType.TEXT, "customer_id"),
                SearchAttributeDef.create(SearchAttributeValueType.TEXT, "aadhaar_id"),
                SearchAttributeDef.create(SearchAttributeValueType.TEXT, "parentWorkflowId")
        );
    }

    @Override
    public List<CommunicationMethodDef> getCommunicationSchema() {
        return List.of(
                SignalChannelDef.create(String.class, "AadhaarOtpSignal"),
                SignalChannelDef.create(String.class, SC_SYSTEM_KYC_COMPLETED)
        );
    }
}
```

`StateDef.startingState`: Starting step/task/activity which workflow will execute.
`StateDef.nonStartingState`: It will be executed based on the State's decision.
`getPersistenceSchema()`: return types of data that will be accessed by the workflow. This data will be persisted as long as workflow history is preserved.
`getCommunicationSchema()`: different types of communication that workflow will require to complete the tasks.

### Step 2: Write Workflow State
It is also called the actual business rules that you want workflow to execute.
```java
public class ValidateAadhaarOtpStep implements WorkflowState<String> {
    @Override
    public Class<String> getInputType() {
        return String.class;
    }

    @Override
    public CommandRequest waitUntil(Context context, String input, Persistence persistence, Communication communication) {
        return CommandRequest.forAllCommandCompleted(
                SignalCommand.create("AadhaarOtpSignal")
        );
    }

    @Override
    public StateDecision execute(Context context, String aadhaarReferenceId, CommandResults commandResults, Persistence persistence, Communication communication) {
        var otp = (String) commandResults.getSignalValueByIndex(0);
        if (validateOtp(aadhaarReferenceId, otp)) {
            var details = fetchAadhaarDetails(aadhaarReferenceId, otp);
            return StateDecision.singleNextState(SaveAadhaarDetailsStep.class, details);

        }

        return StateDecision.singleNextState(ValidateAadhaarOtpStep.class, aadhaarReferenceId);
    }

    private Boolean validateOtp(String aadhaarReferenceId, String otp) {
        log.info("call aadhaar validate OTP API and fetch details for referenceId:{} and OTP:{}", aadhaarReferenceId, otp);
        return Objects.equals(otp, "1234");
    }
}
```
`waitUntil()` and `execute()`: are the two sub-steps that the workflow state executed in sequence to finish the task.  
`waitUntil()`: It returns Signals, Timer, or Internal event that the task is waiting to happen. Once that event is completed, execute() will be invoked.  
`StateDecision`: It returns the next state that workflow should be expected to execute. This will be executed only when the Temporal/Cadence Server schedules the task on the internal worker queue.

### Step 3: REST API endpoint to provide input to workflow

```java
    @PostMapping("/kyc/aadhaar/otp")
    ResponseEntity<Response> validateAadhaarOtp(
            @RequestParam String otp,
            @RequestParam String customerId) {
        var workflowId = getWorkflowIdForAadhaar(customerId);
        var response = client.describeWorkflow(workflowId);

        if (response.getWorkflowStatus().equals(WorkflowStatus.RUNNING)) {
            client.signalWorkflow(AadhaarKycWorkflow.class,
                    workflowId, "AadhaarOtpSignal", otp);
            return ResponseEntity.ok(new Response("success", ""));
        }
        return ResponseEntity.internalServerError().body(new Response("Workflow not running", ""));
    }

    private String getWorkflowIdForAadhaar(String customerId) {
        return "WF-LAMF-KYC-"+customerId;
    }
```

## Part 5: Different Use Cases
Below are the examples to understand the usage of different APIs of the iWF framework.
- [Microservice Orchestration](https://github.com/indeedeng/iwf/wiki/Use-case-study-%E2%80%90%E2%80%90-Microservice-Orchestration)
- [user signup workflow](https://github.com/indeedeng/iwf/wiki/Use-case-study-%E2%80%90%E2%80%90-user-signup-workflow)
- [KYC Workflow](https://github.com/pravinyo/workflow/tree/main/src/main/java/dev/pravin/workflow/kyc)
- [Product order workflow](https://github.com/pravinyo/workflow/tree/main/src/main/java/dev/pravin/workflow/shop)
- [Loan application workflow](https://github.com/pravinyo/workflow/tree/main/src/main/java/dev/pravin/workflow/lamf)

Project Link:
- [Github project](https://github.com/pravinyo/workflow)
- [iWF Project](https://github.com/indeedeng/iwf)

## Conclusion
iWF framework has really simplified writing applications using workflow-oriented architecture. Writing applications with the direct APIs provided by Cadence/Temporal has a steep learning curve. Due to this, beginners make some common mistakes, and writing a workflow that uses the full potential of the system is challenging for newcomers.

iWF Project is basically a wrapper on the top of Cadence and Temporal System which helps lower the learning curve and also helps writing workflow that uses the full potential of the system which is really great.