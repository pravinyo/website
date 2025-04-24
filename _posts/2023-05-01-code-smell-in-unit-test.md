---
title: "Code Smell Series: Unit Test"
author: pravin_tripathi
date: 2023-05-01 22:30:00 +0800
categories: [Blogging, CodeSmellSeries]
tags: [coding, smells]
media_subpath: /assets/img/code-smell-series/unit-test/
image:
  path: header_test.jpg
  width: 1920   # in pixels
  height: 1280   # in pixels
  alt: Photo by Testalize.me on Unsplash 
---
> The secret to high-quality code is following the path of clean code.
{: .prompt-warning }

A code will remain at its highest quality if it has an understandable and meaningful test. Robert C. Martin in his book, “Clean Code: A Handbook of Agile Software Craftsmanship” mentioned a very nice acronym for clean code in unit testing.

## **F.I.R.S.T.**
As per clean code practice, all tests follow the five rules below that form the `F.I.R.S.T.` acronym. The below explanation is taken from the Clean Code Book by Robert C. Martin,

**<u>Fast</u>**: Tests should be fast. They should run quickly. When tests run slow, you won’t want to run them frequently. If you don’t run them frequently, you won’t find problems early enough to fix them easily. You won’t feel as free to clean up the code. Eventually, the code will begin to rot.

**<u>Independent</u>**: Tests should not depend on each other. One test should not set up the conditions for the next test. You should be able to run each test independently and run the tests in any order you like. When tests depend on each other, then the first one to fail causes a cascade of downstream failures, making diagnosis difficult and hiding downstream defects.

**<u>Repeatable</u>**: Tests should be repeatable in any environment. You should be able to run the tests in the production environment, in the QA environment, and on your local development machine without a network. If your tests aren’t repeatable in any environment, then you’ll always have an excuse for why they fail. You’ll also find yourself unable to run the tests when the environment isn’t available.

**<u>Self-Validating</u>**: The tests should have a boolean output. Either they pass or fail. You should not have to read through a log file to tell whether the tests pass. You should not have to manually compare two different text files to see whether the tests pass. If the tests aren’t self-validating, then failure can become subjective, and running the tests can require a long manual evaluation.

**<u>Timely</u>**: The tests need to be written in a timely fashion. Unit tests should be written just before the production code that makes them pass. If you write tests after the production code, then you may find the production code to be hard to test. You may decide that some production code is too hard to test. You may not design the production code to be testable.

## **Rule 1: Single Concept per Test**

As per this rule, a single test function should test a single thing/concept. We don’t want long test functions that go testing one miscellaneous thing after another. Below code snippet is an example of such a test. This test function should be segregated into three independent tests because it tests three independent things. Merging them all together into the same function forces the reader to figure out why each section is there and what is being tested by that section.

Example of bad test,
```java
/**
* tests for the addMonths() method.
*/
@Test
public void should_add_Months() {
    var d1 = CustomDate.createInstance(31, 5, 2004);

    var d2 = CustomDate.addMonths(1, d1);
    
    assertEquals(30, d2.getDayOfMonth());
    assertEquals(6, d2.getMonth());
    assertEquals(2004, d2.getYYYY());

    var d3 = CustomDate.addMonths(2, d1);

    assertEquals(31, d3.getDayOfMonth());
    assertEquals(7, d3.getMonth());
    assertEquals(2004, d3.getYYYY());

    var d4 = CustomDate.addMonths(1, CustomDate.addMonths(1, d1));

    assertEquals(30, d4.getDayOfMonth());
    assertEquals(7, d4.getMonth());
    assertEquals(2004, d4.getYYYY());
}
```

## **Rule 2: One Assert per Test**

This rule says that every test function in a JUnit test should have one and only one assert statement. This rule may seem harsh, but the advantage can be seen in the below code snippet. Those tests come to a single conclusion that is quick and easy to understand.

Let's see one code example to understand this rule.  
Before:

```java
@Test
public void should_return_page_hierarchy_as_json() throws Exception {
    makePages("PageOne", "PageOne.ChildOne", "PageTwo");
    submitRequest("root", "type:pages");
    assertResponseIsJson();
    assertResponseContains(
        "{
            {\"page\": \"PageOne\"},
            {\"page\": \"PageTwo\"},
            {\"page\": \"ChildOne\"}
        }"
    );
}
```

After:

```java
@Test
public void should_return_page_hierarchy_as_json() throws Exception {
    givenPages("PageOne", "PageOne.ChildOne", "PageTwo");
    whenRequestIsIssued("root", "type:pages");
    thenResponseShouldBeJson();
}

@Test
public void should_return_page_hierarchy_with_right_tags() throws Exception {
    givenPages("PageOne", "PageOne.ChildOne", "PageTwo");
    whenRequestIsIssued("root", "type:pages");
    thenResponseShouldContain(
        "{
            {\"page\": \"PageOne\"},
            {\"page\": \"PageTwo\"},
            {\"page\": \"ChildOne\"}
        }"
    );
}
```

