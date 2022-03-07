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

