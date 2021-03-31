---
title: Getting Started
sort: 2
---

# Getting Started with OpenBMP

This guide walks through the steps to install an OpenBMP system using PostgreSQL on
a single VM instance using docker-compose. 

## VM Sizing
Sizing depends on the number of prefixes being monitored and the number of BGP updates
per second.  Internet peering with approx. 850,000 IPv4 prefixes has normally a rate
of 15 updates per second.  Internal peering, such as with remote sites, should be very
stable with less than 1 update a second.  In fact, internal peering has very little to
no updates per second and instead has bursts of updates (normally less than 1000) every
so often. 

When monitoring internet peers with full IPv4 routing tables, you will need to size
the main persistent storage to 1GB per peer with 50MB per day for updates (timeseries storage).
Internal peering is totally different.  It's expected that internal peering is not as
chatty as internet full table peering/monitoring.  For internal peering with
say less than 5000 prefixes with updates averaging at 5000 per day, you should size
the main storage at 50MB and 2MB for timeseries per peer.   

Disk IOPS are the bottleneck in terms of performance.  You should use fast SSD
or array disks to achieve high IOPS.  A fast SSD has a sustained rate of >= 10000
IOPS per second.  This works very well.   The target IOPS needed is >= 5000.  

Memory is not such as big deal in terms of data collection. It's more on the SQL queries that
require a lot of memory.  The number of connections makes a **huge** difference.  If you
plan to support many connections accessing and running queries, you will need more
memory.  You should be able to support <= 100 concurrent connections to PostgreSQL
with a system that has **32GB of RAM**. Responses times to queries that use indexes
should be less than 2 seconds.  

The number of vCPUs is also driven by the number of concurrent connections to
PostgreSQL.  A good starting point is to have at least **16 vCPUs**.

