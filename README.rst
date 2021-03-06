***************
Norm: A Nim ORM
***************

.. image:: https://travis-ci.com/moigagoo/norm.svg?branch=develop
    :alt: Build Status
    :target: https://travis-ci.com/moigagoo/norm

.. image:: https://raw.githubusercontent.com/yglukhov/nimble-tag/master/nimble.png
    :alt: Nimble
    :target: https://nimble.directory/pkg/norm


**Norm** is an object-oriented, framework-agnostic ORM for Nim that supports SQLite and PostgreSQL.

-   `Repo <https://github.com/moigagoo/norm>`__
    -   `Issues <https://github.com/moigagoo/norm/issues>`__
    -   `Pull requests <https://github.com/moigagoo/norm/pulls>`__
-   `Sample app <https://github.com/moigagoo/norm-sample-webapp>`__
-   `API index <theindex.html>`__
-   `Changelog <https://github.com/moigagoo/norm/blob/develop/changelog.rst>`__


Quickstart
==========

Install Norm with `Nimble <https://github.com/nim-lang/nimble>`_:

.. code-block:: nim

    $ nimble install norm

Add Norm to your .nimble file:

.. code-block:: nim

    requires "norm"

Here's a brief intro to Norm. Save as ``hellonorm.nim`` and run with ``nim c -r hellonorm.nim``:

.. code-block:: nim

    import norm/sqlite                        # Import SQLite backend; ``norm/postgres`` for PostgreSQL.

    import unicode, options                   # Norm supports `Option` type out of the box.

    import logging                            # Import logging to inspect the generated SQL statements.
    addHandler newConsoleLogger()


    db("petshop.db", "", "", ""):             # Set DB connection credentials.
      type                                    # Describe models in a type section.
        User = object                         # Model is a Nim object.
          age: Positive                       # Nim types are automatically converted into SQL types
                                              # and back.
                                              # You can specify how types are converted using
                                              # ``parser``, ``formatter``,
                                              # ``parseIt``, and ``formatIt`` pragmas.
          name {.
            formatIt: ?capitalize(it)         # E.g., enforce ``name`` stored in DB capitalized.
          .}: string
          ssn: Option[int]                    # ``Option`` fields are allowed to be NULL.


    withDb:                                   # Start DB session.
      createTables(force=true)                # Create tables for objects.
                                              # ``force=true`` means “drop tables if they exist.”

      var bob = User(                         # Create a ``User`` instance as you normally would.
        age: 23,                              # You can use ``initUser`` if you want.
        name: "bob",                          # Note that the instance is mutable. This is necessary,
        ssn: some 456                         # because implicit ``id``attr is updated on insertion.
      )
      bob.insert()                            # Insert ``bob`` into DB.
      echo "Bob ID = ", bob.id                # ``id`` attr is added by Norm and updated on insertion.

      discard insertId User(                  # Insert an immutable model instance and return its ID.
        age: 12,
        name: "alice",
        ssn: none int
      )

    withCustomDb("mirror.db", "", "", ""):    # Override default DB credentials
      createTables(force=true)                # to connect to a different DB with the same models.

    withDb:
      let bobs = User.getMany(                # Read records from DB:
        100,                                  # - only the first 100 records
        cond="name LIKE 'Bob%' ORDER BY age"  # - matching condition
      )

      echo "Bobs = ", bobs

    withDb:
      var bob = User.getOne(1)                # Fetch record from DB and store it as ``User`` instance.
      bob.age += 10                           # Change attr value.
      bob.update()                            # Update the record in DB.

      bob.delete()                            # Delete the record.
      echo "Bob ID = ", bob.id                # ``id`` is 0 for objects not stored in DB.

    withDb:
      transaction:                            # Put multiple statements under ``transaction`` to run
        for i in 1..10:                       # them as a single DB transaction. If any operation fails,
          var user = User(                    # the entire transaction is cancelled.
            age: 20+i,
            name: "User " & $i,
            ssn: some i
          )
          insert user

    withDb:
      dropTables()                            # Drop all tables.


