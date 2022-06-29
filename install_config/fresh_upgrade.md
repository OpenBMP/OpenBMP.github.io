---
title: Fresh Install Upgrade
---

# Fresh Install to Upgrade

> You can separate the compose file to be deployed using many VMs.  In order to do that, you'll need to
adjust the FQDNs and ports that are used to access resources, such as Kafka and Postgres.

## 1) Define ENV

```warning
You MUST export this var for the below commands to work.
```

```tip
Run all the below commands as **root** instead of using sudo. You can use sudo, but you'll need
to specify files for any wildcard. 
```

```
export OBMP_DATA_ROOT=/var/openbmp
```

## 2) Shutdown and remove current deployment 

```
OMP_DATA_ROOT=${OBMP_DATA_ROOT}  docker-compose -p obmp down
```

## 3) Get latest docker compose

```
wget --backups=3 https://raw.githubusercontent.com/OpenBMP/obmp-docker/main/docker-compose.yml
```

## 4) Update the docker-compose.yml
Update the compose file variables and volumes based on your previous compose file.
You can use ```diff``` to see the differences that need to be merged/updated.

```
diff -u docker-compose.yml.1 docker-compose.yml
```

## 5) Allow DB to be reinitialized

```
rm -f ${OBMP_DATA_ROOT}/config/do_not_init_db
```

## 6) Remove persistent **obmp-psql.yml**

```
mv ${OBMP_DATA_ROOT}/config/obmp-psql.yml ${OBMP_DATA_ROOT}/config/obmp-psql.yml.bk
``` 

## 7) Remove data

Remove persistent data for kafka, zookeeper, postgres and grafana.

```warning
The below does remove grafana data. Backup your grafana data first if you have custom
changes, such as custom dashboards.  You can also delete only the provisioning data.
```

```
sudo rm -rf ${OBMP_DATA_ROOT}/kafka-data
sudo rm -rf ${OBMP_DATA_ROOT}/kafka-data
sudo rm -rf ${OBMP_DATA_ROOT}/zk-data
sudo rm -rf ${OBMP_DATA_ROOT}/zk-log
sudo rm -rf ${OBMP_DATA_ROOT}/postgres/data
sudo rm -rf ${OBMP_DATA_ROOT}/postgres/ts
sudo rm -rf ${OBMP_DATA_ROOT}/grafana/

```

## 8) Recreate the persistion data directories

Create persistent data for kafka, zookeeper, and postgres.

```

sudo mkdir -m 777 ${OBMP_DATA_ROOT}/kafka-data
sudo mkdir -m 777 ${OBMP_DATA_ROOT}/zk-data
sudo mkdir -m 777 ${OBMP_DATA_ROOT}/zk-log
sudo mkdir -m 777 ${OBMP_DATA_ROOT}/postgres/data
sudo mkdir -m 777 ${OBMP_DATA_ROOT}/postgres/ts
sudo mkdir -m 777 ${OBMP_DATA_ROOT}/grafana

sudo chown -R 1000 ${OBMP_DATA_ROOT}/postgres ${OBMP_DATA_ROOT}/postgres/
sudo chown -R 1000 ${OBMP_DATA_ROOT}/postgres ${OBMP_DATA_ROOT}/postgres/data
sudo chown -R 1000 ${OBMP_DATA_ROOT}/postgres ${OBMP_DATA_ROOT}/postgres/ts

```


## 9) Update Grafana
```
git clone https://github.com/OpenBMP/obmp-grafana.git
# or do: git pull

sudo cp -r obmp-grafana/dashboards obmp-grafana/provisioning ${OBMP_DATA_ROOT}/grafana/
sudo chmod go+xr -R ${OBMP_DATA_ROOT}/grafana/
```

```danger
Make sure the files copied are owned by the container user. If not, provisioning will not load.
```
   
   
## 10) Start the new/upgraded version of OBMP 
This will reinitialize the DB.  It does take a little time
on initial start.

```
OBMP_DATA_ROOT=${OBMP_DATA_ROOT} docker-compose -p obmp up -d
```  

## 11) Update **obmp-psql.yml**
```note
Skip this step if you don't have a **custom obmp-psql.yml**
```

If you have a custom **obmp-psql.yml**, you can merge them to the updated file. The
**obmp-psql.yml** file will be created when you start ```psql-app``` container.  Run a diff
to the backup copy to identify the changes needed to be merged.  Once merged, restart the ```obmp-psql-app``` container.

## 12) Sync the global IP RIB

The global IP RIB could be missing prefixes when initial RIB dumps are out of sync with the global rib cron job.
The latest changes should not result in this problem, but in case it does the
```sync_global_ip_rib()``` function can be used to sync the global rib.  The function can take a while to run and
will cause a lot of extra disk IOPS. It is recommended to run it after initial RIB dumps.


Run the below to synchronize the global IP RIB table:
```
truncate global_ip_rib;

select sync_global_ip_rib();
```  