## **Do we have any real examples?**
Now, let’s see another example that is taken from the production code. Below is the code snippet of the unit test.

```java
@Test
void test_something() {
    // Given
    ...

    // Action
    var wrapperResponse = loanCreationWrapperService.createLoan(wrapperRequest);

    // Assert
    var response = wrapperResponse.getWrapperResponseBody();
    assertEquals (Status.SUCCESS, wrapperResponse.getStatus());
    assertEquals ("ID0001", wrapperResponse.getLosLeadId());
    assertEquals (Status.SUCCESS.getCode(), wrapperResponse.getCode());
    assertNull (wrapperResponse.getFailureType());

    var clientCreationResponse = response.getClientCreation();
    assertEquals (Status.SKIPPED, clientCreationResponse.getStatus());
    assertEquals ("10001", clientCreationResponse.getCustomerId());
    assertEquals ("LOS0001", clientCreationResponse.getUniqueRecordId());
    assertEquals (Status.SKIPPED.getCode(),clientCreationResponse.getCode());
    assertNull (clientCreationResponse.getFailureType());

    var loanAccountCreationResponse =response.getLoanAccountCreation();
    assertEquals (Status.SKIPPED, loanAccountCreationResponse.getStatus());
    assertEquals ("10001", loanAccountreationResponse.getLoanAccountId());
    assertEquals ("1234", loanAccountCreationResponse.getUniqueRecordId());
    assertEquals (Status.SKIPPED.getCode(), loanAccountCreationResponse.getCode());
    assertNul1 (loanAccountCreationResponse.getFailureType());

    var loanContractCreationResponse = response.getLoanContractCreation();
    assertEquals (Status.SUCCESS, loanContractCreationResponse.getStatus());
    assertEquals ("10001", loanContractCreationResponse.getLoanContractId());
    assertEquals ("XXX123GYU", loanContractCreationResponse.getUniqueRecordId());
    assertEquals (Status.SUCCESS.getCode(),loanContractCreationResponse.getCode());
    assertNull (loanContractCreationResponse.getFailureType());
}
```

As you can see, the above test code is very complex and has too many assertions. This test is hard to understand. As a matter of fact, the code that this test is written for is also very complex. As we know test is a documentation for the project/service, so the above test will confuse more developers instead of helping when things go bad.
> Let’s apply the earlier 2 rules discussed in the above test code.
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
    assertEquals ("ID0001", wrapperResponse.getLosLeadId());
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
The above-refactored code looks pretty nice compared to the earlier test, but still, there are many asserts.
> A test should have at max 5 assertions per function but as a best practice, it should have a single assertion per test. **<u>if assertions count > 5, it means you are doing something different and it is complex.</u>**
{: .prompt-warning }

Following a single assertion per test rule, we can further decompose our test. Let’s take **test_something_for_client_creation** test in the last step and further break it down into smaller tests.

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

## **Rule 3: Tests should not depend on each other**

As a general rule, each test function should contain all the code and resources that it requires to test the piece of code. A test function is a mini-universe that contains all things it needs to test the piece of code. For a particular class under test, there will be many mini-universe. A failure in one test shouldn’t affect the other test directly or indirectly.

Example: In the below code snippet, we are using a static resource called Security context. It is used to handle user authentication and extract user details from the token. If we see carefully, we are creating a static mock of the `SecurityContext.class` and using the object to mock the method response.

```java
@Test
void test_something() {
    ...
    var securityContext = Mockito.mockStatic(SecurityContext.class);
	securityContext.when(SecurityContext::getUserId).thenReturn(CUSTOMER_ID);
    ...
    securityContext.close();
}
```
If the above test fails and fails to release the securityContext static mock object, It will cause failure in other tests. And debugging will be tough if we are not aware of this problem. If you are not following proper development practices, there is a good chance that your code will be having similar issues. One way to solve the above problem is to use `try-with-resources` statement to handle this failure. Refer to the code snippet for the above code refactored to use try-with-resources statement.

```java
@Test
void test_something() {
	try (var securityContext = Mockito.mockStatic(SecurityContext.class)) {
	securityContext.when(SecurityContext::getUserId).thenReturn(CUSTOMER_ID);
	...
	}
}
```

### The common reason for occurring this issue:
- Not following correct testing practices.
- Not releasing the static mock resource after completion of the test.
- Not handling commonly shared resources in testing.

## **Conclusion**

Based on my observation, many developers generally don’t follow proper development practices. They either write production code before test code or don’t give importance to testing and put less effort into writing such tests. I have witnessed such a project, where the developer after a few years of development of the feature, hesitates to add new modifications with the excuse that code is complex, and he doesn’t have the context to add changes to that feature. It is a disaster for the project.

It would be a lot better if proper development practices were followed in the project to minimize such an impact. Practices such as <u>Pair Programming, Mob Review, Test Driven Development, Code Review, and Code Documentation</u> are some practices to ensure high-quality code and less time to deploy new changes in production.

> You have something to share, go ahead and add in comments 😉 » Happy Learning!!
{: .prompt-tip }