Reference Guide
===============

Model Declaration
-----------------

-   ``db(connection, user, password, database: string, body: untyped)``

    Declare models from a type section with object declarations.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlite.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgres.nim

-   ``dbFromTypes(connection, user, password, database: string, types: openArray[typedesc])``

    Declare models from type sections in other modules. The type sections must be wrapped in ``dbTypes``.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlitefromtypes.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgresfromtypes.nim

-   ``dbTypes``

    Make a type section usable as a model declaration in ``dbFromTypes``.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/models/user.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/models/pet.nim


Connection
----------

-   ``withDb(body: untyped)``

    Connect to the DB using credentials defined in ``db`` section. The connection is closed on block exit.

    The connection can be accessed via ``dbConn`` variable if needed.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlite.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgres.nim

-   ``withCustomDb(customConnection, customUser, customPassword, customDatabase: string, body: untyped)``

    Connect to a custom DB. The connection is closed on block exit.

    The connection can be accessed via ``dbConn`` variable if needed.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlite.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgres.nim


Setup
-----

-   ``createTables(force = false)``

    Generate and execute DB schema for all models.

    ``force=true`` prepends ``DROP TABLE IF EXISTS`` for all genereated tables.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlite.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgres.nim


Teardown
--------

-   ``dropTables(T: typedesc)``

    Drop tables for all models.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlite.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgres.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlitefromtypes.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgresfromtypes.nim



Create Records
--------------

-   ``insert(obj: var object, force = false)``

    Store a model instance into the DB as a row.

    The input object must be mutable because its ``id`` field, initially equal ``0``, is updated after the insertion to reflect the row ID returned by the DB.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlite.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgres.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlitefromtypes.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgresfromtypes.nim

-   ``insertId(obj: object, force = false)``

    Store an immutable model instance into the DB as a row, returning the new record ID.

    The object's ``id`` field is **not** updated.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlite.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgres.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlitefromtypes.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgresfromtypes.nim



Read Records
------------

-   ``getOne(T: typedesc, id: int)``

    Fetch one row by ID and store it into a new model instance.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlite.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgres.nim


-   ``getOne(obj: var object, id: int)``

    Fetch one row by ID and store it into as existing instance.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlite.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgres.nim

-   ``getOne(T: typedesc, cond: string, params: varargs[DbValue, dbValue])``

    Fetch the first row that matches the given condition. Store into a new instance.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlite.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgres.nim

-   ``getOne(obj: var object, cond: string, params: varargs[DbValue, dbValue])``

    Fetch the first row that matches the given condition. Store into an existing instance.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlite.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgres.nim

-   ``getMany(T: typedesc, limit: int, offset = 0, cond = trueCond, params: varargs[DbValue, dbValue])``

    Fetch at most ``limit`` rows from the DB that math the given condition with the given params. The result is stored into a new sequence of model instances.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlite.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgres.nim

-   ``getMany(objs: var seq[object], limit: int, offset = 0, cond = trueCond, params: varargs[DbValue, dbValue])``

    Fetch at most ``limit`` rows from the DB that math the given condition with the given params. The result is stored into an existing sequence of model instances.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlite.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgres.nim

-   ``getAll(T: typedesc, cond = trueCond, params: varargs[DbValue, dbValue])``

    Get all rows from a table that match the given condition.

    **Warning:** This is a dangerous operation because you're fetching an unknown number of rows, which could be millions. Consider using ``getMany`` instead.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlite.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgres.nim


Update Records
--------------

-   ``update(obj: object, force = false)``

    Update a record in the DB with the current field values of a model instance.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlite.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgres.nim


Delete Records
--------------

-   ``delete(obj: var object)``

    Delete a record from the DB by ID from a model instance. The instance's ``id`` fields is set to ``0``.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlite.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgres.nim


Transactions
------------

-   ``transaction(transactionBody: untyped)``

    Wrap statements in a ``transaction`` block to run them as a single DB transaction: if any statements fails, the entire transaction is cancelled.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlitemigrate.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgresmigrate.nim

