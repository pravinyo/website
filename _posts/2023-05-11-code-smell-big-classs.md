---
title: "Code Smell Series: Big Class"
author: pravin_tripathi
date: 2023-05-11 00:00:00 +0530
readtime: true
img_path: /assets/img/code-smell-series/big-class/
categories: [Blogging, CodeSmellSeries]
tags: [coding, smells]
image:
  path: header.jpg
  width: 1000   # in pixels
  height: 400   # in pixels
  alt: Photo by Clint McKoy on Unsplash
---
> The secret to high-quality code is following the path of clean code.
{: .prompt-warning }

## Is it a bad approach to have large code files?

Yes, It's bad to have big classes/files in programming. A big file means it is doing many things, and gets modified for multiple reasons. A file should have only one reason to change. It means it should be responsible for one thing.

## How to resolve the issue of big class?

One of the techniques is to have the test in place for the target code for refactoring. If the previous developer has followed TDD, you already have the test code; Otherwise, write enough tests to cover the core functionality of the target code. Perform the refactoring to segregate the target code into different small files until each file serves a single purpose. Rerun the test. Once all test is green, we confidently say, the core functionality is preserved and refactoring is successful.

## Why is my refactoring taking more time?

If there is a refactoring problem, it means your code is complex and has more dependencies on other objects. In the long run, creates an unexpected class that is dangerous to touch and hard to understand called **God Class**. God class means knowing more than it should know and is doing more than it should.

Too much dependency causes code to take more space and time to instantiate. Refactoring takes more time, which indirectly means the target code has poor understandability. Here, most of the time spent in understanding the behavior and flow. It is not good, and refactoring method like method extraction and segregation could take more time than anticipated by the other developer.

Below is an example of an implementation of a wrapper class that aggregates 3 API and expose a single API endpoint to be used by the requester to invoke 3 APIs in sequence.

