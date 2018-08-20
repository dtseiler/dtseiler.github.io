---
layout: default
title: PostgreSQL - psql Notes
---

# PostgreSQL - psql Notes

## Logging Output
The psql -L and -o options don't show timings. Just pipe to tee!
```
psql -aX -f test.sql 2>&1 | tee -a test.log
```
*The -a will force echo all output, the -X will ignore any .psqlrc settings (which is preferred when running scripts).*

Or if you want to have dynamic SQL in a bash script, for example:
```
psql -aX -d btfprod <<EOSQL 2>&1 | tee -a test.log
        SELECT now();
EOSQL
```