Small deployments with less than 1M prefixes and less than 100,000 updates per day can easily run on a system
with 4vCPUs, 4GB RAM and 80GB disk (<1000 IOPS).  Please **note** that [RPKI validator](https://github.com/RIPE-NCC/rpki-validator-3) tends to take up a bit
of memory.  It would be better if you disabled RPKI validator for a small deployment where you have <= 4GB RAM.  Surprisingly
RPKI validator requires greater than 2GB RAM. It is also CPU intensive at times.  OpenBMP is looking to switch RPKI validator
for a better memory and cpu focused implementation.   

## Docker Containers
There are several docker containers defined in [OpenBMP docker-compose.yml](https://github.com/OpenBMP/obmp-docker/blob/main/docker-compose.yml).

These containers can be distributed over more than one VM/host.  These can also be added to kubernetes clusters, but make
sure that you have the IOPS needed for PostgreSQL.  

Kafka, Zookeeper, and Grafana are standard off-the-shelf. You can use any Kafka and Grafana install as long as you
note the settings that are used. 

PostgreSQL container is basically the same container as [docker timescale/timescaledb](https://hub.docker.com/r/timescale/timescaledb),
with some extra tuning applied.  You can use any PostgreSQL cluster, but note that we do use (and require) >= version 12 of PostgreSQL. 
See [postgres Dockerfile](https://github.com/OpenBMP/obmp-docker/blob/main/postgres/Dockerfile) for details on what you
will need to configure in your postgres cluster.

## Steps to bring up OpenBMP on a single VM

### (1) Install docker and docker-compose. 
* Follow the [Docker Install](https://docs.docker.com/installation/) to install a current version of docker.  
* Follow the [Docker Compose Install](https://docs.docker.com/compose/install/) to install a current version of docker-compose.

### (2) Download Files

```
wget https://raw.githubusercontent.com/OpenBMP/obmp-docker/main/docker-compose.yml 
git clone https://github.com/OpenBMP/obmp-grafana.git
```

### (3) Setup persistent storage
You can re-use existing configurations, grafana dashboards, etc.

#### (3.a) Define a root location
Normally we place everything in ```/var/openbmp```, but you can place it anywhere.

```
export OBMP_DATA_ROOT=/var/openbmp
sudo mkdir -p $OBMP_DATA_ROOT
sudo chmod -R 7777 $OBMP_DATA_ROOT
```

#### (3.b) Create sub-directories
To keep it simple, we normally create the sub-directories under the root.  

> **NOTE**
> You can mount other partitions/slices or external disks using anyone of these paths. It is advisable
> that you at least mount/partition both ```${OBMP_DATA_ROOT}/postgres/data``` and ```${OBMP_DATA_ROOT}/postgres/ts```
> Postgres **data** partition doesn't require as much as **ts**. **ts** (timeseries) size depends on
> how long you want to keep data.  The default is 4 weeks of history.

```
mkdir -p ${OBMP_DATA_ROOT}/config
mkdir -p ${OBMP_DATA_ROOT}/kafka-data
mkdir -p ${OBMP_DATA_ROOT}/zk-data
mkdir -p ${OBMP_DATA_ROOT}/zk-log
mkdir -p ${OBMP_DATA_ROOT}/postgres/data
mkdir -p ${OBMP_DATA_ROOT}/postgres/ts
mkdir -p ${OBMP_DATA_ROOT}/grafana

chmod -R 7777 $OBMP_DATA_ROOT/*
```

### (4) Copy Grafana Provisioning
Copy the grafana provisioning data from the repo to the ```${OBMP_DATA_ROOT}/grafana``` directory. 

```
cp -r obmp-grafana/dashboards obmp-grafana/provisioning ${OBMP_DATA_ROOT}/grafana/
```

> **NOTE**:
> Repeat the above when grafana provisioning has changed.


### (5) Customize the docker-compose.yml settings
Edit the ```docker-compose.yml``` file and tune the config for your deployment.

* Change the **MEM** environment variable value (in GB) based on your install in both
  the ```psql``` and ```psql-app``` containers. 


### (6) Run OpenBMP docker-compose.yml

```
OBMP_DATA_ROOT=/var/openbmp docker-compose -f ./docker-compose.yml -p obmp up -d
```

Containers will restart if they crash or if the system is rebooted.

### (7) Set grafana home

* Login to grafana via http://<ip/hostname>:3000/
* Click the link at the bottom left (above the help icon) to **Sign In**.
* Sign in as **admin** using the password in the compose file
* Click on the dashbard icon (middle left) and select **Manage**
* Click on **Home** to open that dashoard
* In the upper left, next to the dashboad name of **Home** there is a **star** icon. Click that. 
* Click on the user icon (same to sign in) and select **Preferences**
* Under **Preferences** select **Home** from the **Home Dashboard** list. 
* Click Save. 

Now when you login, the home dashboard will be set.

> TODO: Will try to fix this via grafana provisioning or similar.


### (8) Verify

You should have a similar output as below from ```docker ps``:

```
CONTAINER ID        IMAGE                             COMMAND                  CREATED             STATUS              PORTS                                            NAMES
f3e1e6861593        openbmp/psql-app:build-50         "/usr/sbin/run"          32 minutes ago      Up 20 minutes       0.0.0.0:8080->8080/tcp, 0.0.0.0:9005->9005/tcp   obmp-psql-app
7a0e1393fffa        confluentinc/cp-kafka:6.0.2       "/etc/confluent/dock…"   About an hour ago   Up About an hour    0.0.0.0:9092->9092/tcp                           obmp-kafka
f3de5896671d        grafana/grafana:latest            "/run.sh"                About an hour ago   Up About an hour    0.0.0.0:3000->3000/tcp                           obmp-grafana
b4250fd91080        openbmp/collector:build-50        "/usr/sbin/run"          About an hour ago   Up About an hour    0.0.0.0:5000->5000/tcp                           obmp-collector
7c8aa1766f9d        openbmp/postgres:build-50         "docker-entrypoint.s…"   About an hour ago   Up About an hour    0.0.0.0:5432->5432/tcp                           obmp-psql
379afacb3b1d        confluentinc/cp-zookeeper:6.0.2   "/etc/confluent/dock…"   About an hour ago   Up About an hour    2181/tcp, 2888/tcp, 3888/tcp                     obmp-zookeeper
```

You should be able to login to grafana at http://<vm ip/name>:3000/

### (9) Configure Routers
Configure routers to send BMP to ```<vm ip/hostname> port 5000```


- - -



## Troubleshooting

### Check Consumer Lag

```
docker exec -it obmp-psql-app /bin/bash
kafka-tools -b obmp-kafka:29092 print_consumer_lag openbmp.parsed.unicast_prefix obmp-psql-consumer
```

You should see positive numbers in the offset columns and a lag normally less than a few thousand. 
When you have a lot of peers that are all RIB dumping at the same time, the lag might be in the
millions. This should not last too long and it should normalize where you see a lag less than 1000.

### Clear and start a container

```
OBMP_DATA_ROOT=/var/openbmp docker-compose -p obmp stop <container/service>
OBMP_DATA_ROOT=/var/openbmp docker-compose -p obmp rm <container/service>

# Optionally also remove the image
# docker rmi <image name>:<tag>

OBMP_DATA_ROOT=/var/openbmp docker-compose -p obmp up -d <container/service>
```

### Shutdown and remove all containers

```
OBMP_DATA_ROOT=/var/openbmp docker-compose -p obmp down
```

### Restart a container 

```
OBMP_DATA_ROOT=/var/openbmp docker-compose -p obmp stop <container>
OBMP_DATA_ROOT=/var/openbmp docker-compose -p obmp start <container>

# Use the below if you want to recreate and start it 
OBMP_DATA_ROOT=/var/openbmp docker-compose -p obmp up -d <container>
```


### Fix issues relating to persistent data
Sometimes persistent data can become a problem for a container.  You can attempt to fix it or 
just purge the data and start over. 

To purge and start over, start by purging only the specific data that might be a problem,
such as kafa-data or postgres.  If that doesn't work, remove them all and then recreate and
populate as mentioned in the above install steps.

```rm -rf /var/openbmp/*```
