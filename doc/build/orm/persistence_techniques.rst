=================================
Additional Persistence Techniques
=================================

.. _flush_embedded_sql_expressions:

Embedding SQL Insert/Update Expressions into a Flush
====================================================

This feature allows the value of a database column to be set to a SQL
expression instead of a literal value. It's especially useful for atomic
updates, calling stored procedures, etc. All you do is assign an expression to
an attribute::

    class SomeClass(object):
        pass
    mapper(SomeClass, some_table)

    someobject = session.query(SomeClass).get(5)

    # set 'value' attribute to a SQL expression adding one
    someobject.value = some_table.c.value + 1

    # issues "UPDATE some_table SET value=value+1"
    session.commit()

This technique works both for INSERT and UPDATE statements. After the
flush/commit operation, the ``value`` attribute on ``someobject`` above is
expired, so that when next accessed the newly generated value will be loaded
from the database.

.. _session_sql_expressions:

Using SQL Expressions with Sessions
===================================

SQL expressions and strings can be executed via the
:class:`~sqlalchemy.orm.session.Session` within its transactional context.
This is most easily accomplished using the
:meth:`~.Session.execute` method, which returns a
:class:`~sqlalchemy.engine.ResultProxy` in the same manner as an
:class:`~sqlalchemy.engine.Engine` or
:class:`~sqlalchemy.engine.Connection`::

    Session = sessionmaker(bind=engine)
    session = Session()

    # execute a string statement
    result = session.execute("select * from table where id=:id", {'id':7})

    # execute a SQL expression construct
    result = session.execute(select([mytable]).where(mytable.c.id==7))

The current :class:`~sqlalchemy.engine.Connection` held by the
:class:`~sqlalchemy.orm.session.Session` is accessible using the
:meth:`~.Session.connection` method::

    connection = session.connection()

The examples above deal with a :class:`~sqlalchemy.orm.session.Session` that's
bound to a single :class:`~sqlalchemy.engine.Engine` or
:class:`~sqlalchemy.engine.Connection`. To execute statements using a
:class:`~sqlalchemy.orm.session.Session` which is bound either to multiple
engines, or none at all (i.e. relies upon bound metadata), both
:meth:`~.Session.execute` and
:meth:`~.Session.connection` accept a ``mapper`` keyword
argument, which is passed a mapped class or
:class:`~sqlalchemy.orm.mapper.Mapper` instance, which is used to locate the
proper context for the desired engine::

    Session = sessionmaker()
    session = Session()

    # need to specify mapper or class when executing
    result = session.execute("select * from table where id=:id", {'id':7}, mapper=MyMappedClass)

    result = session.execute(select([mytable], mytable.c.id==7), mapper=MyMappedClass)

    connection = session.connection(MyMappedClass)

.. _session_forcing_null:

Forcing NULL on a column with a default
=======================================

The ORM considers any attribute that was never set on an object as a
"default" case; the attribute will be omitted from the INSERT statement::

    class MyObject(Base):
        __tablename__ = 'my_table'
        id = Column(Integer, primary_key=True)
        data = Column(String(50), nullable=True)

    obj = MyObject(id=1)
    session.add(obj)
    session.commit()  # INSERT with the 'data' column omitted; the database
                      # itself will persist this as the NULL value

Omitting a column from the INSERT means that the column will
have the NULL value set, *unless* the column has a default set up,
in which case the default value will be persisted.   This holds true
both from a pure SQL perspective with server-side defaults, as well as the
behavior of SQLAlchemy's insert behavior with both client-side and server-side
defaults::

    class MyObject(Base):
        __tablename__ = 'my_table'
        id = Column(Integer, primary_key=True)
        data = Column(String(50), nullable=True, server_default="default")

    obj = MyObject(id=1)
    session.add(obj)
    session.commit()  # INSERT with the 'data' column omitted; the database
                      # itself will persist this as the value 'default'

However, in the ORM, even if one assigns the Python value ``None`` explicitly
to the object, this is treated the **same** as though the value were never
assigned::

    class MyObject(Base):
        __tablename__ = 'my_table'
        id = Column(Integer, primary_key=True)
        data = Column(String(50), nullable=True, server_default="default")

    obj = MyObject(id=1, data=None)
    session.add(obj)
    session.commit()  # INSERT with the 'data' column explicitly set to None;
                      # the ORM still omits it from the statement and the
                      # database will still persist this as the value 'default'

