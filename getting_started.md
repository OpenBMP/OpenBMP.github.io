---
title: Getting Started
excerpt: Getting started with OpenBMP
nav_order: 2
nav_exclude: false
search_exclude: false
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
of memory.  It would be better if you disabled RPKI validator for a small deployment where you have <= 4GB ram.  Surprisingly
RPKI validator requires greater than 2GB RAM. It also is pretty CPU intensive at times.  Looking to switch RPKI validator
for a better memory and cpu focused implementation.   

## Docker Containers
There are several docker containers defined in [docker-compose](https://github.com/OpenBMP/obmp-docker/blob/main/docker-compose.yml).

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

### (2) 