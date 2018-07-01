---
layout: post
title: "NULL Means NULL"
subtitle: "A Story of the Laziest User Recreation Script Ever"
tags: postgresql
date: '2018-07-01'
---

## Encountering the NULL

Recently I wanted to write a script to drop and recreate app users, keeping their existing passwords. For this post we'll say I have 3 users: bigsam, marco, and rhino. My first attempt something like this:

```
postgres=# select 'CREATE USER '||usename||' ENCRYPTED PASSWORD '''||passwd||''';'
postgres-# from pg_shadow
postgres-# where usename in ('bigsam','rhino','marco');

                                    ?column?
--------------------------------------------------------------------------------
 [NULL]
 [NULL]
 CREATE USER bigsam ENCRYPTED PASSWORD 'md5045f1ed469387739a262aaffe59d8bd3';
(3 rows)
```

Very curious why it only gave me the line for bigsam. Upon checking the users, I realized that marco and rhino never had their passwords set, so their password was `NULL` in the `pg_shadow` table:

```
postgres=# select usename, passwd from pg_shadow
postgres-# where usename in ('bigsam','rhino','marco');

 usename |               passwd
---------+-------------------------------------
 marco   | [NULL]
 rhino   | [NULL]
 bigsam  | md5045f1ed469387739a262aaffe59d8bd3
(3 rows)
```
... and when you concatenate a string with `NULL`, the answer will be `NULL`, for example:

```
postgres=# select 'foo';
 ?column?
----------
 foo
(1 row)

postgres=# select 'foo'||null;
 ?column?
----------
 [NULL]
(1 row)
```

## Handling the NULL

So I needed to handle cases where the password was NULL. My second, lazy attempt was to just set the password to an empty string if it was previously NULL:

```
postgres=# select 'CREATE USER '||usename||' ENCRYPTED PASSWORD '''||coalesce(passwd,'')||''';'
postgres-# from pg_shadow
postgres-# where usename in ('bigsam','rhino','marco');

                                    ?column?
--------------------------------------------------------------------------------
 CREATE USER marco ENCRYPTED PASSWORD '';
 CREATE USER rhino ENCRYPTED PASSWORD '';
 CREATE USER bigsam ENCRYPTED PASSWORD 'md5045f1ed469387739a262aaffe59d8bd3';
(3 rows)
```

But clearly that isn't the right way to go about things, since I'm now setting a password (and a rather bad one) when there shouldn't be one. The correct way would be to just not set the password, like so:

```
postgres=# select 'CREATE USER '||usename||
postgres-#     case when passwd is null then ''
postgres-#     else ' ENCRYPTED PASSWORD '''||passwd||''''
postgres-#     end
postgres-#     ||';'
postgres-# from pg_shadow
postgres=# where usename in ('bigsam','rhino','marco');

                                   ?column?
-------------------------------------------------------------------------------
 CREATE USER marco;
 CREATE USER rhino;
 CREATE USER bigsam ENCRYPTED PASSWORD 'md5045f1ed469387739a262aaffe59d8bd3;
(3 rows)
```

This works for my purposes but obviously it doesn't handle all of the various options for user creation. I knew in my case that none of those applied for this set of users, so I didn't pursue it. I'll leave that as a fun exercise for the reader.

An alternative would have been to run `pg_dumpall` with the `--globals-only` option and then grab the lines from there to recreate the users, with all the options explicitly set or not set:

```
pg_dumpall --globals-only > globals.sql

CREATE ROLE bigsam;
ALTER ROLE bigsam WITH NOSUPERUSER INHERIT NOCREATEROLE NOCREATEDB LOGIN NOREPLICATION NOBYPASSRLS PASSWORD 'md5045f1ed469387739a262aaffe59d8bd3';
CREATE ROLE marco;
ALTER ROLE marco WITH NOSUPERUSER INHERIT NOCREATEROLE NOCREATEDB LOGIN NOREPLICATION NOBYPASSRLS;
CREATE ROLE rhino;
ALTER ROLE rhino WITH NOSUPERUSER INHERIT NOCREATEROLE NOCREATEDB LOGIN NOREPLICATION NOBYPASSRLS;
```