The above operation will persist into the ``data`` column the
server default value of ``"default"`` and not SQL NULL, even though ``None``
was passed; this is a long-standing behavior of the ORM that many applications
hold as an assumption.

So what if we want to actually put NULL into this column, even though the
column has a default value?  There are two approaches.  One is that
on a per-instance level, we assign the attribute using the
:obj:`~.expression.null` SQL construct::

    from sqlalchemy import null

    obj = MyObject(id=1, data=null())
    session.add(obj)
    session.commit()  # INSERT with the 'data' column explicitly set as null();
                      # the ORM uses this directly, bypassing all client-
                      # and server-side defaults, and the database will
                      # persist this as the NULL value

The :obj:`~.expression.null` SQL construct always translates into the SQL
NULL value being directly present in the target INSERT statement.

If we'd like to be able to use the Python value ``None`` and have this
also be persisted as NULL despite the presence of column defaults,
we can configure this for the ORM using a Core-level modifier
:meth:`.TypeEngine.evaluates_none`, which indicates
a type where the ORM should treat the value ``None`` the same as any other
value and pass it through, rather than omitting it as a "missing" value::

    class MyObject(Base):
        __tablename__ = 'my_table'
        id = Column(Integer, primary_key=True)
        data = Column(
          String(50).evaluates_none(),  # indicate that None should always be passed
          nullable=True, server_default="default")

    obj = MyObject(id=1, data=None)
    session.add(obj)
    session.commit()  # INSERT with the 'data' column explicitly set to None;
                      # the ORM uses this directly, bypassing all client-
                      # and server-side defaults, and the database will
                      # persist this as the NULL value

.. topic:: Evaluating None

  The :meth:`.TypeEngine.evaluates_none` modifier is primarily intended to
  signal a type where the Python value "None" is significant, the primary
  example being a JSON type which may want to persist the JSON ``null`` value
  rather than SQL NULL.  We are slightly repurposing it here in order to
  signal to the ORM that we'd like ``None`` to be passed into the type whenever
  present, even though no special type-level behaviors are assigned to it.

.. versionadded:: 1.1 added the :meth:`.TypeEngine.evaluates_none` method
   in order to indicate that a "None" value should be treated as significant.

.. _orm_server_defaults:

Fetching Server-Generated Defaults
===================================

As introduced in the sections :ref:`server_defaults` and :ref:`triggered_columns`,
the Core supports the notion of database columns for which the database
itself generates a value upon INSERT and in less common cases upon UPDATE
statements.  The ORM features support for such columns regarding being
able to fetch these newly generated values upon flush.   This behavior is
required in the case of primary key columns that are generated by the server,
since the ORM has to know the primary key of an object once it is persisted.

In the vast majority of cases, primary key columns that have their value
generated automatically by the database are  simple integer columns, which are
implemented by the database as either a so-called "autoincrement" column, or
from a sequence associated with the column.   Every database dialect within
SQLAlchemy Core supports a method of retrieving these primary key values which
is often native to the Python DBAPI, and in general this process is automatic,
with the exception of a database like Oracle that requires us to specify a
:class:`.Sequence` explicitly.   There is more documentation regarding this
at :paramref:`.Column.autoincrement`.

For server-generating columns that are not primary key columns or that are not
simple autoincrementing integer columns, the ORM requires that these columns
are marked with an appropriate server_default directive that allows the ORM to
retrieve this value.   Not all methods are supported on all backends, however,
so care must be taken to use the appropriate method. The two questions to be
answered are, 1. is this column part of the primary key or not, and 2. does the
database support RETURNING or an equivalent, such as "OUTPUT inserted"; these
are SQL phrases which return a server-generated value at the same time as the
INSERT or UPDATE statement is invoked. Databases that support RETURNING or
equivalent include PostgreSQL, Oracle, and SQL Server.  Databases that do not
include SQLite and MySQL.

Case 1: non primary key, RETURNING or equivalent is supported
-------------------------------------------------------------