_file name: LoanCreationWrapper.class(simplified)_
```java
@Autowired ClientService clientService;
@Autowired LoanAccountService loanAccountService;
@Autowired LoanContractService loanContractService;

public WrapperResponse createLoan (WrapperRequest wrapperRequest) {...}
private void prepareWrapperResponse (WrapperResponse response, 
WrapperResponseBody responseBody) {...}   

private static LoanAccountCreation getSkippedLoanAccountResponse (
  WrapperLoanAccountCreationRequest loanAccountData){...}
private static LoanContractCreation getSkippedLoanContractResponse (
  WrapperLoanContractCreationRequest loanContractData){...}
private LoanContractCreation createLoanContract (
  WrapperLoanContractCreationRequest loanContractData,String loanAccountId){...}
private static LoanContractCreation getFailedIrrecoverableLoanContractCreationResponse(
  WrapperLoanContractCreationRequest loanContractData, String message) {...}
private static LoanContractCreation getFailedRecoverableLoanContractCreationResponse (
  WrapperLoanContractCreationRequest loanContractData, String message) {...}
private static LoanContractCreation getSkippedLoanContractCreationResponse (
  WrapperLoanContractCreationRequest loanContractData){...}
private static LoanContractCreation getSuccessLoanContractCreationResponse (
  WrapperLoanContractCreationRequest loanContractData,LoanContractCreationSuccessResponse loanContractCreationServiceResponse){...}
private LoanContractCreationCallerRequest getLoanContractCreationCallerRequest (
  WrapperLoanContractCreationRequest loanContractData,String loanAccountId
  ) throws DataNotFoundException,JsonProcessingException, InternalServerError{...}

private LoanAccountCreation createLoanAccount (
  WrapperLoanAccountCreationRequest loanAccountData,String customerId,
  String losLeadId) {...}
private LoanAccountCreationRequest getLoanAccountCreationCallerRequest (
  WrapperLoanAccountCreationRequest loanAccountData,String customerId,
  String losLeadId) 
  throws DataNotFoundException, InternalServerError{...}
private static LoanAccountCreation getSkippedLoanAccountCreationResponse (
  WrapperLoanAccountCreationRequest loanAccountData){...}
private static LoanAccountCreation getFailedRecoverableLoanAccountCreationResponse(
  WrapperLoanAccountCreationRequest loanAccountData,String message) {...}
private static LoanAccountCreation getFailedIrrecoverableLoanAccountCreationResponse(
  WrapperLoanAccountCreationRequest loanAccountData, String message) {...}
private static LoanAccountCreation getSuccessLoanAccountCreationResponse(
  WrapperLoanAccountCreationRequest loanAccountData,
  LoanAccountCreationSuccessResponse loanAccountCreationServiceResponse){...}
private MandateDetails getMandateDetails(
  WrapperMandateDetails mandateDetails) {...}
private SanctionLimit getSanctionLimit(
  WrapperSanctionLimit sanctionLimit) {...}
private ClientCreditInformation getClientCreditInformation (
  WrapperClientCreditInformation clientCreditInformation) throws DataNotFoundException, InternalServerError {...}
private CreditBureauMilesData fetchCreditBureauData (
  WrapperClientCreditInformation clientCreditInformation) throws DataNotFoundException, InternalServerError{...}
private LoanAccountDetailsRequest getLoanAccountDetails (
  WrapperLoanAccountDetails loanAccountDetails,String customerId, String losLeadId)
  throws DataNotFoundException, InternalServerError{...}
private PaymentModeMilesData fetchPaymentModeData (
  String paymentMode) throws DataNotFoundException, InternalServerError {...}

private ClientCreation createClient 
(WrapperClientCreationRequest clientData) {...}
private static ClientCreation getFailedRecoverableClientCreationResponse (
  WrapperClientCreationRequest clientData, String message) {...}
private static ClientCreation getSkippedClientCreationResponse (
  WrapperClientCreationRequest clientData){...}
private static ClientCreation getSkippedClientCreationResponse (
  WrapperClientCreationRequest clientData){...}
private static ClientCreation getSuccessClientCreationResponse (
  WrapperClientCreationRequest clientData,ClientCreationSuccessResponse clientCreationServiceResponse){...}
private static ClientCreation getFailedIrrecoverableClientCreationResponse (
  WrapperClientCreationRequest clientData,String message) {...}
private ClientCreationCallerRequest getClientCreationCallerRequest (
  WrapperClientCreationRequest clientData) throws InternalServerError, 
  JsonProcessingException, DataNotFoundException {...}
private String fetchOccupationCode (
  WrapperClientCreationRequest clientData) throws DataNotFoundException,
  InternalServerError{...}
private CountryMilesData fetchCountryData (
  WrapperClientCreationRequest clientData) throws DataNotFoundException, 
  InternalServerError {...}
private String fetchProfessionData (
  WrapperClientCreationRequest clientData) throws DataNotFoundException, 
  InternalServerError {...}
private StateMilesData fetchStateData (
  String stateAbbreviation) throws DataNotFoundException, InternalServerError {...}
private List<Bank> getBankDetails (
  List<WrapperClientBankDetails> clientBankDetails) throws
  JsonProcessingException,InternalServerError,DataNotFoundException {...}
private Branch getBranchDetails (
  WrapperClientBranchDetails clientBranchDetails) 
  throws DataNotFoundException, InternalServerError{...}
private BranchData fetchBranchData (
  WrapperClientBranchDetails clientBranchDetails) 
  throws DataNotFoundException, InternalServerError {...}
private SubBroker getSubBrokerDetails (
  WrapperClientSubBrokerDetails clientSubBrokerDetails) 
  throws JsonProcessingException, InternalServerError, DataNotFoundException{...}
private PrimaryRelationshipManager getPrimaryRMDetails (
  WrapperClientPrimaryRM clientPrimarvRM) throws JsonProcessingException, 
  InternalServerError, DataNotFoundException{...}
private GST getGSTDetails (WrapperClientGSTDetails clientGSTDetails) 
  throws DataNotFoundException. InternalServerError{...}
```

