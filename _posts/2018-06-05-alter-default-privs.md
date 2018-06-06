---
layout: post
title: "Altering Default Privileges For Fun and Profit"
subtitle: "What NOT to expect when you're expecting."
tags: postgresql security
date: '2018-06-05'
---

One of the big changes I came upon as I transitioned from Oracle to PostgreSQL was how privileges are handled. In Oracle, an object's schema also determines its owner, as the schema is generally that user's objects. PostgreSQL disassociates schemas from users. Schemas do have owners, but a given user (or role) could own many schemas. Furthermore, objects such as tables in those schemas could be owned by completely different users or roles. Once I [unlearned what I had learned](https://www.youtube.com/watch?v=z4jeREy7Pbc) and got my head around that, I next came upon the concept of "default privileges".

Let's say we have a database, privtest, with a schema `app_schema` owned by `app_owner`. We also have 3 users: bigsam, rhino and marco. All 3 users have `USAGE` granted on privtest, while bigsam and rhino also have `CREATE`:

```
> grant usage on schema app_schema to bigsam;
GRANT
> grant usage on schema app_schema to rhino;
GRANT
> grant usage on schema app_schema to marco;
GRANT
> grant create on schema app_schema to bigsam;
GRANT
> grant create on schema app_schema to rhino;
GRANT
```

## Setting Default Privileges
So we first connect as bigsam and define default privileges for tables he creates in the `app_schema` schema.

```
> \connect privtest bigsam
You are now connected to database "privtest" as user "bigsam".

> alter default privileges in schema app_schema
        grant select on tables to marco;
ALTER DEFAULT PRIVILEGES
```

Then let's go ahead and create a new table:

```
> create table app_schema.sam_tables as select * from pg_tables;
SELECT 62
```

Default privileges allow you to say what privileges should be granted on the specified object types (tables, sequences, functions, etc.) that **you** create in the future. Notice that I emphasized "**you**". At first I thought it applied to all objects in the schema, but it only applies to objects in the schema created by the user executing the `ALTER DEFAULT PRIVILEGE` command. So if I only grant default privileges as `bigsam` but then create the objects as `app_owner`, then marco still can't see that table.

Also it's important to re-state that this only applies to *future* objects. To change privileges on already-existing objects, use the standard `GRANT` syntax.

So we see in the above example that bigsam has said that all tables that he creates in the `app_schema` schema in the future should be readable by marco. So let's take a look at the table privileges in the schema at this point with the `\dp` command:

```
> \dp app_schema.*
                                   Access privileges
   Schema   |    Name    | Type  |   Access privileges   | Column privileges | Policies
------------+------------+-------+-----------------------+-------------------+----------
 app_schema | sam_tables | table | bigsam=arwdDxt/bigsam+|                   |
            |            |       | marco=r/bigsam        |                   |
(1 row)
```

We see we have just the one table, owned by bigsam with full privileges. Then we also see that marco has the `SELECT` privilege (`marco=r`), and that it was granted by bigsam. The `\dp` access privilege codes can be found in the [`GRANT` documentation](https://www.postgresql.org/docs/current/static/sql-grant.html).

So now as marco we can see the data in that table:

```
> \connect privtest marco
You are now connected to database "privtest" as user "marco".

> select count(*) from app_schema.sam_tables;
 count
-------
    62
(1 row)
```

## Not Setting Default Privileges
Now let's log in as rhino and create a table, but without setting any default privileges for marco or anybody:
```
> \connect privtest rhino
You are now connected to database "privtest" as user "rhino".

> create table app_schema.rhino_tables as select * from pg_tables;
SELECT 63

> \dp app_schema.*
                                    Access privileges
   Schema   |     Name     | Type  |   Access privileges   | Column privileges | Policies
------------+--------------+-------+-----------------------+-------------------+----------
 app_schema | rhino_tables | table |                       |                   |
 app_schema | sam_tables   | table | bigsam=arwdDxt/bigsam+|                   |
            |              |       | marco=r/bigsam        |                   |
(2 rows)
```

As expected, we have our new table with no special access privileges granted (the owner would still have full privileges). Let's log back in as marco and see what we can see:

```
> \connect privtest marco
You are now connected to database "privtest" as user "marco".

> select count(*) from app_schema.rhino_tables;
psql:alter_default.sql:43: ERROR:  permission denied for relation rhino_tables
```

To fix this we can log back in as rhino (or a superuser) and grant select on this table:
```
> \connect privtest rhino
You are now connected to database "privtest" as user "rhino".

grant select on app_schema.rhino_tables to marco;
GRANT

> \connect privtest marco
You are now connected to database "privtest" as user "marco".
> \dp app_schema.*
                                    Access privileges
   Schema   |     Name     | Type  |   Access privileges   | Column privileges | Policies
------------+--------------+-------+-----------------------+-------------------+----------
 app_schema | rhino_tables | table | rhino=arwdDxt/rhino  +|                   |
            |              |       | marco=r/rhino         |                   |
 app_schema | sam_tables   | table | bigsam=arwdDxt/bigsam+|                   |
            |              |       | marco=r/bigsam        |                   |
(2 rows)

> select count(*) from app_schema.rhino_tables;
 count
-------
    63
(1 row)
```

A superuser could also grant privileges for all tables in the schema, e.g.:

```
# GRANT SELECT ON ALL TABLES IN SCHEMA app_schema TO marco;
GRANT
```

In this current example, the results would be the same since marco already has `SELECT` privileges on `sam_tables`.

Stay tuned for part 2, in which our hero discovers another fun fact about default privileges!
