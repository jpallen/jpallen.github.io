---
author: James Allen
title: Unit Testing Node.js
published: false
---

Unit testing makes coding so easy it's boring because you just follow the same
patterns over and over again. This post is about some of the patterns that I've
spotted and internalised.

It took me a long time, and many false starts before I actually groked unit testing.
I don't think this is because it's hard, but instead because I was
never introduced to unit testing in the right way. Almost every guide
I've come across about unit testing focusing on the testing framework and how to do
clever things with mocks, and stubs, and fixtures, and yadda yadda yadda. This is the
wrong approach. To be able to unit test, your code needs to be easily unit testable, and
then the tests follow so easily that there's actually next to nothing involved in
writing them. _To unit test properly, you need to improve your code, not your tests._

General project structure
=========================

Don't use instances of objects
------------------------------

Then we have to mock inside the class and figure out when it gets
instantiated. Less implicit state == good

Small files with a single responsibility
----------------------------------------

Keep context very small, and any calls to other files should have very descriptive
names. Having to include too many external files becomes a sign of too much
responsibility. I.e. including the email file in the database access file is
suspicious.

Method structure
================

One level of error handling
---------------------------

Each method should only have two non-trivial code paths. If it has more then you
end up testing too many input conditions and the tests get messy.

Things done in loops should be encapsulated in a separate method
----------------------------------------------------------------

Checking that 3 different things were done to 3 different objects is pain. Just check
that one method was called on 3 different objects.

Event bindings
--------------

Make the callback function a separate function so that it can tested
in its own right.

What sort of errors unit testing protects you from
--------------------------------------------------

Does:

* Misspelling a variable name (important to catch in a dynamic language)
