---
title: Upgrade
---

# Upgrade

> You can separate the compose file to be deployed using many VMs.  In order to do that, you'll need to
adjust the FQDNs and ports that are used to access resources, such as Kafka and Postgres.

It is possible to upgrade a current deployment without having to do a [Fresh Install Upgrade](fresh_upgrade.md). Use
this method mainly if you are upgrading from one or two dot revivisons behind. For example, 2.2.0 to 2.2.2 should be fine.
The ```psql-app``` container will attempt to do a DB schema upgrade. ```docker logs obmp-psql-app``` should show the
upgrade.

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

## 2) Get latest docker compose

```warning
The command below will try to backup the yml, but you can do it manually first as well. 
```

```
cp docker-compose.yml docker-compose.yml.bk
wget --backups=3 https://raw.githubusercontent.com/OpenBMP/obmp-docker/main/docker-compose.yml
```

## 3) Update the docker-compose.yml
Update the compose file variables and volumes based on your previous compose file.
You can use ```diff``` to see the differences that need to be merged/updated.

```
diff -u docker-compose.yml.1 docker-compose.yml
```

## 4) Update Grafana

```note
**Skip this** step if there are no changes to grafana.  You can tell if there are changes when you do a git pull. If
nothing is updated, then do not perform this step. 
```

```
git clone https://github.com/OpenBMP/obmp-grafana.git
# or do: git pull

sudo cp -r obmp-grafana/dashboards obmp-grafana/provisioning ${OBMP_DATA_ROOT}/grafana/
sudo chmod go+xr -R ${OBMP_DATA_ROOT}/grafana/
```

```danger
Make sure the files copied are owned by the container user. If not, provisioning will not load.
```
   
  
## 5) Start the new/upgraded version of OBMP 
**psql-app** should perform an upgrade to the DB schemas if needed.

The below will recreate the changed/upgraded containers. Depending on which containers are upgraded/changed,
this can be intrusive. For example, upgrading the collector causes the BMP sessions to reset due to the container
recreate. 

```
OBMP_DATA_ROOT=${OBMP_DATA_ROOT} docker-compose -p obmp up -d
```  

## 6) Sync the global IP RIB

The global IP RIB could be missing prefixes when initial RIB dumps are out of sync with the global rib cron job.
The latest changes should not result in this problem, but in case it does the
```sync_global_ip_rib()``` function can be used to sync the global rib.  The function can take a while to run and
will cause a lot of extra disk IOPS. It is recommended to run it after initial RIB dumps.


Run the below to synchronize the global IP RIB table:
```
truncate global_ip_rib;

select sync_global_ip_rib();
```  

