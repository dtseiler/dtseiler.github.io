---
layout: post
title: "USAGE Privilege Prevents Dropping Users"
subtitle: "ACLs FTW"
tags: postgresql
date: '2018-05-11'
---

## The Incident
While cleaning up some obsolete user accounts, one of my `DROP USER` commands failed with:

```
ERROR:  2BP01: role "joe" cannot be dropped because some objects depend on it
```

Querying the usual catalog views, I couldn't find any objects owned by this user. That's when I noticed the line just after the ERROR:
```
DETAIL:  privileges for schema public
```

The \dn+ command confirmed that user joe had explicit USAGE privileges on the public schema:

```
                           List of schemas
   Name   |  Owner   |  Access privileges   |      Description
----------+----------+----------------------+------------------------
 public   | postgres | postgres=UC/postgres+| standard public schema
          |          | =UC/postgres        +|
          |          | joe=U/postgres       |
```

I revoked this and was then able to drop that user.
```
> REVOKE USAGE ON SCHEMA public FROM joe;
REVOKE
> DROP USER joe;
DROP ROLE
```

## Replicating The Problem
It's simple enough to replicate the problem, and it isn't limited to the special public schema (although that does bring up another question for later). 

### Setup
We already have the public schema, let's create another one for testing and then create our three lab rat users:
```
-- Create a test schema, we'll use this in addition to public
> CREATE SCHEMA appstuff;
CREATE SCHEMA

-- Create our three users
> CREATE USER moe;
CREATE ROLE
> CREATE USER larry;
CREATE ROLE
> CREATE USER curly;
CREATE ROLE
```

Now let's grant USAGE on the two schemas to two of the users:
```
> GRANT USAGE ON SCHEMA public TO moe;
GRANT
> GRANT USAGE ON SCHEMA appstuff TO larry;
GRANT

-- Show the access privileges on the schemas
> \dn+
                           List of schemas
   Name   |  Owner   |  Access privileges   |      Description
----------+----------+----------------------+------------------------
 appstuff | postgres | postgres=UC/postgres+|
          |          | larry=U/postgres     |
 public   | postgres | postgres=UC/postgres+| standard public schema
          |          | =UC/postgres        +|
          |          | moe=U/postgres       |
```

### DROP USER
Now that we have things set up, let's try to drop our three stooges:
```
> DROP USER moe;
psql:user_drop_usage.sql:19: ERROR:  2BP01: role "moe" cannot be dropped because some objects depend on it
DETAIL:  privileges for schema public
LOCATION:  DropRole, user.c:1045

> DROP USER larry;
psql:user_drop_usage.sql:20: ERROR:  2BP01: role "larry" cannot be dropped because some objects depend on it
DETAIL:  privileges for schema appstuff
LOCATION:  DropRole, user.c:1045

> DROP USER curly;
DROP ROLE
```

So now we've duplicated the problem with the default public role as well as a new user-created role. The `USAGE` ACL privilege clearly behaves differently than other privileges.

Using `REVOKE` to remove the ACL clears the way for `DROP USER`:
```
> REVOKE USAGE ON SCHEMA public FROM moe;
REVOKE
> DROP USER moe;
DROP ROLE

> REVOKE USAGE ON SCHEMA appstuff FROM larry;
REVOKE
> DROP USER larry;
DROP ROLE
```

So we're all cleaned up now. You can drop the appstuff schema now to really reset things. As far as `USAGE` goes, I still don't understand why PostgreSQL can't automatically revoke the privilege and drop the user. We don't have to manually revoke object privileges granted to a user before revoking it.  I'll be sure to update this post if I find out more, or discus in the comments!
