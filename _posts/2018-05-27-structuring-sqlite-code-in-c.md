---
layout: post
title: Structuring SQLite Code in C
subtitle: '2018-05-27'
date: 2018-05-27 05:00:00 +0000
thumb_img_path: ''
content_img_path: ''
excerpt: In this post, I describe an approach to structuring C SQLite access code
  that I have found a very useful to producing code that is easy to understand, maintain,
  and test.

---

As one of the most widely deployed software libraries in the world, SQLite is used in many contexts from small embedded systems to smartphones to web browsers. As a self-contained C library, SQLite is particularly useful as a persistent data management solution for embedded C applications. However, working with SQLite in C requires quite a lot of boilerplate SQL access code. Managing data sets of any complexity with SQLite in C can very quickly lead to a mess of spaghetti code. In this post, I describe an approach to structuring C SQLite access code that I have found a very useful to producing code that is easy to understand, maintain, and test.

## Concerns in SQLite Access Code

Any C SQLite access code must manage a variety of concerns, including:

- Raw SQL access code through the SQLite C API to prepare, execute, and read queries
- Business logic for validating and maintaining data consistency
- Concurrency management if the database is accessible from multiple threads
- Transaction management
- State management since the SQLite database connection handle represents an instance of global state

The initial approach to writing SQLite management code may often be to mix all of these concerns together. While this may seem simple and straightforward at first, it quickly leads to spaghetti code that becomes complex and difficult to test as the complexity of the data model grows.

Consider a very simple (and contrived) example application where we wish to keep track of running race results. A race may have many runners and a runner may participate in many races. We can represent races and runners in their own respective tables and represent a runner's result in a race as a row in a separate join table. We might organize our tables to look like this

<p align="center">
    <img src="/assets/img/SqliteInC/RacesDataModel.svg" />
</p>

Suppose we need to implement a function to add a new race result. This function will receive a reference to a race and a runner as well as the time it took the runner to complete the race. For both the runner and the race, our function must create a new record in the runners and races tables, respectively, and use the newly inserted row ids in the record for the race result if the runner or the race do not already exist. Otherwise, the function can just use the existing row ids for the runner and race records.

If we are keeping all of our code in a single source file, then we might implement our function like this:

```c
/**
    Global database handle
*/
static sqlite3 * g_db;
static mutex_t g_db_mutex;

// ...

/**
    Adds a new race result into the database
*/
int race_result_add_new
    (
    race_t const *      race,
    runner_t const *    runner,
    uint32_t            time_ms
    )
{
    int success;
    row_id_t race_id = INVALID_ROW_ID;
    row_id_t runner_id = INVALID_ROW_ID;
    int runner_exists = 0;
    int race_exists = 0;
    sqlite3_stmt * select_runner_query = NULL;
    sqlite3_stmt * insert_runner_query = NULL;
    sqlite3_stmt * select_race_query = NULL;
    sqlite3_stmt * insert_race_query = NULL;
    sqlite3_stmt * insert_result_query = NULL;

    mutex_lock( &g_db_mutex );

    success = ( SQLITE_OK == sqlite3_exec( g_db, "BEGIN TRANSACTION;", NO_CALLBACK, NO_CALLBACK_PARAM, NO_ERROR_MESSAGE ) );

    // Get the runner's id or create a new runner if the runner does not exist.
    if( success )
    {
        runner_exists = ( SQLITE_OK == sqlite3_prepare_v2( g_db, "SELECT * FROM runners WHERE id = ?;", READ_TO_END, &select_runner_query, NO_TAIL ) ) &&
                        ( SQLITE_OK == sqlite3_bind_int64( select_runner_query, 1, runner->id ) ) &&
                        ( SQLITE_ROW == sqlite3_step( select_runner_query ) );

        if( runner_exists )
        {
            runner_id = runner->id;
        }
        else
        {
            success = ( SQLITE_OK == sqlite3_prepare_v2( g_db, "INSERT INTO runners (name) VALUES (?);", READ_TO_END, &insert_runner_query, NO_TAIL ) ) &&
                      ( SQLITE_OK == sqlite3_bind_text( insert_runner_query, 1, runner->name, READ_TO_END, SQLITE_TRANSIENT ) ) &&
                      ( SQLITE_DONE == sqlite3_step( insert_runner_query ) );

            if( success )
            {
                runner_id = sqlite3_last_insert_rowid( g_db );
            }
        }
    }

    // Same thing for getting the race id
    // ...

    // Now insert the new race result
    if( success )
    {
        success = ( SQLITE_OK == sqlite3_prepare_v2( g_db, "INSERT INTO race_results (race_id, runner_id, time_ms) VALUES (?, ?, ?);", READ_TO_END, &insert_result_query, NO_TAIL ) ) &&
                  ( SQLITE_OK == sqlite3_bind_int64( insert_result_query, 1, race_id ) ) &&
                  ( SQLITE_OK == sqlite3_bind_int64( insert_result_query, 2, runner_id ) ) &&
                  ( SQLITE_OK == sqlite3_bind_int64( insert_result_query, 3, time_ms ) ) &&
                  ( SQLITE_DONE == sqlite3_step( insert_result_query ) );
    }


    // Clean up
    sqlite3_finalize( select_runner_query );
    sqlite3_finalize( insert_runner_query );
    sqlite3_finalize( select_race_query );
    sqlite3_finalize( insert_race_query );
    sqlite3_finalize( insert_result_query );

    if( success )
    {
        sqlite3_exec( g_db, "END TRANSACTION", NO_CALLBACK, NO_CALLBACK_PARAM, NO_ERROR_MESSAGE );
    }
    else
    {
        sqlite3_exec( g_db, "ROLLBACK TRANSACTION", NO_CALLBACK, NO_CALLBACK_PARAM, NO_ERROR_MESSAGE );
    }

    mutex_unlock( &g_db_mutex );

    return success;
}
```

While this approach where all concerns are jumbled together might seem to be the easiest way to get started when working with SQLite databases in C, it has a number of downsides

- **This code is difficult to test** because it works on a single global database handle and must maintain a mutex lock when accessing the database handle. Any tests must run against this single database instance.
- **The business logic is hard to see** amongst all the boilerplate code for preparing and executing SQLite queries.
- **This code is not very re-usable** since it only works for a single logical operation on a single database instance. Some parts of this function, such as looking up a runner by name or inserting a new runner, may be useful elsewhere.

## Splitting Concerns

To make SQLite access code more understandable, maintainable, and testable, I find it very useful to split the separate concerns into separate source files. In SQLite code, there are three areas of concerns that I find useful to keep separated into distinct layers

1. **Raw SQLite access code** for preparing and executing queries and reading query results. This code involves quite a lot of boilerplate and should be very reusable. Additionally, this code is very error-prone since it involves working with raw strings and query parameter orderings that cannot be checked at compile time. Therefore, it is very important that this code is well unit tested.
2. **Business logic** for managing, validating, and ensuring consistency of the data in the SQLite database. This code should be cleanly separated from the raw SQLite access code so that code authors and readers can focus on the important business logic rather than on the low-level details of accessing the SQLite APIs. It is also very important to unit test this code.
3. **Transaction and concurrency management** contains all of the hard-to-unit-test code and should be separated from the raw SQLite access code and the business logic so that those components can be easily tested.

These three areas of concerns nicely produce three distinct layers for SQLite management code which, from lowest level to highest level, I call

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

<p align="center">
    <img src="/assets/img/SqliteInC/LayerDiagram.svg" />
</p>

Usually, I like to have one accessor per table in the database, possibly multiple controllers if the data is complex enough, and a single interface. The source dependency graph for a somewhat complex dataset with multiple controllers may look something like

<p align="center">
    <img src="/assets/img/SqliteInC/DependencyDiagram.svg" />
</p>

