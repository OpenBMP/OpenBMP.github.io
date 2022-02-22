---
title: Fresh Install Upgrade
---

# Fresh Install to Upgrade

> You can separate the compose file to be deployed using many VMs.  In order to do that, you'll need to
adjust the FQDNs and ports that are used to access resources, such as Kafka and Postgres.

1. Login to your VM and make a backup of the current docker-compose.yml
   ```cp docker-compose.yml docker-comopose.yml.bk```
2. Run ```OBMP_DATA_ROOT=/var/openbmp docker-compose -p obmp down``` to shutdown and remove the
   current deployment.
3. ```wget https://raw.githubusercontent.com/OpenBMP/obmp-docker/main/docker-compose.yml```
4. Update the docker-compose.yml file variables and volumes based on your previous compose file.
   You can use ```diff``` to see the differences that need to be merged/updated.
5. ```rm -f ${OBMP_DATA_ROOT}/config/do_not_init_db``` to allow the DB to be reinitialized
6. ```rm -rf ${OBMP_DATA_ROOT}/postgres/data/*``` and ```rm -rf ${OBMP_DATA_ROOT}/postgres/ts/*``` to
   remove the current postgres data. **Make sure the files are deleted.** If using ```sudo``` the wildcard
   doesn't work. Use ```sudo bash -c ...``` instead.
7. ```rm -rf ${OBMP_DATA_ROOT}/kafka-data/*``` to remove the current Kafka data.
8. ```rm -rf ${OBMP_DATA_ROOT}/zk-data/*; rm -rf ${OBMP_DATA_ROOT}/zk-log/*``` to remove the
   current zookeeper data.
9. Update Grafana.
    1. ```rm -rf ${OBMP_DATA_ROOT}/grafana/plugins ${OBMP_DATA_ROOT}/grafana/alerting ${OBMP_DATA_ROOT}/grafana/grafana.db```
    2. ```rm -rf ${OBMP_DATA_ROOT}/grafana/dashboards/```
    3. ```rm -rf ${OBMP_DATA_ROOT}/grafana/provisioning/```
    4. ```git clone https://github.com/OpenBMP/obmp-grafana.git``` or do a ```git pull```
    5.  ```cp -r obmp-grafana/dashboards obmp-grafana/provisioning ${OBMP_DATA_ROOT}/grafana/```
10. ```OBMP_DATA_ROOT=${OBMP_DATA_ROOT} docker-compose -p obmp up -d``` to start the new/upgraded version
    of OBMP. This will reinitialize the DB.  It does take a little time on initial start. 
