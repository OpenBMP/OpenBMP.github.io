---
title: Container Configuration
---



# Container Configuration

Configuration is primarily defined in [docker-compose.yml](https://github.com/OpenBMP/obmp-docker/blob/main/docker-compose.yml).  Each container has it's own configuration. 
For the OpenBMP containers under https://github.com/OpenBMP/obmp-docker, you can get the container configuration information within each repo.  There are other containers 
that are included in the compose.  Those containers are documented under their documentation.  For example, the Kafka and Zookeeper configurations are documented under
https://docs.confluent.io/platform/current/installation/docker/config-reference.html.   


## OpenBMP Containers

OpenBMP containers use a ```/config``` volume mount for configurations.  Configuration files can be placed 
under ```${OBMP_DATA_ROOT}/config:/config``` to maintain persistent configuration.  
There are a few exceptions to these configurations that the environment variables via docker
will override the configuration file setting. 

Configuration files in ```/config``` are considered persistent and therefore may not be updated
by the docker environment variables.   Instead, you should update those configuration files
directly. 

## Postgres Container

The postgres container uses TimescaleDB container.  The TimescaleDB container uses the standard postgres
container as a base. Configuring postgres is documented under both of the below links:

* [Postgres Container](https://hub.docker.com/_/postgres)
* [TimescaleDB Container](https://docs.timescale.com/timescaledb/latest/how-to-guides/configuration/docker-config/)

### TimescaleDB Tune Script
The TimescaleDB container will configure itself using a TimescaleDB tune script. You can adjust various
parameters. 

* Adjust memory available using ```TS_TUNE_MEMORY=<n>GB``` environment variable
* Adjust cpus available using ```TS_TUNE_NUM_CPUS=<n>``` environment variable
* See [README](https://github.com/timescale/timescaledb-docker/blob/master/README.md) for more details on tuning