For a dataset as simple as the example races dataset, a single controller to contain all the business logic will likely be all that is needed. If the dataset were to grow more complex, then we could very easily refactor to multiple controllers if needed

<p align="center">
    <img src="/assets/img/SqliteInC/RacesDependencyDiagram.svg" />
</p>

### Accessor Layer

By restructuring the SQLite access code for our race results application into these three layers, we would end up with a single accessor for each of our three tables which would provide very simple, re-usable, primitive functions for reading and writing data to and from each table. For the runners table, this may include a function to find a runner by id...

```c
// File: runners_accessor.c

/**
    Outputs true if a record exists with the given id.
*/
int runner_exists_by_id
    (
    sqlite3 *   db,
    row_id_t    id,
    int *       exists_out
    )
{
    int success;
    sqlite3_stmt * query = NULL;

    *exists_out = 0;

    success = ( SQLITE_OK == sqlite3_prepare_v2( db, "SELECT * FROM runners WHERE id = ?;", READ_TO_END, &query, NO_TAIL ) ) &&
              ( SQLITE_OK == sqlite3_bind_int64( query, 1, id ) );

    if( success )
    {
        *exists_out = ( SQLITE_ROW == sqlite3_step( query ) );
    }

    sqlite3_finalize( query );

    return success;
}
```

...and inserting a new runner...

```c
// File: runners_accessor.c

/**
  Inserts a new runner into the database
*/
int runner_insert_new
    (
    sqlite3 *           db,
    runner_t const *    runner,
    row_id_t *          insert_id_out
    )
{
    int success;
    sqlite3_stmt * query = NULL;

    *insert_id_out = INVALID_ROW_ID;

    success = ( SQLITE_OK == sqlite3_prepare_v2( db, "INSERT INTO runners (name) VALUES (?);", READ_TO_END, &query, NO_TAIL ) ) &&
              ( SQLITE_OK == sqlite3_bind_text( query, 1, runner->name, READ_TO_END, SQLITE_TRANSIENT ) );

    if( success )
    {
        success = ( SQLITE_DONE == sqlite3_step( query ) );
    }

    if( success )
    {
        *insert_id_out = sqlite3_last_insert_rowid( db );
    }

    sqlite3_finalize( query );

    return success;
}
```

...among other possible functions that may be useful. These functions are all very simple and very understandable because they are focused on one thing only: reading and writing data to and from the runners table. These functions are all easily testable because they receive the database handle on which they operate as a parameter and thus refer to no global state allowing us to inject whatever database handle we want into these functions in our unit tests. Additionally, since these functions do not manage any transactions, they can be re-used and composed in higher-level functions to perform reads and writes within the context of a larger transaction. Transactions become a higher level concern not dealt with at this layer.

### Controller Layer

Once we have a suite of low-level functions for reading and writing data to and from the tables in the database in the accessor layer, we can start building the controller layer using those low-level functions. The controller layer focuses entirely on the data management business logic. Nowhere in the controller layer should we be dealing with preparing, executing, and reading queries. Instead, we form the high-level business logic by composing the lower-level accessor functions together. If we were implementing our "add new race result" function from earlier using a layered approach, we would place that in the controller layer since it represents high-level business logic that must ensure data consistency across multiple tables. In our controller layer, this function may look like

