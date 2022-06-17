---
title: Postgres
---

# Postgres Troubleshooting

## Missing Tables

### Symptom
You might see errors in ```obmp-psql.log``` indicating missing tables or in Grafana you'll have
blank/empty graphs. 

### Cause
There are several time series tables (www.timescale.com) that are configured to use
[postgres tablespace](https://www.postgresql.org/docs/14/manage-ag-tablespaces.html) **timeseries**.  This
tablespace is created when you first install the ```openbmp/postgres``` container. 

If you are missing the timeseries tablespace, then several tables will be missing.

The reason for this issue is often related to permission problems with the ```${OBMP_DATA_ROOT}/postgres/ts```
mount to the ```openbmp/postgres``` container and when using another postgres deployment. 

### Fix
If you are hitting permission problems for ```${OBMP_DATA_ROOT}/postgres/ts``` with the ```openbmp/postgres```
container, then you should be able to fix that by doing a ```chmod 7777 -R ${OBMP_DATA_ROOT}/postgres/ts```.  Also,
make sure that you have ```export OBMP_DATA_ROOT=/var/openbmp``` set to the OBMP_DATA_ROOT. After fixing, you'll need
to reinitialize the DB as indicated below.

If you are not using the ```openbmp/postgres``` container, then you will need to manually
configure the **timeseries** tablespace.  This can use the same disk/volume as main, but normally
you would dedicate storage for the time series data to allow for larger retention times.

See [obmp-docker/postgres/Dockerfile](https://github.com/OpenBMP/obmp-docker/blob/main/postgres/Dockerfile) for
details on how it creates the **timeseries** tablespace.

After creating the timeseries tablespace, you'll need to re-initialize the DB. You can do that by
running the following commands. 

```warning
The below will erase your current DB. If you do not want to do that, then you'll need to manually
add the timeseries tables. This is in [obmp-psql/database](https://github.com/OpenBMP/obmp-psql/blob/main/database/1_base.sql).
You will also need to add the views that are missing becuase the tables were not created the first time.
There are only a few views that reference the timeseries tables. 
```

```
rm -f ${OBMP_DATA_ROOT}/config/do_not_init_db
OBMP_DATA_ROOT=${OBMP_DATA_ROOT} docker-compose -p obmp stop psql-app
OBMP_DATA_ROOT=${OBMP_DATA_ROOT} docker-compose -p obmp up -d
```

## Checkpoints occurring too frequently

### Symptom

The below messages are often in the logs output:

```
2022-03-08 13:41:38.169 UTC [87] HINT:  Consider increasing the configuration parameter "max_wal_size".
2022-03-08 13:41:54.370 UTC [87] LOG:  checkpoints are occurring too frequently (16 seconds apart)
```

### Cause

The cause is the setting of **max_wal_size** being too low based on the change rate of your updates/inserts.
The default is to have this set to **10GB**. 

### Fix

You can increase this by changing the value below in ```docker-compose.yml```:

```
    command: >
      -c max_wal_size=10GB
```

## Global IP RIB is not updated with all prefixes

The global IP RIB could be missing prefixes when initial RIB dumps are out of sync with the global rib cron job. The
```sync_global_ip_rib()``` function can be used to sync the global rib.  The function can take a while to run and 
will cause a lot of extra disk IOPS. It is recommended to run it after initial RIB dumps. 

Run ```select sync_global_ip_rib();``` to synchronize the global IP RIB table. 
