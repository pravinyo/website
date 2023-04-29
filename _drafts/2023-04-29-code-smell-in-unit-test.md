---
title: "Code Smell Series: Unit Test"
author: pravin_tripathi
date: 2023-04-29 00:34:00 +0800
categories: [Blogging, CodeSmellSeries]
tags: [coding, smells]
published: true
---
> The secret to high quality code is following the path of clean code.
{: .prompt-warning }

A code will remain in it's highest quality if it has understandable and meaningful test. Robert C. Martin in his book, "Clean Code: A Handbook of Agile Software Craftsmanship" have mentioned a very nice acronym for clean code in unit testing.

## **F.I.R.S.T.**
Clean tests follow five other rules that form the above acronym:

**<u>Fast</u>**: Tests should be fast. They should run quickly. When tests run slow, you won’t want to run them frequently. If you don’t run them frequently, you won’t find problems early enough to fix them easily. You won’t feel as free to clean up the code. Eventually the code will begin to rot.

**<u>Independent</u>**: Tests should not depend on each other. One test should not set up the conditions for the next test. You should be able to run each test independently and run the tests in any order you like. When tests depend on each other, then the first one to fail causes a cascade of downstream failures, making diagnosis difficult and hiding downstream defects.

**<u>Repeatable</u>**: Tests should be repeatable in any environment. You should be able to run the tests in the production environment, in the QA environment, and on your laptop while riding home on the train without a network. If your tests aren’t repeatable in any environment, then you’ll always have an excuse for why they fail. You’ll also find yourself unable to run the tests when the environment isn’t available.

**<u>Self-Validating</u>**: The tests should have a boolean output. Either they pass or fail. You should not have to read through a log file to tell whether the tests pass. You should not have to manually compare two different text files to see whether the tests pass. If the tests aren’t self-validating, then failure can become subjective and running the tests can require a long manual evaluation.

**<u>Timely</u>**: The tests need to be written in a timely fashion. Unit tests should be written just before the production code that makes them pass. If you write tests after the production code, then you may find the production code to be hard to test. You may decide that some production code is too hard to test. You may not design the production code to be testable.

## **Single Concept per Test**

Perhaps a better rule is that we want to test a single concept in each test function. We don’t want long test functions that go testing one miscellaneous thing after another. Listing 9-8 is an example of such a test. This test should be split up into three independent tests because it tests three independent things. Merging them all together into the same function forces the reader to figure out why each section is there and what is being tested by that section.

Example of bad test,
```java
/**
* Miscellaneous tests for the addMonths() method.
*/
public void testAddMonths() {
    SerialDate d1 = SerialDate.createInstance(31, 5, 2004);
    SerialDate d2 = SerialDate.addMonths(1, d1);
    assertEquals(30, d2.getDayOfMonth());
    assertEquals(6, d2.getMonth());
    assertEquals(2004, d2.getYYYY());

    SerialDate d3 = SerialDate.addMonths(2, d1);
    assertEquals(31, d3.getDayOfMonth());
    assertEquals(7, d3.getMonth());
    assertEquals(2004, d3.getYYYY());

    SerialDate d4 = SerialDate.addMonths(1, SerialDate.addMonths(1, d1));
    assertEquals(30, d4.getDayOfMonth());
    assertEquals(7, d4.getMonth());
    assertEquals(2004, d4.getYYYY());
}
```

## One Assert per Test

There is a school of thought that says that every test function in a JUnit test should have one and only one assert statement. This rule may seem draconian, but the advantage can be seen in Listing 9-5. Those tests come to a single conclusion that is quick and easy to understand.

From:

```java
public void testGetPageHierarchyAsXml() throws Exception {
    makePages("PageOne", "PageOne.ChildOne", "PageTwo");
    submitRequest("root", "type:pages");
    assertResponseIsXML();
    assertResponseContains(
        "<name>PageOne</name>", "<name>PageTwo</name>", "<name>ChildOne</name>");
}
```

To:

```java
public void testGetPageHierarchyAsXml() throws Exception {
    givenPages("PageOne", "PageOne.ChildOne", "PageTwo");
    whenRequestIsIssued("root", "type:pages");
    thenResponseShouldBeXML();
}
public void testGetPageHierarchyHasRightTags() throws Exception {
    givenPages("PageOne", "PageOne.ChildOne", "PageTwo");
    whenRequestIsIssued("root", "type:pages");
    thenResponseShouldContain(
        "<name>PageOne</name>", "<name>PageTwo</name>", "<name>ChildOne</name>");
}
```

Example from recent project,

```java
@Test
void test_something() {
    // Given
    ...

    // Action
    var wrapperResponse = loanCreationWrapperService.createLoan(wrapperRequest);

    // Assert
    var response = wrapperResponse.getWrapperResponseBody();
    var clientCreationResponse = response.getClientCreation() ;
    var loanAccountCreationResponse =response.getLoanAccountCreation();
    var loanContractCreationResponse = response.getLoanContractCreation();

    assertEquals (Status.SUCCESS, wrapperResponse.getStatus());
    assertEquals ("LAMF0001", wrapperResponse.getLosLeadId());
    assertEquals (Status.SUCCESS.getCode(), wrapperResponse.getCode());
    assertNull (wrapperResponse.getFailureType());

    assertEquals (Status.SKIPPED, clientCreationResponse.getStatus());
    assertEquals ("10001", clientCreationResponse.getCustomerId());
    assertEquals ("LOS0001", clientCreationResponse.getUniqueRecordId());
    assertEquals (Status.SKIPPED.getCode(),clientCreationResponse.getCode());
    assertNull (clientCreationResponse.getFailureType());

    assertEquals (Status.SKIPPED, loanAccountCreationResponse.getStatus());
    assertEquals ("10001", loanAccountreationResponse.getLoanAccountId());
    assertEquals ("1234", loanAccountCreationResponse.getUniqueRecordId());
    assertEquals (Status.SKIPPED.getCode(), loanAccountCreationResponse.getCode());
    assertNul1 (loanAccountCreationResponse.getFailureType());

    assertEquals (Status.SUCCESS, loanContractCreationResponse.getStatus());
    assertEquals ("10001", loanContractCreationResponse.getLoanContractId());
    assertEquals ("XXX123GYU", loanContractCreationResponse.getUniqueRecordId());
    assertEquals (Status.SUCCESS.getCode(),loanContractCreationResponse.getCode());
    assertNull (loanContractCreationResponse.getFailureType());
}
```

> After applying 2 rules discussed above,
{: .prompt-tip }

```java
...

@Test
void test_wrapper_status_for_something() {
    // Given
    ...

    // Action
    var wrapperResponse = loanCreationWrapperService.createLoan(wrapperRequest);

    // Assert
    assertEquals (Status.SUCCESS, wrapperResponse.getStatus());
    assertEquals (Status.SUCCESS.getCode(), wrapperResponse.getCode());
    assertEquals ("LAMF0001", wrapperResponse.getLosLeadId());
    assertNull (wrapperResponse.getFailureType());
    }

    @Test
    void test_client_creation_for_something() {
    // Given
    ...

    // Action
    var wrapperResponse = loanCreationWrapperService.createLoan(wrapperRequest);

    // Assert
    var response = wrapperResponse.getWrapperResponseBody();
    var clientCreationResponse = response.getClientCreation() ;

    assertEquals (Status.SKIPPED, clientCreationResponse.getStatus());
    assertEquals (Status.SKIPPED.getCode(),clientCreationResponse.getCode());
    assertEquals ("10001", clientCreationResponse.getCustomerId());
    assertEquals ("LOS0001", clientCreationResponse.getUniqueRecordId());
    assertNull (clientCreationResponse.getFailureType());
}

@Test
void test_loan_account_creation_for_something() {
    // Given
    ...

    // Action
    var wrapperResponse = loanCreationWrapperService.createLoan(wrapperRequest);

    // Assert
    var response = wrapperResponse.getWrapperResponseBody();
    var loanAccountCreationResponse =response.getLoanAccountCreation();

    assertEquals (Status.SKIPPED, loanAccountCreationResponse.getStatus());
    assertEquals (Status.SKIPPED.getCode(), loanAccountCreationResponse.getCode());
    assertEquals ("10001", loanAccountreationResponse.getLoanAccountId());
    assertEquals ("1234", loanAccountCreationResponse.getUniqueRecordId());
    assertNul1 (loanAccountCreationResponse.getFailureType());
}

@Test
void test_loan_contract_Creation_for_something() {
    // Given
    ...

    // Action
    var wrapperResponse = loanCreationWrapperService.createLoan(wrapperRequest);

    // Assert
    var response = wrapperResponse.getWrapperResponseBody();
    var loanContractCreationResponse = response.getLoanContractCreation();

    assertEquals (Status.SUCCESS, loanContractCreationResponse.getStatus());
    assertEquals (Status.SUCCESS.getCode(),loanContractCreationResponse.getCode());
    assertEquals ("10001", loanContractCreationResponse.getLoanContractId());
    assertEquals ("XXX123GYU", loanContractCreationResponse.getUniqueRecordId());
    assertNull (loanContractCreationResponse.getFailureType());
}

...
```