```c
// File: race_results_controller.c

/**
    Adds the runner to the database if it does not already exist.
 */
int runner_find_existing_or_add_new
    (
    sqlite3 *           db,
    runner_t const *    runner,
    row_id_t *          id_out
    )
{
    int success;
    int exists = 0;

    *id_out = INVALID_ROW_ID;

    // Accessor function
    success = runner_exists_by_id( db, runner->id, &exists );

    if( success && exists )
    {
        *id_out = runner->id;
    }
    else if( success )
    {
        // Accessor function
        success = runner_insert_new( db, runner, id_out );
    }

    return success;
}

/**
    Adds the race to the database if it does not already exist.
 */
int race_find_existing_or_add_new
     (
     sqlite3 *      db,
     race_t const * race,
     row_id_t *     id_out
     )
{
    // Same thing for the races table
    // ...
}

/**
    Adds a new race result to the database
*/
int race_result_add_new
    (
    sqlite3 *           db,
    race_t const *      race,
    runner_t const *    runner,
    uint32_t            time_ms
    )
{
    int success;
    row_id_t runner_id = INVALID_ROW_ID;
    row_id_t race_id = INVALID_ROW_ID;
    race_result_t race_result;
    row_id_t race_result_id;

    // Controller function
    success = runner_find_existing_or_add_new( db, runner, &runner_id );

    if( success )
    {
        // Controller function
        success = race_find_existing_or_add_new( db, race, &race_id );
    }

    if( success )
    {
        race_result.race_id = race_id;
        race_result.runner_id = runner_id;
        race_result.time_ms = time_ms;

        // Accessor function
        success = race_result_insert_new( db, &race_result, &race_result_id );
    }

    return success;
}
```

With this structure, the business logic is very easy to see since it is not lost among all the low-level boilerplate code manipulating SQLite queries. Like all the accessor functions, the controller functions receive the database handle as a parameter thus allowing us to inject any database we want within our unit tests. Since the controller functions do not perform any transaction management, they are also very composable. The high-level controller functions can be composed of both accessor functions and other controller functions as is done for the `race_result_insert_new` function.

### Interface Layer

The interface layer sits at the top of the layered structure. Its job is to provide the sole point of access to the database for all other components in the application. It holds onto the database handle and provides it to the controller functions which contain the actual business logic. The interface layer is also responsible for ensuring that any necessary locks are acquired and transactions are created, ended, and rolled back as necessary when executing the controller functions.

```c
// File: race_results_interface.c

/**
    Global database handle
 */
static sqlite3 * g_db = NULL;
static mutex_t g_db_mutex;


/**
    Starts a transaction on the database.
 */
static int transaction_begin
    (
    void
    )
{
    mutex_lock( &g_db_mutex );
    return ( SQLITE_OK == sqlite3_exec( g_db, "BEGIN TRANSACTION;", NO_CALLBACK, NO_CALLBACK_PARAM, NO_ERROR_MESSAGE ) );
}


/**
    Ends a transaction on the database rolling back if an error occurred.
 */
static void transaction_end
    (
    int was_successful
    )
{
    if( was_successful )
    {
        sqlite3_exec( g_db, "END TRANSACTION;", NO_CALLBACK, NO_CALLBACK_PARAM, NO_ERROR_MESSAGE );
    }
    else
    {
        sqlite3_exec( g_db, "ROLLBACK TRANSACTION;", NO_CALLBACK, NO_CALLBACK_PARAM, NO_ERROR_MESSAGE );
    }

    mutex_unlock( &g_db_mutex );
}

/**
    Adds a new race result
*/
int race_result_add
     (
     race_t const *      race,
     runner_t const *    runner,
     uint32_t            time_ms
     )
{
    int success;

    success = transaction_begin();

    if( success )
    {
        // Controller function
        success = race_result_add_new( g_db, race, runner, time_ms );
    }

    transaction_end( success );

    return success;
}
```

The functions in the interface layer are all very simple since they contain no business logic at all and instead delegate to the controller functions to perform the business logic. Everything that makes SQLite management code difficult to test--global state, transactions, and concurrency--is isolated in the interface layer. Because it is difficult to test, no unit tests are written for the interface layer which is okay because it is very simple. The things that are easy to mess up--the complex business logic and the error-prone raw SQLite query code--are isolated into the controller and accessor layers and are fully unit tested. The interface layer simply serves as the glue code that makes the interesting, fully-tested code in the lower layers available to the rest of the application.

## Conclusion

SQLite is a very useful data management solution in embedded C applications. However, care must be taken to keep the code understandable, maintainable, and testable and to prevent it from devolving into a spaghetti mess. The key is to separate the concerns into distinct layers so that they can be reasoned about and tested in isolation.