As we can observe from the above code snippet, numerous responsibilities are assigned to the above file. Above file knows too many things about the dependency objects. If we see there is some grouping created with space separation to visually indicate that they are not related and serves different purpose based on functionality.

Refactoring of the above code took more than half a day to understand and decide how to segregate the large class, which is knowing too much about the internal of 3 APIs in the smaller classes. Here methods are moved to LoanContractService, LoanAccountService, and ClientService respectively.

Post refactoring above implementation looks like the below,

_file name: LoanCreationWrapper.class_
```java
@Autowired ClientService clientService;
@Autowired LoanAccountService loanAccountService;
@Autowired LoanContractService loanContractService;

public WrapperResponse createLoan (WrapperRequest wrapperRequest) {...}
private void prepareWrapperResponse (WrapperResponse response, 
  WrapperResponseBody responseBody) {...}
```

_file name: ClientService.class_
```java
...
private ClientCreation createClient 
(WrapperClientCreationRequest clientData) {...}
private static ClientCreation getFailedRecoverableClientCreationResponse (
  WrapperClientCreationRequest clientData, String message) {...}
private static ClientCreation getSkippedClientCreationResponse (
  WrapperClientCreationRequest clientData){...}
private static ClientCreation getSkippedClientCreationResponse (
  WrapperClientCreationRequest clientData){...}
private static ClientCreation getSuccessClientCreationResponse (
  WrapperClientCreationRequest clientData,ClientCreationSuccessResponse clientCreationServiceResponse){...}
private static ClientCreation getFailedIrrecoverableClientCreationResponse (
  WrapperClientCreationRequest clientData,String message) {...}
private ClientCreationCallerRequest getClientCreationCallerRequest (
  WrapperClientCreationRequest clientData) throws InternalServerError, 
  JsonProcessingException, DataNotFoundException {...}
private String fetchOccupationCode (
  WrapperClientCreationRequest clientData) throws DataNotFoundException,
  InternalServerError{...}
private CountryMilesData fetchCountryData (
  WrapperClientCreationRequest clientData) throws DataNotFoundException, 
  InternalServerError {...}
private String fetchProfessionData (
  WrapperClientCreationRequest clientData) throws DataNotFoundException, 
  InternalServerError {...}
private StateMilesData fetchStateData (
  String stateAbbreviation) throws DataNotFoundException, InternalServerError {...}
private List<Bank> getBankDetails (
  List<WrapperClientBankDetails> clientBankDetails) throws
  JsonProcessingException,InternalServerError,DataNotFoundException {...}
private Branch getBranchDetails (
  WrapperClientBranchDetails clientBranchDetails) 
  throws DataNotFoundException, InternalServerError{...}
private BranchData fetchBranchData (
  WrapperClientBranchDetails clientBranchDetails) 
  throws DataNotFoundException, InternalServerError {...}
private SubBroker getSubBrokerDetails (
  WrapperClientSubBrokerDetails clientSubBrokerDetails) 
  throws JsonProcessingException, InternalServerError, DataNotFoundException{...}
private PrimaryRelationshipManager getPrimaryRMDetails (
  WrapperClientPrimaryRM clientPrimarvRM) throws JsonProcessingException, 
  InternalServerError, DataNotFoundException{...}
private GST getGSTDetails (WrapperClientGSTDetails clientGSTDetails) 
  throws DataNotFoundException. InternalServerError{...}
```

