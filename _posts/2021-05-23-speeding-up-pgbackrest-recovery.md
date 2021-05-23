---
layout: post
title: "Speeding Up PostgreSQL Recovery with pgBackRest"
tags: postgresql pgbackrest restore recovery
date: '2021-05-23'
---

I recently had a replica that was restarted but was having trouble getting caught up enough to enable streaming replication. The WAL archive restore & recovery was taking just a little too long. The pgBackRest team offered these very helpful tips and I wanted to share them with all of you.

These three pgBackRest parameters play a key role in speeding up WAL archive restore & recovery times:

* [`archive-async`](https://pgbackrest.org/configuration.html#section-archive/option-archive-async)
* [`archive-get-queue-max`](https://pgbackrest.org/configuration.html#section-archive/option-archive-get-queue-max)
* [`spool-path`](https://pgbackrest.org/configuration.html#section-general/option-spool-path)

During recovery, PostgreSQL will, via the `restore_command` parameter, ask pgBackRest to retrieve the next WAL file and place it in the `pg_wal` directory. This is the process we want to speed up.

## `archive-async`

Enabling `archive-async` tells pgBackRest to pre-fetch WAL files so they will be locally available when PostgreSQL asks for them (which it does serially, only when the previous WAL is recovered). This is a huge win when WAL archives are stored remotely, especially in the cloud like in S3 or Azure. The WAL archives are pre-fetched and stored in the location specified by `spool-path`, and it will store up to the amount (in bytes) specified by `archive-get-queue-max`). Let's discuss these now.

```
archive-async=y
```

## `archive-get-queue-max`

By default, `archive-get-queue-max` is set to `128MB`, which is pretty small. Since a WAL file is usually 16MB, this means that only 8 WAL files could be pre-fetched and stored on disk here. I would suggest going to `1GB` right away and maybe higher (e.g. `4GB`) depending on the volume of activity involved.

Keep in mind that this space will be used in the `spool-path`, so make sure you have enough free space on that volume. Speaking of which ...

```
archive-get-queue-max=4GB
```

## `spool-path`

Last but not least, we come to `spool-path`. As mentioned earlier, this is where pgBackRest will stored the pre-fetched WAL files. Then, when PostgreSQL asks for the next WAL, pgBackRest moves the file from the `spool-path` location to the `pg_wal` directory under the PostgreSQL data directory. This is much better than waiting for pgBackRest to fetch the file from the cloud every time you ask for it, but we can make things a little better still.

```
spool-path=/var/lib/pg_wal/spool
```

### Location, location, location

When pgBackRest moves a file from one directory to another on the same volume, that is a quick `mv` operation. Basically a pointer change. However if the source and target locations are on different volumes, then we have to copy the entire file to the new location. As you may be guessing, the key here is to ensure that `spool-path` is on the same volume as the PostgreSQL `pg_wal` directory. Doing this can shave quite a few milliseconds off of the restore time (in my tests, times regularly went from ~24ms to 2ms, a 12x reduction).

It's important to note that if your `pg_wal` is symlinked to a separate volume from your data directory, then `spool-path` should be on that same separate volume. You want to be where the actual WAL files will be going, not necessarily the data directory location.

### Check Your Permissions

Normally, pgBackRest will create `spool-path` if needed (if it can as postgres). However, if `spool-path` cannot be created (or otherwise written to), then you'll get `archive-get` failures on recovery:

```
2021-05-23 21:56:52.095 UTC [21656] LOG:  starting archive recovery
2021-05-23 21:56:52.102 P00   INFO: archive-get command begin 2.33: [000000010000000000000002, pg_wal/RECOVERYXLOG] --archive-async --exec-id=21660-fdd3647d --log-level-console=info --pg1-path=/var/lib/postgresql/12/main --process-max=2 --repo1-azure-account=<redacted> --repo1-azure-container=pgbackrest --repo1-azure-key=<redacted> --repo1-cipher-pass=<redacted> --repo1-cipher-type=aes-256-cbc --repo1-path=/ --repo1-type=azure --spool-path=/var/lib/pg_wal/spool --stanza=postgres-foo
ERROR: [047]: unable to create path '/var/lib/pg_wal/spool': [13] Permission denied
2021-05-23 21:56:52.103 P00   INFO: archive-get command end: aborted with exception [047]
```

One thing to watch out for is if you try to set up a `spool-path` on a running instance, if you can't write to the path then you'll get an error on `archive-push` but the reason may not be immediately clear because pgBackRest (as of 2.33) does not write the `Permission denied` message in the PostgerSQL log:

```
ERROR: [082]: unable to push WAL file '00000002000185E70000004B' to the archive asynchronously after 60 second(s)
2021-05-19 15:46:19.372 P00   INFO: archive-push command end: aborted with exception [082]
2021-05-19 15:46:19.374 UTC [22000] LOG:  archive command failed with exit code 82
2021-05-19 15:46:19.374 UTC [22000] DETAIL:  The failed archive command was: /usr/bin/pgbackrest --stanza=postgres-foo archive-push pg_wal/000000020
00185E70000004B
2021-05-19 15:46:20.386 P00   INFO: archive-push command begin 2.32: [pg_wal/00000002000185E70000004B] --archive-async --compress-type=lz4 --exec-id=22071-481bde16 --log-level-console=info --log-level-file=info --pg1-path=/var/lib/postgresql/12/main --process-max=3 --repo1-azure-account=<redacted> --repo1-azure-container=pgbackrest --repo1-azure-key=<redacted> --repo1-cipher-pass=<redacted> --repo1-cipher-type=aes-256-cbc --repo1-path=/ --repo1-type=azure --spool-path=/var/lib/pg_wal/spool --stanza=postgres-foo
```

It does, however, write it in the pgBackRest archive-push-async log (usually found under /var/log/pgbackrest):

```
-------------------PROCESS START-------------------
2021-05-19 15:46:20.399 P00   INFO: archive-push:async command begin 2.32: [/var/lib/postgresql/12/main/pg_wal] --archive-async --compress-type=lz4 --
exec-id=22071-481bde16 --log-level-console=off --log-level-file=info --log-level-stderr=off --pg1-path=/var/lib/postgresql/12/main --process-max=3 --r
epo1-azure-account=<redacted> --repo1-azure-container=pgbackrest --repo1-azure-key=<redacted> --repo1-cipher-pass=<redacted> --repo1-cipher-type=aes-2
56-cbc --repo1-path=/ --repo1-type=azure --spool-path=/var/lib/pg_wal/spool --stanza=postgres1-test
2021-05-19 15:46:20.399 P00  ERROR: [047]: unable to create path '/var/lib/pg_wal/spool': [13] Permission denied
2021-05-19 15:46:20.399 P00   INFO: archive-push:async command end: aborted with exception [047]
```

So be sure to check that the `postgres` user (or whatever user `pgbackrest` runs as, but typically it's `postgres`) has privileges to write to (and create, if necessary) the location specified by `spool-path`.
