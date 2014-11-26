---
title: Unit Tests vs Acceptance Tests
layout: post
---
There has been [some discussion](http://software-carpentry.org/blog/2014/10/why-we-dont-teach-testing.html) in the Software Carpentry community about teaching testing as part of their workshops. Actually the discussion was focused around why testing isn't part of the standard curriculum, but I think a lot of the problem comes from confusing different types of tests and their applicableness. The tendency to lump everything together under the umbrella of 'automated testing' means that some very accessible and appropriate forms of testing get ignored because of problems with other types of tests. In my day to day work, I deal with two types of test, *unit tests* and *acceptance tests*, and this article is about understanding the differences.

Unit tests and acceptance tests need fundamentally different approaches in how we think about them, how we write them and how we teach them.

## Unit tests

Unit tests check that your code does was you think it does, but not whether that is the correct thing to do. A good unit test checks that you have implemented the low level structure of your functions (or ‘units’) without any straight-forward errors. These types of errors are things that every programmer, in every discipline, is familiar with:

* Mistakes in syntax or variable names;
* Off-by-one errors in loops or array indices;
* Edge cases like empty input, division by zero or handling NaNs;
* Transcription errors when translating equations into code. It’s easy to make sign changes, miss out factors, use the wrong indices, etc.

The fundamental building blocks of a programming language are the same regardless of the context you’re using it in, so unit testing takes the same approach everywhere.

Unit testing is a replacement for the process of manually testing your function after writing it. Without unit testing, you might load your code into a console environment, and check that it behaves as expected for different input. If your language doesn’t have a console environment, you might try to call your program in a way that will trigger your function with the inputs you want to test. Testing edge cases like this can be very hard if not impossible. With unit testing, we replace this manual process with an automatic process. We still go through the process of thinking of good example input to test our function and its edge cases, but instead of testing it manually we write the tests in code. Automation allows us to run our tests quickly and repeatedly every time we make even a small change to the function.

Unit testing should be seen and taught as an integral part of the development process. If you write your functions first and then think about trying to test them afterwards, all you will discover is that you’ve written functions that are very hard to test. Instead we should think about how we’re going to test our code before we write a single line. This will force us to structure code in very small ‘units’ that do a single job. We can then easily test that each of these units do their single job as expected. These units can be built up into larger units, with tests written to check that the correct lower level functions (or ‘units’) are called as required.

With unit testing, the fundamental question of ‘how do I test this code?’ is the wrong question. Instead we should be asking (and teaching) ‘how do I write this code in a way that is testable?’

There are some quick heuristics to see if we’re dealing with unit tests:

* They don’t read in any external data. Each test case is small enough that small sample inputs can be provided inline when testing the function, and the outputs of each individual function can be verified in a simple way.
* Your entire test suite can run in less than 60 seconds. Each unit test will only test a tiny amount of code so will run very quickly. Even if you have thousands of them, a computer can crunch through them quickly.
* You have the same order of magnitude of test code to program code. Since each function in your program code is small, and has a corresponding unit test, you’ll need to write similar amounts of test code to program code. In fact, in my experience I write 1-2x more test code than program code. However, the test code is much quicker to write since it’s ‘dumb’.

## Acceptance tests

On the other hand, acceptance tests check that the combination of your little ‘units’ of code is doing something that is overall sensible and correct for your use case. You’ve verified that your code does what you expect it to with unit tests, but acceptance tests help to check that your mental model is correct, and that the code you’ve written actually solves the problem you set out to solve. For example, you could write a perfect implementation of a soil temperature model using Kelvin as your temperature units, only to find out that you input data is in Celsius and so your output is clearly physically wrong despite being the ‘correct’ output for your program.

What makes for a sensible acceptance test will depend on your discipline and the purpose of your code. Perhaps you need to check that the output of a simulation is physically reasonable and satisfies certain conversation laws. Perhaps you need to sanity check that your data cleaning code is not affecting the good parts of your data. It will completely depend on context, and that makes it hard to talk about in general. However, I think there are some universal points which can be said about acceptance testing.

Unlike unit testing, acceptance testing is not a replacement for manually verifying the result of running your code. Typically in a complex situation, you will be doing some manual checks to make sure that the result is sensible. Does it pass the ‘gut instinct’ test? Does it match what you understand about the system and expected result? Does it fit with anything you can check analytically? Does the data match observations in the expected way? And so on depending on your field.

Once you’re happy that your program produces the correct output for your inputs, you should pause and encode this in an automated acceptance test. The acceptance test will basically say ‘given this input, does my program produce this output’. The purpose of this test is to protect against unexpected changes in the behaviour of your program in the future. If you refactor your code to be structured differently, or use a different internal data format, or extend your algorithms from 2 dimensions to arbitrary dimensions, your acceptance tests allow you to quickly verify that the behaviour of your code hasn’t changed in a way that you didn’t expect.

Acceptance tests can be fragile, and that’s ok. If you program changes something small in its output, then it’s ok for your acceptance tests to start failing. This will make sure that your attention is drawn to any changes in the way your program behaves. If you expected the change to happen you can quickly fix up your acceptance tests to match the new output. If you didn’t expect the change then you can investigate further and see whether it’s an acceptable change, or whether it’s the result of an unexpected bug or problem.

While unit testing is a fundamental part of a good development workflow, acceptance testing is something that will typically be done after the main body of code has been written and you’re ready to start testing whether the whole thing behaves correctly. 

Some quick heuristics to see if we’re dealing with acceptance tests:

* Your example inputs and outputs are complex enough to be stored externally from your test code.
* Your acceptance tests takes a while to run since they have to run a non-trivial amount of your program code to actually test whether it’s doing what you expect it to.
* Your acceptance test code is an order of magnitude smaller in size than your project code. Verifying a correct solution is typically much easier than finding the correct solution in the first place.

(NB. Take everything below here with a pinch of salt because I don’t have much direct experience teaching these things).

It should be possible to teach unit testing to anyone from any discipline since the methodology and principles apply to any sort of programming. To go further, I don’t think we can separate ‘writing unit testable code’ from ‘writing reusable, modular, and readable code’. They are the same thing, and building up a mental model of programming that involves always asking ‘how can I write this code so that I can test it?’ will teach students how to write a higher quality of code, even if the tests are never written (but of course they will be!)

Teaching acceptance testing is harder, because it will depend on the specific code and discipline. However, I think acceptance testing doesn’t need to be taught in as much depth as unit testing because the general concepts are easier to state. Acceptance testing is an exercise in testing whether your program matches a certain input to a certain output. The definition of ‘input’, ‘output’ and ‘program’ will vary for everyone but the ingredients are all there once students are familiar with writing code, reading data, and automating tests.