-   ``rollback``

    Raise ``RollbackError`` that is catched inside a ``transaction`` block and cancels the transaction.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlitemigrate.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgresmigrate.nim


Migrations
----------

**Note:** Although Norm provides the means to write and apply migrations manually, the plan is to develop a tool to generate migrations from model diffs and apply them with the option to rollback.

-   ``createTable(T: typedesc, force = false)``

    Generate and execute an SQL table schema from a type definition. Column schemas are generated from Nim object field definitions. Basic types are mapped automatically. For custom types, *parser* and *formatter* must be provided.

    Use to update the DB schema after adding new models.

    ``force=true`` prepends `DROP TABLE IF EXISTS` to the generated query.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlitemigrate.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgresmigrate.nim

-   ``addColumn(field: typedesc)``

    Generate and execute an SQL query to add a column to an existing table.

    Use to create columns after adding new fields to existing models.

    ``field`` should point to the model field for which the column is to be created, e.g. ``Pet.age``.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlitemigrate.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgresmigrate.nim

-   ``dropColumns(T: typedesc, cols: openArray[string])``

    PostgreSQL only. Drop all columns of a table.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgresmigrate.nim

-   ``dropUnusedColumns(T: typedesc)``

    Recreate the table from a model, losing unmatching columns in the process. This involves creating a temporary table and copying the data there, then dropping the original table and renaming the temporary one to the original one's name.

    Use to clean up DB after removing a field from a model.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlitemigrate.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgresmigrate.nim

-   ``renameColumnFrom(field: typedesc, oldName: string)``.

    Rename a DB column to match the model field. Provide ``oldName`` to tell Norm which column you are renaming. This has to be done manually since there's no way to guess the programmer's intetion when they rename a model field: is it to rename the underlying DB column or to remove the old column and create a new one instead?

    Use this proc to rename a column. To replace a column, use `addColumn` with conjunction with ``dropUnusedColumns``.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlitemigrate.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlitemigrate.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgresmigrate.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgresmigrate.nim

-   ``renameTableFrom(T: typedesc, oldName: string)``

    Rename a DB table to match the model name. The old table name must be provided explicitly because when the DB table name for a model changes, there's no way to guess which existing table used to match this model.

    Use after renaming a model or changing its ``dbTable`` pragma value.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlitemigrate.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgresmigrate.nim


-   ``dropTable(T: typedesc)``

    Drop table associated with a model.

    Use after removing a model.

    Tests:

    -   https://github.com/moigagoo/norm/blob/develop/tests/tsqlite.nim
    -   https://github.com/moigagoo/norm/blob/develop/tests/tpostgres.nim


Contributing
============

Any contributions are welcome: pull requests, code reviews, documentation improvements, bug reports, and feature requests.

-   See the [issues on GitHub](http://github.com/moigagoo/norm/issues).

-   Run the tests before and after you change the code.

    The recommended way to run the tests is via [Docker](https://www.docker.com/) and [Docker Compose](https://docs.docker.com/compose/):

    .. code-block::

        $ docker-compose run --rm tests                     # run all test suites
        $ docker-compose run --rm test tests/tpostgres.nim  # run a single test suite

    If you don't mind running two PostgreSQL servers on `postgres_1` and `postgres_2`, feel free to run the test suites natively:

    .. code-block::

        $ nimble test

    Note that you only need the PostgreSQL servers to run the PostgreSQL backend tests, so:

    .. code-block::

        $ nim c -r tests/tsqlite.nim    # doesn't require PostgreSQL servers, but requires SQLite
        $ nim c -r tests/tobjutils.nim  # doesn't require anything at all

-   Use camelCase instead of snake_case.

-   New procs must have a documentation comment. If you modify an existing proc, update the comment.

-   Apart from the code that implements a feature or fixes a bug, PRs are required to ship necessary tests and a changelog updates.


❤ Contributors ❤
------------------

Norm would not be where it is today without the efforts of these fine folks: `https://github.com/moigagoo/norm/graphs/contributors <https://github.com/moigagoo/norm/graphs/contributors>`_
