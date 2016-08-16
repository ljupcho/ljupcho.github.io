---
layout: post
title:  "On Unit Testing"
date:   2016-08-15 18:12:10 +0200
tags: phpunit testing
comments: true
--- 

Writing unit tests is divided into two categories: TDD (Test Driver Development) and BDD (Behavior Driver Development). But, first let me write some things about its importance positive or negative.
When I get a requirement I dive into the implementation previously having the design of the classes and the databases in mind or simple wrote on paper. Nobody has time for writing tests. That's the first mistake I usually make. Before the requirement is finalized it is being estimated together with business analysts and managers. So, having in mind that we need to put some extra hours for the writing of tests, the estimations might become bigger and scare the client, even more don't accept that release since usually developers are charged by hours.

In case we don't have written tests, why can't we just run the application in debug mode and see what happens because we think tests should allow us easily find a bug? If we write test, than change something in the code logic, we have to change the test as well? That seems like a lot of work when we can just throw the bug to the developer that work on that logic and fix it himself.

In order not to have the time duplicated once for developing the logic and once for writing the tests, TDD was invented, so the developer can start from writing the test, make it fail, correction, make it pass, move to another test and repeat. Failing the test will guide the developer in writing the code. This might be good if you don't have to run the test so frequently. This is important because if you would write the logic first than you try to write some tests you always end up into refactoring your code and in the very beginnings rewriting the code.

But, I can't just simple start from tests, because what I usually have to do when writing a functionality is think of the design, it means I would think what design patterns I would use, how I would normalize my database, what services I would use. Having the TDD approach is just to make sure your tests pass so you don't have to rewrite the code afterwards. The tests themselves are not there just to be run months latter so we know what piece of code is broken, they would help us predefined out approach when starting to write the code, because we usually might find ourselves not sure what path we should take and have different approaches and instead of convincing ourselves months letter that we took the right approach we can write tests before hand to get the sense of the problem, knowing the path and walking the path are two different things so we might come to a better understanding of the problem and making the decision with more confidence. That would be BDD which is in my opinion more important and probably more time consuming. 

Having the estimations in an appropriate time frame is trade-off such as the technical part of it, writing the tests before or having the skeleton of the component done and do some tests before and some after.

Another thing about this is that I would use already tested component or framework I've done and use generic logic that I know I have tested and I know it is working and just copy the usage example and tweak the input arguments, so when writing a component I would always try to push the logic into generic as far as I can, make it simple, test it and be able to re-use it.

Writing tests is very good for understating how the component is working, like a example so developers can easily understand what the component is meant to do and how it is used. It's just like a short documentation and how-to, like every other test, code based or not.

