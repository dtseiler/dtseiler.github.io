---
layout: post
title: "Sequence Caching: Oracle vs. PostgreSQL"
tags: postgresql oracle
date: '2018-10-02'
---

Many RDBMSes use sequence caching to pre-generate sequence values in advance and store them in memory, allowing them to return new values quickly. If you are doing a lot of inserts that each call the sequence to get the next value, sequence caching is a good thing. Having to wait for the sequence to generate a value on every call could slow things down greatly.

When I made the move from Oracle to PostgreSQL, I noticed some interesting differences in sequence caching behavior and wanted to share them here for those that may be curious about them.

All examples below were conducted in Oracle 12.2.0.1 (12c Enterprise Edition) and PostgreSQL 9.6.9 (via [PGDG Yum Repository](https://yum.postgresql.org/repopackages.php)).

## Cache Value Sharing

In Oracle, when a sequence cache is generated, all sessions access the same cache. However in PostgreSQL, each session gets its own cache. We demonstrate this with a couple of quick-and-easy examples below.

### Oracle
First let's create our sequence, using a cache size of 25.

```
SQL> CREATE SEQUENCE dts_seq
  2          INCREMENT BY 1
  3          CACHE 25;

Sequence created.
```

Now I can open a couple of sessions and call `nextval` on that sequence to increment it from two different sessions:

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

-- Session B
SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
       152

-- Session A
SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
       153

-- Session B
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

Things get a little interesting when we see how each DBMS handles cached values across different types of shutdowns. Generally there are two types of shutdowns, consistent and inconsistent. With a consistent shutdown, things are safely preserved and written to disk before the database processes are terminated. An inconsistent (sometimes called "abort") shutdown is a fast termination of all database processes, akin to a sudden power loss. Data in memory is lost, and the database will have to perform crash recovery from redo logs or WAL files upon the next startup.

## Oracle
### Consistent Shutdown

On consistent shutdown in Oracle, a sequence starts where it left off:

```
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

SQL> shutdown immediate;
SQL> startup

SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
	 7

SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
	 8

SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
	 9
```

### Inconsistent Shutdown
On inconsistent shutdowns, Oracle discards all cached values and starts at the end of the previous cache on the next startup. In this case the cache started at 7 on the last population, so with a cache of 25 it would start at 32 (7+25).

```
SQL> shutdown abort;
SQL> startup

SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
	32

SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
	33

SQL> SELECT dts_seq.nextval FROM dual;

   NEXTVAL
----------
	34
```

If we were to repeat another `shutdown abort`, it would start at 32+25=57 and so on.

### PostgreSQL

### Consistent Shutdown

PostgreSQL will discard any cached values no matter what type of shutdown is used. In the case of a consistent shutdown, any opened caches will have their values discarded unused, and on the next startup and `nextval()` call, the sequence will start where the last cache ended:

```
postgres=# CREATE SEQUENCE dts_seq
postgres-#         INCREMENT BY 1
postgres-#         CACHE 25;
CREATE SEQUENCE
postgres=# SELECT nextval('dts_seq');
 nextval
---------
       1
(1 row)

postgres=# SELECT nextval('dts_seq');
 nextval
---------
       2
(1 row)

postgres=# SELECT nextval('dts_seq');
 nextval
---------
       3
(1 row)
```

Now perform a consistent shutdown and restart:

```
$ pg_ctl stop -m fast
$ pg_ctl start -w
```

Now when we request sequence values, we'll see the first cached set of 25 are discarded, even though we only used 3:

```
postgres=# SELECT nextval('dts_seq');
 nextval
---------
      26
(1 row)

```

### Inconsistent Shutdown

With PostgreSQL inconsistent shutdowns, we notice something different. As with consistent shutdowns, any cached sequence values are discarded. But the sequences start even further from the leaving point:

```
$ pg_ctl stop -m immediate
$ pg_ctl start -w

...

postgres=# SELECT nextval('dts_seq');
 nextval
---------
      83
(1 row)
```

Given that the sequence started at 26 after my last shutdown, I expected the sequence to start at 51 (26 plus the cache size of 25) the next time. When I played with different cache sizes of 50 and 100, I saw every time that the sequence start value after an inconsistent shutdown was always the cache size plus 32. A search of the PostgreSQL source code (and a nudge from Stephen Frost) turned up this [`define` in sequence.c](https://github.com/postgres/postgres/blob/master/src/backend/commands/sequence.c#L49):

```
/*
 * We don't want to log each fetching of a value from a sequence,
 * so we pre-log a few fetches in advance. In the event of
 * crash we can lose (skip over) as many values as we pre-logged.
 */
#define SEQ_LOG_VALS	32
```

So the gap is based on this hard-coded variable, which could obviously be changed if one really felt the need to for whatever reason.

## Conclusion

I hope these examples helped to illustrate the differences and answer any questions my dear readers may have had about sequence caching. It's not a very sexy topic but one of the first differences I noticed when I made the leap from Oracle to PostgreSQL. Feel free to try these out for yourself and comment if you have any comments or questions. I'd be interested as well to see what is done in MySQL/Maria or MS SQL Server.