In this case, columns should be marked as :class:`.FetchedValue` or with an
explicit :paramref:`.Column.server_default`.   The
:paramref:`.orm.mapper.eager_defaults` flag may be used to indicate that these
columns should be fetched immediately upon INSERT and sometimes UPDATE::


    class MyModel(Base):
        __tablename__ = 'my_table'

        id = Column(Integer, primary_key=True)
        timestamp = Column(DateTime(), server_default=func.now())

        # assume a database trigger populates a value into this column
        # during INSERT
        special_identifier = Column(String(50), server_default=FetchedValue())

        __mapper_args__ = {"eager_defaults": True}

Above, an INSERT statement that does not specify explicit values for
"timestamp" or "special_identifier" from the client side will include the
"timestamp" and "special_identifier" columns within the RETURNING clause so
they are available immediately. On the PostgreSQL database, an INSERT for the
above table will look like:

.. sourcecode:: sql

   INSERT INTO my_table DEFAULT VALUES RETURNING my_table.id, my_table.timestamp, my_table.special_identifier

Case 2: non primary key, RETURNING or equivalent is not supported or not needed
--------------------------------------------------------------------------------

This case is the same as case 1 above, except we don't specify
:paramref:`.orm.mapper.eager_defaults`::

    class MyModel(Base):
        __tablename__ = 'my_table'

        id = Column(Integer, primary_key=True)
        timestamp = Column(DateTime(), server_default=func.now())

        # assume a database trigger populates a value into this column
        # during INSERT
        special_identifier = Column(String(50), server_default=FetchedValue())

After a record with the above mapping is INSERTed, the "timestamp" and
"special_identifier" columns will remain empty, and will be fetched via
a second SELECT statement when they are first accessed after the flush, e.g.
they are marked as "expired".

If the :paramref:`.orm.mapper.eager_defaults` is still used, and the backend
database does not support RETURNING or an equivalent, the ORM will emit this
SELECT statement immediately following the INSERT statement.   This is often
undesirable as it adds additional SELECT statements to the flush process that
may not be needed.  Using the above mapping with the
:paramref:`.orm.mapper.eager_defaults` flag set to True against MySQL results
in SQL like this upon flush (minus the comment, which is for clarification only):

.. sourcecode:: sql

    INSERT INTO my_table () VALUES ()

    -- when eager_defaults **is** used, but RETURNING is not supported
    SELECT my_table.timestamp AS my_table_timestamp, my_table.special_identifier AS my_table_special_identifier
    FROM my_table WHERE my_table.id = %s

Case 3: primary key, RETURNING or equivalent is supported
----------------------------------------------------------

A primary key column with a server-generated value must be fetched immediately
upon INSERT; the ORM can only access rows for which it has a primary key value,
so if the primary key is generated by the server, the ORM needs a way for the
database to give us that new value immediately upon INSERT.

As mentioned above, for integer "autoincrement" columns as well as
PostgreSQL SERIAL, these types are handled automatically by the Core; databases
include functions for fetching the "last inserted id" where RETURNING
is not supported, and where RETURNING is supported SQLAlchemy will use that.

However, for non-integer values, as well as for integer values that must be
explicitly linked to a sequence or other triggered routine,  the server default
generation must be marked in the table metadata.

For an explicit sequence as we use with Oracle, this just means we are using
the :class:`.Sequence` construct::

    class MyOracleModel(Base):
        __tablename__ = 'my_table'

        id = Column(Integer, Sequence("my_sequence"), primary_key=True)
        data = Column(String(50))

The INSERT for a model as above on Oracle looks like:

.. sourcecode:: sql

    INSERT INTO my_table (id, data) VALUES (my_sequence.nextval, :data) RETURNING my_table.id INTO :ret_0

Where above, SQLAlchemy renders ``my_sequence.nextval`` for the primary key column
and also uses RETURNING to get the new value back immediately.

For datatypes that generate values automatically, or columns that are populated
by a trigger, we use :class:`.FetchedValue`.  Below is a model that uses a
SQL Server TIMESTAMP column as the primary key, which generates values automatically::

    class MyModel(Base):
        __tablename__ = 'my_table'

        timestamp = Column(TIMESTAMP(), server_default=FetchedValue(), primary_key=True)

An INSERT for the above table on SQL Server looks like:

.. sourcecode:: sql

    INSERT INTO my_table OUTPUT inserted.timestamp DEFAULT VALUES

Case 4: primary key, RETURNING or equivalent is not supported
--------------------------------------------------------------