> A test should have at max 5 assertion per function but as a best practice it should include single assertion per test. **if assertions count > 5, it means you are doing something different and it is complex.**
{: .prompt-warning }

Following single assertion per test principle, we can further decompose our test. Let’s take **test_something_for_client_creation** test in last step and further break it down to smaller test.

```java
@Test
void test_client_creation_that_has_skipped_status_for_something() {
    // Given
    ...

    // Action
    var wrapperResponse = loanCreationWrapperService.createLoan(wrapperRequest);

    // Assert
    var response = wrapperResponse.getWrapperResponseBody();
    var clientCreationResponse = response.getClientCreation() ;

    assertEquals (Status.SKIPPED, clientCreationResponse.getStatus());
}

@Test
void test_client_creation_that_has_skipped_status_code_for_something() {
    // Given
    ...

    // Action
    var wrapperResponse = loanCreationWrapperService.createLoan(wrapperRequest);

    // Assert
    var response = wrapperResponse.getWrapperResponseBody();
    var clientCreationResponse = response.getClientCreation() ;

    assertEquals (Status.SKIPPED.getCode(),clientCreationResponse.getCode());
}

@Test
void test_client_creation_that_has_customerId_for_something() {
    // Given
    ...

    // Action
    var wrapperResponse = loanCreationWrapperService.createLoan(wrapperRequest);

    // Assert
    var response = wrapperResponse.getWrapperResponseBody();
    var clientCreationResponse = response.getClientCreation() ;

    assertEquals ("10001", clientCreationResponse.getCustomerId());
}

@Test
void test_client_creation_that_has_uniqueRecordId_for_something() {
    // Given
    ...

    // Action
    var wrapperResponse = loanCreationWrapperService.createLoan(wrapperRequest);

    // Assert
    var response = wrapperResponse.getWrapperResponseBody();
    var clientCreationResponse = response.getClientCreation() ;

    assertEquals ("LOS0001", clientCreationResponse.getUniqueRecordId());
}

@Test
void test_client_creation_that_has_success_failure_type_for_something() {
    // Given
    ...

    // Action
    var wrapperResponse = loanCreationWrapperService.createLoan(wrapperRequest);

    // Assert
    var response = wrapperResponse.getWrapperResponseBody();
    var clientCreationResponse = response.getClientCreation() ;

    assertNull (clientCreationResponse.getFailureType());
}

... // adding similarly for other test
```

## **Tests should not depend on each other**

As a general rule, each test function should contain all the code and resources that it requires to test the piece of code. A failure in one test shouldn’t affect the other test directly or indirectly.

Example of static mock:

```java
@Test
void test_something() {
	try (var securityContext = Mockito.mockStatic(SecurityContext.class)) {
	securityContext.when(SecurityContext::getUserId).thenReturn(CUSTOMER_ID);
	...
	}
}
```

Common reason for occurring this issue:

- Not following correct testing practice.
- Not releasing the static mock resource after completion of test
- Not properly handling commonly shared resources in testing
