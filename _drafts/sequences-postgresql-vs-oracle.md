---
layout: post
title: "Sequence Caching: Oracle vs. PostgreSQL"
#subtitle: "ACLs FTW"
tags: postgresql oracle
date: '2018-06-02'
---

## What Is Sequence Caching?

Many RDBMSes use sequence caching to pre-generate sequence values in advance, allowing them to return new values quickly. If you are doing a lot of inserts that each call the sequence to get the next value, this is a good thing. Having to wait for the sequence to generate a value on every call would slow things down greatly.

When I made the move from Oracle to PostgreSQL, I noticed some interesting differences in sequence caching behavior and wanted to share them here.

All examples below were conducted in Oracle 12.2.0.1 (12c Enterprise Edition) and PostgreSQL 9.6.9 (via [PGDG Yum Repository](https://yum.postgresql.org/repopackages.php)).

## Cache Value Sharing

In Oracle, when a sequence cache is generated, all sessions access the same cache. However in PostgreSQL, each session gets its own cache.

### Oracle
First let's create our sequence, using a cache size of 25. In all examples here, I'm using Oracle 12c (12.2.0.1).

```
SQL> CREATE SEQUENCE dts_seq
  2          INCREMENT BY 1
  3          CACHE 25;

Sequence created.
```

Now I can open a couple of sessions and call `nextval` on it:

```
-- First session
SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
         1

SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
         2

SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
         3


-- Second session
SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
         4

SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
         5

SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
         6
```

It's even more clear when alternating between the two sessions:

```
-- Session A
SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
       151

SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
       153
```

At the same time:
```
-- Session B
SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
       152

SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
       154
```

So we've seen clearly that all sessions are pulling from the same cache of values in Oracle.

### PostgreSQL

In PostgreSQL, one of the first things I noticed was that multiple sessions were each using their own cache pool.

As before, let's create a sequence with a cache of 25:

```
CREATE SEQUENCE dts_seq
        INCREMENT BY 1
        CACHE 25;
CREATE SEQUENCE
```

Now let's open our first session and pull a few values:

```
SELECT nextval('dts_seq');
 nextval
---------
       1
(1 row)

SELECT nextval('dts_seq');
 nextval
---------
       2
(1 row)

SELECT nextval('dts_seq');
 nextval
---------
       3
(1 row)
```

Now le's open our second session and pull some values:

```
SELECT nextval('dts_seq');
 nextval
---------
      26
(1 row)

SELECT nextval('dts_seq');
 nextval
---------
      27
(1 row)

SELECT nextval('dts_seq');
 nextval
---------
      28
(1 row)
```

So you can see that the second session started at value 26, outside of the initial cache of 25. I can now go back to session one and it would pick up where it left off:

```
SELECT nextval('dts_seq');
 nextval
---------
       4
(1 row)

SELECT nextval('dts_seq');
 nextval
---------
       5
(1 row)
```

But any new session will get its own set of 25 cached values beginning after the end of any previously generated caches. So a third session would start at 51, and fourth at 76, etc.


## Shutdowns
* Demonstrate cache values being preserved in Oracle with consistent shutdown, but lost on abort.
* Demonstrate cache value being lost in PostgreSQL with both shutdown types.
* Why are extra values lost on abort in PostgreSQL
