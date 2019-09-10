---
layout: post
title: Structuring SQLite Code in C
subtitle: '2018-05-27'
date: 2018-05-27 05:00:00 +0000
thumb_img_path: ''
content_img_path: ''
excerpt: Structuring C SQLite access code to be easy to understand, maintain, and test.

---

As one of the most widely deployed software libraries in the world, SQLite is used in many contexts from small embedded systems to smartphones to web browsers. As a self-contained C library, SQLite is particularly useful as a persistent data management solution for embedded C applications. However, working with SQLite in C requires quite a lot of boilerplate SQL access code. Managing data sets of any complexity with SQLite in C can very quickly lead to a mess of spaghetti code. In this post, I describe an approach to structuring C SQLite access code that I have found a very useful for producing code that is easy to understand, maintain, and test.

## Concerns in SQLite Access Code

Any C SQLite access code must manage a variety of concerns, including:

- Raw SQL access code through the SQLite C API to prepare, execute, and read queries
- Business logic for validating and maintaining data consistency
- Concurrency management if the database is accessible from multiple threads
- Transaction management
- State management since the SQLite database connection handle represents an instance of global state

The initial approach to writing SQLite management code may often be to mix all of these concerns together. While this may seem simple and straightforward at first, it quickly leads to spaghetti code that becomes complex and difficult to test as the complexity of the data model grows.

Consider a very simple (and contrived) example application where we wish to keep track of running race results. A race may have many runners and a runner may participate in many races. We can represent races and runners in their own respective tables and represent a runner's result in a race as a row in a separate join table. We might organize our tables to look like this

![](/images/structuring-sqlite-code-in-c/RacesDataModel.svg)

Suppose we need to implement a function to add a new race result. This function will receive a reference to a race and a runner as well as the time it took the runner to complete the race. For both the runner and the race, our function must create a new record in the runners and races tables, respectively, and use the newly inserted row ids in the record for the race result if the runner or the race do not already exist. Otherwise, the function can just use the existing row ids for the runner and race records.

If we are keeping all of our code in a single source file, then we might implement our function like this:

{% gist 4ddd138b5764a9e575049be649ee1042 %}

While this approach where all concerns are jumbled together might seem to be the easiest way to get started when working with SQLite databases in C, it has a number of downsides

- **This code is difficult to test** because it works on a single global database handle and must maintain a mutex lock when accessing the database handle. Any tests must run against this single database instance.
- **The business logic is hard to see** amongst all the boilerplate code for preparing and executing SQLite queries.
- **This code is not very re-usable** since it only works for a single logical operation on a single database instance. Some parts of this function, such as looking up a runner by name or inserting a new runner, may be useful elsewhere.

## Splitting Concerns

To make SQLite access code more understandable, maintainable, and testable, I find it very useful to split the separate concerns into separate source files. In SQLite code, there are three areas of concerns that I find useful to keep separated into distinct layers

1. **Raw SQLite access code** for preparing and executing queries and reading query results. This code involves quite a lot of boilerplate and should be very reusable. Additionally, this code is very error-prone since it involves working with raw strings and query parameter orderings that cannot be checked at compile time. Therefore, it is very important that this code is well unit tested.
2. **Business logic** for managing, validating, and ensuring consistency of the data in the SQLite database. This code should be cleanly separated from the raw SQLite access code so that code authors and readers can focus on the important business logic rather than on the low-level details of accessing the SQLite APIs. It is also very important to unit test this code.
3. **Transaction and concurrency management** contains all of the hard-to-unit-test code and should be separated from the raw SQLite access code and the business logic so that those components can be easily tested.

These three areas of concerns nicely produce three distinct layers for SQLite management code which, from the lowest level to the highest level, I call

1. **Accessor layer**
   - Contains all raw SQLite access code.
   - Lowest layer.
   - Contains NO business logic.
   - All functions in this layer receive a database handle as a parameter so that they can be tested using a test-specific, in-memory database.
   - Fully unit-testable.
   - Usually one accessor per table in the database.
2. **Controller layer**
   - Contains all business logic.
   - Middle layer.
   - Contains NO raw SQLite access code.
   - Contains NO transaction or concurrency management code.
   - All functions in this layer receive a database handle as a parameter so that they can be tested using a test-specific, in-memory database.
   - Fully unit-testable.
   - May be multiple controllers depending on the complexity of the data.
