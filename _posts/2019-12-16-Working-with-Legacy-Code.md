---
layout: post
title: "Working with Legacy Code"
date: 2019-12-16
categories: [iOS, Mobile App Development, Swift, Programming]
---
Sooner or later, in your career as a software engineer, you are going to have to deal with Legacy Code. In this post, I’m going to illustrate some of the challenges I faced working with Legacy Code and some of the strategies that helped me deal with it.

## What is Legacy  Code anyway?
Let’s start by clarifying what Legacy Code is. There are a few different definitions but, for the sake of this post, I’m going to define Legacy Code as:
1. Source code inherited from someone else.
2. Source code without tests.

Each one of the above definitions focuses on a particular aspect of Legacy Code. In particular:
1. Code inherited from someone else could be very hard to understand because of missing documentation and/or lack of meaningful variable and method names. This is even more challenging if whoever wrote the original code is not around anymore and nobody in your team has deep knowledge of the existing codebase.
2. The lack of tests makes Legacy Code difficult to work with because:
    * Well written tests give developers confidence that changing the code won’t introduce bugs (because that should make tests fail). In a codebase without a meaningful amount of tests it’s really easy to introduce bugs that go unnoticed until users start complaining about them. 
    * Tests also provide an informal documentation for source code, as they show how classes and methods are used. The lack of tests, then, can be seen as another aspect of missing documentation. 

Other common issues with Legacy Code include:
* No coding standards enforced.
* No separation of concerns.
* Extensive use of anti-patterns (the most popular is the abuse of singletons).
* Extensive copypasta (lack of DRYness).

The above issues create a spaghetti code scenario where the lack unit tests is likely caused, among other things, by:
* Untestable business logic.
* Insufficient abstraction.
* Absence of dependency injection.

## How to work with confidence with Legacy Code
Now that we clarified what we define as Legacy Code, I’m going to illustrate some of the strategies that helped me deal with it over the years.

How can we mitigate the risk of introducing bugs when modifying Legacy Code and/or when adding new features on top of it?
The “by the book” suggestion to work with Legacy Code can be summarized as: “write tests for the code you are about to change before you actually start changing it”. This is a great suggestion and, if your team has enough resources to spare, you should definitely write tests before changing the code.

But what should you do if you find yourself in the situation where you can’t delay a new feature by a few weeks to add missing tests? 
If you’re just changing some business logic, there’s a possibility that you can get a reasonable code coverage just by writing a few tests. The likelihood of this scenario is related to how localized are the changes you’re planning to make. If you’re considering changing a single class, chances are good that you are in the above scenario. If your planned changes, instead, involve modifying many classes it’s likely that you won’t be able to achieve a reasonable code coverage in a short amount of time.

Let’s focus on a rather common and even more challenging scenario: Your task is to build an entirely new feature with a brand new UI and functionality that is going to replace the current legacy UI. I’m going to assume that, as it usually happens, adding the missing unit and UI tests before starting on the new code is not feasible. What could you do complete your task with the reasonable confidence that you’re not introducing lots of bugs?

## Feature flags to the rescue
A strategy that worked really well in my experience is to use feature flags. If you make sure to add all new code behind a dedicated feature flag you can build the new feature with a much improved confidence of not introducing lots of bugs. In particular, by adopting a feature flag you can easily:
* Compare the new app behavior (feature flag enabled) and the old app behavior (feature flag disabled). This is really useful both during development and manual testing to make sure you are building the right functionality.
* Turn on and off the feature flag once you release the new app version (this assumes you have a way to fetch feature flags from the network). This allows to easily fallback on the previous functionality in case your users are experiencing issues with the new feature (or in case your new feature requires some additional work).
* Easily track the obsolete code to be removed once the new feature is fully functional and tested.

This can be summarized as:

> **The firewall refactoring technique**  
> We build the new implementation in parallel and once its fully functional  
> we tear down the old one and make the switch!

## Legacy Code and Tech Debt
Legacy Code can be (and I think it should be) considered Tech Debt. This is because making changes inside Legacy Code is way more difficult and time consuming that writing code from scratch. But we should also not underestimate the tendency of even brand new code to become Legacy Code and turn into Tech Debt.
One strategy that worked really well for me in the past is to create a team practice to dedicate a fixed amount of development time to address Tech Debt. There’s likely no one size fits all approach for this but, for instance, allocating an entire quarter or a few sprints throughout the year to reduce Tech Debt should go a long way. After all, shouldn’t we include Continuous Refactoring anyway as part of our Agile practices?

## How do you fix Legacy Code?
As we plan to refactor portions of our Legacy Code, what could we do to maximize the effectiveness of our effort? Answering this question is particularly important within codebases with a large amount of Legacy Code, when we feel the task is daunting.
I’m going to answer to the above question by paraphrasing a well know book:

> How do you fix Legacy Code?
> One class at a time!

In addition to the above, I also think the safest (and likely most effective) way to refactor old code is by adopting a bottom-up size approach: Start with the smallest class first!
Refactoring small classes first should allow you to make progress faster as you can start testing the new behavior a little bit at a time. Leveraging a feature flag (as discussed earlier) is going to help quite a bit through the entire process since you will be able to keep the old code around until the entire functionality has been refactored and fully tested.

## To summarize
The above illustrated approach to working with Legacy Code can be summarized with the following recipe:
1. Target the smaller classes first.
2. Create a new feature flag to support switching between old and new implementation.
3. Write the new class (Swift) and test its functionality (A feature flag allows easy functionality comparison).
4. Once functionality is fully tested, remove feature flag and old class.
5. Rinse and repeat!
