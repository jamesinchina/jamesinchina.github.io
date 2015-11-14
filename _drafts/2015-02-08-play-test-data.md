---
layout: post
title: Manage test data intelligently and stay sane (using Play2, Slick and H2DB)
---


The Problem
-----------

If we're good TDD programmers then we'll be unit testing all our data access code.  It might seem logical to include the test data with each unit test, but we soon find that very similar data for frequently used tables (I'm lookin' at you, `users` table) is inserted in many tests with much repetition.  Further, a core part of development is smoke testing the applicat (starting up the whole application stack and logging in as a user to see how the unit-test-passed code you just wrote actually plays out in practice). This requires yet more test data.  Not to mention integration tests (moar test data), functional tests (moar moar moar).. you get the idea.  Test data here, there and everywhere.  Some in version control, some in databases on developer machines.  Some is SQL and some in Scala test cases.  Some written in Python for Selenium.  Then the data model changes, and EVERYTHING needs updating.  Cue hair-tearing general pandemonium.  There must be a better way.

Things we tried
---------------

Here are some approaches I've seen tried before that make me wince now looking back.  For those asking whether I thought they were a good idea at the time... I maintain plausible deniability.

1) ## Let each developer reinvent the wheel

With a lack of any coherent policy on test data, every developer keep his own private .sql file of test data for smoke testing.  Every time the data model changes, every developer updates his own test data, wasting hours.  As it isn't under version control, it becomes impossible to run older versions of the app.
Developers rewrite test data generation code similar to that already entered many times in the code base, but subtly different enough each time that extracting all that commonality would be a vast amount of work.  Updating the data model becomes an exercise in masochism.

2) ## Real data is the best data

The entire production data for the application is copied onto developer machines, under the laudable but misguided perception that the closer the data is to real users, the better developers will understand their needs.  Apart from the data protection issues this raises (even after user personal data has been randomized and passwords deleted), this quickly becomes untenable.  The size of the data after the application has been in production a few months leads to wasted time as the massive data file is transferred, and then more wasted time as scripts are written to pare the data down to a representative sample and then more as bugs with those scripts cause problems down the line.  Even if the data is kept under version control, data from one version of the app can't be applied to another version. Fail.  Caveat: some bugs are extremely difficult to track down without replicating production data, but after the bug is localized and a failing unit test added, the production data is no longer needed.

3) ## Keep it in Git

A big .sql file of data for smoke testing gets checked into version control.  This is a big improvement on (1) and (2) as now only one file needs to be updated when the model changes, it is of reasonable size, and there is at least one version of the test data which can be applied to any given version of the application.  However, it is still stored as a big file of INSERT statements, and is fragile to any modifications of the data model.  And the problem of duplicated work in generating data for unit tests is still unsolved.

4) ## Use your smoke test data for unit tests.

Lightbulb! Why not use your smoke test data for unit tests and eliminate all that example generation code? Add a `beforeEach` function to each test suite that loads the data from the SQL file and sends it straight to the database.  Wipe it away on `afterEach`. Nice idea, but doesn't play out so well in practice.  As the codebase grows, changes to the test data will start causing unrelated tests to fail, increasing in number at O(n<sup>2</sup>) with the size of the codebase.  Tests are awkwardly written to fit the existing test data.  Because so much now depends on it, updating the test data file becomes very dangerous.  As the test data groweth, the unit tests sloweth.
Also, dealing with several thousand lines of SQL insert statements is just plain unpleasant. Things you already wrote functions to deal with (for example: generating normalized search keys) you have to do again, by hand.  Wouldn't it be nicer to deal with this in code?


Spoiler
-------

For the TL;DR crowd who got this far, this is a sketch of the final solution we settled on for the [Schoolshape](http://schoolshape.com) app in production which has over 150 different data types in separate tables and has been developed continuously over more than 5 years.

- Generate test data in code, reusing your existing data access code and decoupling from the final data representation in SQL.
- Split test data into a logical hierarchy, allowing unit tests to pick and choose the minimum needed.
- Parameterize the test data, introducing flexibility for both automated and smoke testing.
- Share test data across all kinds of automated and manual testing.
- Do all kinds of manual and automated testing against an in-memory database which is as close to your production environment as possible.

The Lowdown
-----------

Here I'll share the arcane incantations required to make this hum using the Play framework, MySQL as the production database and H2DB as the development database.  If you use different technologies your mileage may vary, but the same principles should apply.

For this example let's assume we're running a pet store

````scala
case class Pet {
    val name: String
    val numLegs: Int
}
````

## The data access code
TODO

## The test data
TODO

## Writing unit tests
TODO

## Setting up smoke tests
TODO

Conclusion
----------

After much thrashing about with poor solutions in the early days, we have settled on this aproach and it has brought us peace and harmony.  What's your experience with managing test data?