_file name: LoanAccountService.class_
```java
...
private LoanAccountCreation createLoanAccount (
  WrapperLoanAccountCreationRequest loanAccountData,String customerId,
  String losLeadId) {...}
private LoanAccountCreationRequest getLoanAccountCreationCallerRequest (
  WrapperLoanAccountCreationRequest loanAccountData,String customerId,
  String losLeadId) 
  throws DataNotFoundException, InternalServerError{...}
private static LoanAccountCreation getSkippedLoanAccountCreationResponse (
  WrapperLoanAccountCreationRequest loanAccountData){...}
private static LoanAccountCreation getFailedRecoverableLoanAccountCreationResponse(
  WrapperLoanAccountCreationRequest loanAccountData,String message) {...}
private static LoanAccountCreation getFailedIrrecoverableLoanAccountCreationResponse(
  WrapperLoanAccountCreationRequest loanAccountData, String message) {...}
private static LoanAccountCreation getSuccessLoanAccountCreationResponse(
  WrapperLoanAccountCreationRequest loanAccountData,
  LoanAccountCreationSuccessResponse loanAccountCreationServiceResponse){...}
private MandateDetails getMandateDetails(
  WrapperMandateDetails mandateDetails) {...}
private SanctionLimit getSanctionLimit(
  WrapperSanctionLimit sanctionLimit) {...}
private ClientCreditInformation getClientCreditInformation (
  WrapperClientCreditInformation clientCreditInformation) throws DataNotFoundException, InternalServerError {...}
private CreditBureauMilesData fetchCreditBureauData (
  WrapperClientCreditInformation clientCreditInformation) throws DataNotFoundException, InternalServerError{...}
private LoanAccountDetailsRequest getLoanAccountDetails (
  WrapperLoanAccountDetails loanAccountDetails,String customerId, String losLeadId)
  throws DataNotFoundException, InternalServerError{...}
private PaymentModeMilesData fetchPaymentModeData (
  String paymentMode) throws DataNotFoundException, InternalServerError {...}
private static LoanAccountCreation getSkippedLoanAccountResponse (
  WrapperLoanAccountCreationRequest loanAccountData){...}
```

_file name: LoanContractService.class_
```java
...
private LoanContractCreation createLoanContract (
  WrapperLoanContractCreationRequest loanContractData,String loanAccountId){...}
private static LoanContractCreation getFailedIrrecoverableLoanContractCreationResponse(
  WrapperLoanContractCreationRequest loanContractData, String message) {...}
private static LoanContractCreation getFailedRecoverableLoanContractCreationResponse (
  WrapperLoanContractCreationRequest loanContractData, String message) {...}
private static LoanContractCreation getSkippedLoanContractCreationResponse (
  WrapperLoanContractCreationRequest loanContractData){...}
private static LoanContractCreation getSuccessLoanContractCreationResponse (
  WrapperLoanContractCreationRequest loanContractData,LoanContractCreationSuccessResponse loanContractCreationServiceResponse){...}
private LoanContractCreationCallerRequest getLoanContractCreationCallerRequest (
  WrapperLoanContractCreationRequest loanContractData,String loanAccountId
  ) throws DataNotFoundException,JsonProcessingException, InternalServerError{...}
private static LoanContractCreation getSkippedLoanContractResponse (
  WrapperLoanContractCreationRequest loanContractData){...}
```

Structure of Wrapper response is,
_file name: WrapperResponse.class_
```java
public class WrapperResponse {
// other fields
	private WrapperResponseBody wrapperResponseBody;
}

public class WrapperResponseBody {
	private ClientResponse clientResponse;
	private LoanAccountResponse loanAccountResponse;
	private LoanContractResponse loanContractResponse;
}
```

In short, the wrapper should only invoke the API and collect the responses and return.

Different refactoring techniques used to achieve the above results,

- Extract Method
- Move Method
- Remove Middle Man
- Push Down Method

## Conclusion

When we dirty our hands while working with such code in a real project, it will not be that easy to identify patterns or grouping. It took me time to understand the code and later perform the refactoring. Refactoring could take additional time if the test of the target code is complex. I have experienced one instance, where I had to understand code and tests to perform the refactoring. Due to its complex implementation and testing, it took more time than expected.