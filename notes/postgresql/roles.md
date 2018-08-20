---
layout: default
title: PostgreSQL - Role Management
---

# PostgreSQL - Role Management

Find all users in the `it_developer` role:
```
SELECT r.rolname
FROM pg_roles r
WHERE r.oid IN (
        SELECT m.member
        FROM pg_auth_members m
        JOIN pg_roles b ON (m.roleid = b.oid)
        WHERE b.rolname='it_developer'
);
```