In this area we are generating rows for a database such as SQLite or
more typically MySQL where some means of generating a default is occurring
on the server, but is outside of the database's usual autoincrement routine.
In this case, we have to make sure SQLAlchemy can "pre-execute" the default,
which means it has to be an explicit SQL expression.   Again using the example
of TIMESTAMP for MySQL, we unfortunately need to use our own explicit default::

    class MyModel(Base):
        __tablename__ = 'my_table'

        timestamp = Column(TIMESTAMP(), default=func.cast(func.now(), Binary), primary_key=True)

Where above, we select the "NOW()" function and also cast to binary to
be used with MySQL's TIMESTAMP column, that is in fact a binary datatype.
The SQL generated by the above is:

.. sourcecode:: sql

    SELECT CAST(now() AS BINARY) AS anon_1
    INSERT INTO my_table (timestamp) VALUES (%s)
    (b'2018-08-09 13:08:46',)

The Core currently does not support a means of fetching the timestamp value
after the fact without using RETURNING, so on MySQL must run a SELECT ahead of
time to pre-select the value.

.. seealso::

    :ref:`metadata_defaults_toplevel`


.. _session_partitioning:

Partitioning Strategies
=======================

Simple Vertical Partitioning
----------------------------

Vertical partitioning places different kinds of objects, or different tables,
across multiple databases::

    engine1 = create_engine('postgresql://db1')
    engine2 = create_engine('postgresql://db2')

    Session = sessionmaker(twophase=True)

    # bind User operations to engine 1, Account operations to engine 2
    Session.configure(binds={User:engine1, Account:engine2})

    session = Session()

Above, operations against either class will make usage of the :class:`.Engine`
linked to that class.   Upon a flush operation, similar rules take place
to ensure each class is written to the right database.

The transactions among the multiple databases can optionally be coordinated
via two phase commit, if the underlying backend supports it.  See
:ref:`session_twophase` for an example.

Custom Vertical Partitioning
----------------------------

More comprehensive rule-based class-level partitioning can be built by
overriding the :meth:`.Session.get_bind` method.   Below we illustrate
a custom :class:`.Session` which delivers the following rules:

1. Flush operations are delivered to the engine named ``master``.

2. Operations on objects that subclass ``MyOtherClass`` all
   occur on the ``other`` engine.

3. Read operations for all other classes occur on a random
   choice of the ``slave1`` or ``slave2`` database.

::

    engines = {
        'master':create_engine("sqlite:///master.db"),
        'other':create_engine("sqlite:///other.db"),
        'slave1':create_engine("sqlite:///slave1.db"),
        'slave2':create_engine("sqlite:///slave2.db"),
    }

    from sqlalchemy.orm import Session, sessionmaker
    import random

    class RoutingSession(Session):
        def get_bind(self, mapper=None, clause=None):
            if mapper and issubclass(mapper.class_, MyOtherClass):
                return engines['other']
            elif self._flushing:
                return engines['master']
            else:
                return engines[
                    random.choice(['slave1','slave2'])
                ]

The above :class:`.Session` class is plugged in using the ``class_``
argument to :class:`.sessionmaker`::

    Session = sessionmaker(class_=RoutingSession)

This approach can be combined with multiple :class:`.MetaData` objects,
using an approach such as that of using the declarative ``__abstract__``
keyword, described at :ref:`declarative_abstract`.

Horizontal Partitioning
-----------------------

Horizontal partitioning partitions the rows of a single table (or a set of
tables) across multiple databases.

See the "sharding" example: :ref:`examples_sharding`.

.. _bulk_operations:

Bulk Operations
===============

.. note::  Bulk Operations mode is a new series of operations made available
   on the :class:`.Session` object for the purpose of invoking INSERT and
   UPDATE statements with greatly reduced Python overhead, at the expense
   of much less functionality, automation, and error checking.
   As of SQLAlchemy 1.0, these features should be considered as "beta", and
   additionally are intended for advanced users.

.. versionadded:: 1.0.0

Bulk operations on the :class:`.Session` include :meth:`.Session.bulk_save_objects`,
:meth:`.Session.bulk_insert_mappings`, and :meth:`.Session.bulk_update_mappings`.
The purpose of these methods is to directly expose internal elements of the unit of work system,
such that facilities for emitting INSERT and UPDATE statements given dictionaries
or object states can be utilized alone, bypassing the normal unit of work
mechanics of state, relationship and attribute management.   The advantages
to this approach is strictly one of reduced Python overhead:

* The flush() process, including the survey of all objects, their state,
  their cascade status, the status of all objects associated with them
  via :func:`.relationship`, and the topological sort of all operations to
  be performed is completely bypassed.  This reduces a great amount of
  Python overhead.

* The objects as given have no defined relationship to the target
  :class:`.Session`, even when the operation is complete, meaning there's no
  overhead in attaching them or managing their state in terms of the identity
  map or session.

* The :meth:`.Session.bulk_insert_mappings` and :meth:`.Session.bulk_update_mappings`
  methods accept lists of plain Python dictionaries, not objects; this further
  reduces a large amount of overhead associated with instantiating mapped
  objects and assigning state to them, which normally is also subject to
  expensive tracking of history on a per-attribute basis.

* The set of objects passed to all bulk methods are processed
  in the order they are received.   In the case of
  :meth:`.Session.bulk_save_objects`, when objects of different types are passed,
  the INSERT and UPDATE statements are necessarily broken up into per-type
  groups.  In order to reduce the number of batch INSERT or UPDATE statements
  passed to the DBAPI, ensure that the incoming list of objects
  are grouped by type.

* The process of fetching primary keys after an INSERT also is disabled by
  default.   When performed correctly, INSERT statements can now more readily
  be batched by the unit of work process into ``executemany()`` blocks, which
  perform vastly better than individual statement invocations.

* UPDATE statements can similarly be tailored such that all attributes
  are subject to the SET clase unconditionally, again making it much more
  likely that ``executemany()`` blocks can be used.

The performance behavior of the bulk routines should be studied using the
:ref:`examples_performance` example suite.  This is a series of example
scripts which illustrate Python call-counts across a variety of scenarios,
including bulk insert and update scenarios.

.. seealso::

  :ref:`examples_performance` - includes detailed examples of bulk operations
  contrasted against traditional Core and ORM methods, including performance
  metrics.

Usage
-----

The methods each work in the context of the :class:`.Session` object's
transaction, like any other::

    s = Session()
    objects = [
        User(name="u1"),
        User(name="u2"),
        User(name="u3")
    ]
    s.bulk_save_objects(objects)

For :meth:`.Session.bulk_insert_mappings`, and :meth:`.Session.bulk_update_mappings`,
dictionaries are passed::

    s.bulk_insert_mappings(User,
      [dict(name="u1"), dict(name="u2"), dict(name="u3")]
    )

.. seealso::

    :meth:`.Session.bulk_save_objects`

    :meth:`.Session.bulk_insert_mappings`

    :meth:`.Session.bulk_update_mappings`


Comparison to Core Insert / Update Constructs
---------------------------------------------

The bulk methods offer performance that under particular circumstances
can be close to that of using the core :class:`.Insert` and
:class:`.Update` constructs in an "executemany" context (for a description
of "executemany", see :ref:`execute_multiple` in the Core tutorial).
In order to achieve this, the
:paramref:`.Session.bulk_insert_mappings.return_defaults`
flag should be disabled so that rows can be batched together.   The example
suite in :ref:`examples_performance` should be carefully studied in order
to gain familiarity with how fast bulk performance can be achieved.

ORM Compatibility
-----------------

The bulk insert / update methods lose a significant amount of functionality
versus traditional ORM use.   The following is a listing of features that
are **not available** when using these methods:

* persistence along :func:`.relationship` linkages

* sorting of rows within order of dependency; rows are inserted or updated
  directly in the order in which they are passed to the methods

* Session-management on the given objects, including attachment to the
  session, identity map management.

* Functionality related to primary key mutation, ON UPDATE cascade

* SQL expression inserts / updates (e.g. :ref:`flush_embedded_sql_expressions`)

* ORM events such as :meth:`.MapperEvents.before_insert`, etc.  The bulk
  session methods have no event support.

Features that **are available** include:

* INSERTs and UPDATEs of mapped objects

* Version identifier support

* Multi-table mappings, such as joined-inheritance - however, an object
  to be inserted across multiple tables either needs to have primary key
  identifiers fully populated ahead of time, else the
  :paramref:`.Session.bulk_save_objects.return_defaults` flag must be used,
  which will greatly reduce the performance benefits


