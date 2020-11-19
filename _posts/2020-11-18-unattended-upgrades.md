---
layout: post
title: "Unattended Upgrades, Ubuntu 18.04, and PostgreSQL 10: The Perfect Storm"
tags: postgresql ubuntu
date: '2020-11-18'
---

We recently enabled `unattended-upgrades` on our Ubuntu 18.04 LTS (Bionic) servers, including our PostgreSQL hosts. By default, `unattended-upgrades` will ignore PGDG packages (where all of our PostgreSQL packages come from), so I had assumed we wouldn't have any interruptions due to `apt` installing updates and triggering a database restart. One evening, however, I received an alert that the nightly backup job had failed. Upon investigation I saw that the PostgreSQL 10 database had been restarted (not ideal in production), and then noticed that it had been upgraded! The apt logs under `/var/log/apt/history.log` confirmed that `unattended-upgrades` was responsible, and that it upgraded from the PGDG package to the Ubuntu-supplied version.

It seems that, while `unattended-upgrades` won't upgrade *to* a new PGDG package, it will still upgrade an installed PGDG package to the Ubuntu equivalent.

I set out to repeat this in a local VM. I started out with an Ubuntu 18.04 VM with PostgreSQL 10.13 installed from the PGDG packages (I have a local repo mirror so I still have older packages on-hand). Running `apt policy` shows what is currently installed, and that the 10.15 PGDG package is the best candidate upgrade. Note that it also sees the 10.15 Ubuntu package:

```
$ apt policy postgresql-10
postgresql-10:
  Installed: 10.13-1.pgdg18.04+1
  Candidate: 10.15-1.pgdg18.04+1
  Version table:
     10.15-1.pgdg18.04+1 500
        500 http://apt.postgresql.org/pub/repos/apt bionic-pgdg/main amd64 Packages
     10.15-0ubuntu0.18.04.1 500
        500 http://archive.ubuntu.com/ubuntu bionic-updates/main amd64 Packages
        500 http://security.ubuntu.com/ubuntu bionic-security/main amd64 Packages
 *** 10.13-1.pgdg18.04+1 100
        100 /var/lib/dpkg/status
     10.3-1 500
        500 http://archive.ubuntu.com/ubuntu bionic/main amd64 Packages
```

We then perform a manual run of `unattended upgrades`:

```
$ sudo unattended-upgrades
```

Checking `apt policy` again shows that it chose the Ubuntu package (since PGDG is excluded from the unattended-upgrades `Allowed-Origins` list):

```
$ apt policy postgresql-10
postgresql-10:
  Installed: 10.15-0ubuntu0.18.04.1
  Candidate: 10.15-1.pgdg18.04+1
  Version table:
     10.15-1.pgdg18.04+1 500
        500 http://apt.postgresql.org/pub/repos/apt bionic-pgdg/main amd64 Packages
 *** 10.15-0ubuntu0.18.04.1 500
        500 http://archive.ubuntu.com/ubuntu bionic-updates/main amd64 Packages
        500 http://security.ubuntu.com/ubuntu bionic-security/main amd64 Packages
        100 /var/lib/dpkg/status
     10.3-1 500
        500 http://archive.ubuntu.com/ubuntu bionic/main amd64 Packages
```

Checking `/var/log/apt/history.log` confirms the switcheroo:

```
Start-Date: 2020-11-19  03:37:28
Commandline: /usr/bin/unattended-upgrades
Requested-By: vagrant (1000)
Upgrade: postgresql-10:amd64 (10.13-1.pgdg18.04+1, 10.15-0ubuntu0.18.04.1)
End-Date: 2020-11-19  03:37:31

Start-Date: 2020-11-19  03:37:33
Commandline: /usr/bin/unattended-upgrades
Requested-By: vagrant (1000)
Upgrade: postgresql-client-10:amd64 (10.13-1.pgdg18.04+1, 10.15-0ubuntu0.18.04.1)
End-Date: 2020-11-19  03:37:33
```

This is unique to PostgreSQL version 10, since that is the version that Ubuntu 18.04 was released with. Checking `apt-policy` for other versions shows only the PGDG sources:

```
$ apt policy postgresql-12
postgresql-12:
  Installed: (none)
  Candidate: 12.5-1.pgdg18.04+1
  Version table:
     12.5-1.pgdg18.04+1 500
        500 http://apt.postgresql.org/pub/repos/apt bionic-pgdg/main amd64 Packages
$ apt policy postgresql-9.6
postgresql-9.6:
  Installed: (none)
  Candidate: 9.6.20-1.pgdg18.04+1
  Version table:
     9.6.20-1.pgdg18.04+1 500
        500 http://apt.postgresql.org/pub/repos/apt bionic-pgdg/main amd64 Packages
```

On Ubuntu 20.04, I would expect people to see the same problem with `postgresql-12` packages.

Now, while the Ubuntu postgresql-10 packages seem to be compatible with the PGDG packages, we still want to be on the PGDG packages and we do NOT want to have PostgreSQL restarted with any random unattended upgrades. To prevent that, we need to edit the `/etc/apt/apt.conf.d/50unattended-upgrades` configuration file and add the desired PostgreSQL packages to the blacklist:

```
Unattended-Upgrade::Package-Blacklist {
        "postgresql-*";
        "libpq5";
        "pgbackrest";
        "python3-psycopg2";
};
```

In summary, if you have `postgresql-10` on Ubuntu 18.04 (Bionic) (or `postgresql-12` on Ubuntu 20.04 (Focal)), AND are running `unattended-upgrades`, watch out for your PostgreSQL packages being updated (and subsequently having the databases restarted) if you don't blacklist them!