3. **Interface layer**
   - Contains all transaction and concurrency management code.
   - Highest layer.
   - Contains NO business logic.
   - Contains NO raw SQLite access code.
   - Holds a reference to the database handle. Provides the database handle to the controller layer.
   - Sole access point to the database for all other components in the application.
   - Typically only one interface for the database.

![](/images/structuring-sqlite-code-in-c/LayerDiagram.svg)

Usually, I like to have one accessor per table in the database, possibly multiple controllers if the data is complex enough, and a single interface. The source dependency graph for a somewhat complex dataset with multiple controllers may look something like

![](/images/structuring-sqlite-code-in-c/DependencyDiagram.svg)

For a dataset as simple as the example races dataset, a single controller to contain all the business logic will likely be all that is needed. If the dataset were to grow more complex, then we could very easily refactor to multiple controllers if needed

![](/images/structuring-sqlite-code-in-c/RacesDependencyDiagram.svg)

### Accessor Layer

By restructuring the SQLite access code for our race results application into these three layers, we would end up with a single accessor for each of our three tables which would provide very simple, re-usable, primitive functions for reading and writing data to and from each table. For the runners table, this may include a function to find a runner by id...

{% gist dc84d5312009938d303abcdc1f3d64a1 %}

...and inserting a new runner...

{% gist ab0c677ccbdbe8c4c7c4edde5174e474 %}

...among other possible functions that may be useful. These functions are all very simple and very understandable because they are focused on one thing only: reading and writing data to and from the runners table. These functions are all easily testable because they receive the database handle on which they operate as a parameter and thus refer to no global state allowing us to inject whatever database handle we want into these functions in our unit tests. Additionally, since these functions do not manage any transactions, they can be re-used and composed in higher-level functions to perform reads and writes within the context of a larger transaction. Transactions become a higher level concern not dealt with at this layer.

### Controller Layer

Once we have a suite of low-level functions for reading and writing data to and from the tables in the database in the accessor layer, we can start building the controller layer using those low-level functions. The controller layer focuses entirely on the data management business logic. Nowhere in the controller layer should we be dealing with preparing, executing, and reading queries. Instead, we form the high-level business logic by composing the lower-level accessor functions together. If we were implementing our "add new race result" function from earlier using a layered approach, we would place that in the controller layer since it represents high-level business logic that must ensure data consistency across multiple tables. In our controller layer, this function may look like

{% gist 2e4c37c55cc8da6cd64e0764ae5b23be %}

With this structure, the business logic is very easy to see since it is not lost among all the low-level boilerplate code manipulating SQLite queries. Like all the accessor functions, the controller functions receive the database handle as a parameter thus allowing us to inject any database we want within our unit tests. Since the controller functions do not perform any transaction management, they are also very composable. The high-level controller functions can be composed of both accessor functions and other controller functions as is done for the `race_result_insert_new` function.

### Interface Layer

The interface layer sits at the top of the layered structure. Its job is to provide the sole point of access to the database for all other components in the application. It holds onto the database handle and provides it to the controller functions which contain the actual business logic. The interface layer is also responsible for ensuring that any necessary locks are acquired and transactions are created, ended, and rolled back as necessary when executing the controller functions.

{% gist f62bab8f90a964979b1d9e0da554d4a5 %}

The functions in the interface layer are all very simple since they contain no business logic at all and instead delegate to the controller functions to perform the business logic. Everything that makes SQLite management code difficult to test--global state, transactions, and concurrency--is isolated in the interface layer. Because it is difficult to test, no unit tests are written for the interface layer which is okay because it is very simple. The things that are easy to mess up--the complex business logic and the error-prone raw SQLite query code--are isolated into the controller and accessor layers and are fully unit tested. The interface layer simply serves as the glue code that makes the interesting, fully-tested code in the lower layers available to the rest of the application.

## Conclusion

SQLite is a very useful data management solution in embedded C applications. However, care must be taken to keep the code understandable, maintainable, and testable and to prevent it from devolving into a spaghetti mess. The key is to separate the concerns into distinct layers so that they can be reasoned about and tested in isolation.