---
layout: post
title: Writing Maintainable Tests
subtitle: ''
date: 2019-01-07 00:00:00 +0000
thumb_img_path: ''
content_img_path: ''
excerpt: I recently read XUnit Test Patterns by Gerard Meszaros, and it has changed
  how I approach writing test code.

---
I recently read *[XUnit Test Patterns](http://xunitpatterns.com/index.html)* by Gerard Meszaros, and it has changed how I approach writing test code. When I initially began writing unit tests, I focused primarily on ensuring my production code was well-written, maintainable, and testable. Because I did not give any special consideration towards the maintainability of the test code itself, some of the unit tests I wrote turned out to be a burden when making later changes.

The key principle for writing maintainable test code that I took away from *XUnit Test Patterns* is to **minimize the dependency of the test code on the system under test (SUT)**. All information about the SUT that is irrelevant to a test case should be kept out of the test. Every piece of knowledge that a test contains about the SUT is an additional coupling point between the test and the SUT. The more coupling points a test case has to the SUT, the more likely that a change to the SUT will require a corresponding change to the test, thereby increasing the maintenance burden. A test case should specify only the absolute minimum necessary amount of information about the SUT required to perform the test and no more.

To see how to apply this principle to test code, suppose we are writing tests for a system to manage registrations for running race events such as 5ks and marathons. To test that we can register a runner for a race, we might start off with a test that looks like this:

{% gist 2222114756139a9f23d80e5f4a4b9001 %}

This is what most of my tests looked like when I first started out with unit testing. However, there are some serious problems with a test like this that will lead to a large maintenance burden should we ever change the SUT. This test case knows way too much about the SUT and contains quite a lot of information that is completely irrelevant to what this test is trying to accomplish.

For instance, consider the consequences if we needed to make a change to record a runner's date of birth rather than their age. We would have to update this test case and likely many others to correctly construct a runner with a date rather than a numerical age value, greatly increasing the cost of making such a change. The age of the runner is totally irrelevant to what this test case is trying to accomplish.

*XUnit Test Patterns* describes many useful techniques that we can apply to refactor test code like this into something that is maintainable and readable. The three techniques that I have found most useful to writing maintainable test code are: **creation methods**, **test utility methods**, and **assertion methods**.

## Creation Methods

In the arrange step of a test, we often need to create objects to be used in the test. The process of creating such objects may involve many steps or require a lot of information that is irrelevant to the test. Creation methods encapsulate the logic for creating objects for use in test cases. Creation methods allow test cases to supply only the information relevant to the test case and provide defaults for all other values required to create an object. Creation methods can be reused by many test cases. If changes are made to the SUT which affect how an object is constructed, then only a handful of creation methods need to be updated rather than many dozens of test cases.

For instance, in our example, we might define the following creation methods for creating runner and race objects in test cases:

{% gist bf171839f308f89e02cd63d2f07af9a6 %}

The creation methods provide valid defaults for all values required to create a runner or a race object. In the `RegisterRunner` test case, the particular values used to create the runner and the race are irrelevant, so we simply use all default values. Now, if we ever need to make the change to use the runner's date of birth rather than their numerical age, we would only have to update the one creation method and the test cases could remain exactly the same.

If a test case needs to specify a particular value when creating an object, that value can be provided as a labeled argument to the creation method

{% gist e4614191b19e6e25d010b3dcfcfe7a86 %}

## Test Utility Methods

When the act step of a test is complex or involves irrelevant information about the SUT, test utility methods can be created to abstract the interaction between the test and the SUT. Like creation methods, test utility methods can be reused by many tests and isolate code locations that must be updated whenever the SUT changes.

In our example, the call to `race.RegisterRunner` contains several pieces of irrelevant information such as the runner's T-shirt size and their credit card payment information. If we ever change how payment information is provided, for instance using PayPal rather than credit cards, then we would have to update every test case that calls `race.RegisterRunner` directly. If we are using test utility methods, then we only have to update those utility methods when making changes. We can remove all irrelevant details from our test case by creating a test utility method to register a runner and encapsulate all the details necessary to do so.

{% gist 50263b2532e6dc4c697a6b585f9b13cc %}

## Assertion Methods

Anything more than a non-trivial assertion usually makes the test harder to understand because complex, involved assertions do not clearly reveal *what* the test is checking for and expecting. Assertion methods can be used to factor out non-trivial assertions so that they may be reused with intention-revealing names.

In our example, the assertion involves a few steps which we can factor out into a separate assertion method.

{% gist 702f6dfea80596b689ac67403a2fc3e8 %}

Taken together, the use of the creation methods, test utility methods, and assertion methods makes this test very easy to read and understand. The test case now reads like a specification written in the language of the domain.

## Creating a Test API

Creation methods, test utility methods, and assertion methods all serve to form a *test API* for test cases to use to interact with the SUT. The dependencies of the test cases on the SUT are minimized because the test API provides a layer of indirection between the tests and the SUT. The tests are decoupled from the SUT because they rarely interact directly with the SUT. The test API allows us to make changes to the SUT without requiring updates to dozens of test cases because the test API isolates where updates need to be made for the tests when the SUT changes. This allows tests to serve as valuable regression checks when making changes without incurring a large maintenance burden.