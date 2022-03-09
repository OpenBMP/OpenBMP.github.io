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

You can configure the postgres container with your own ```postgres.conf``` as documented by the first link. If you are only
making a few changes, then you can do that via ```docker-compose.yml``` **command:** blob.  The command blob is a list of
```-c <setting>=<value>``` entries.  Below is an example of the setting for max_wall_size.  

```
    command: >
      -c max_wal_size=10GB
```

This will change the running instance of postgres.  It will not change the ```postgres.conf``` file. If you have a lot
of settings to change, then it is recommended to that via the ```postgres.conf``` file. 

### TimescaleDB Tune Script
The TimescaleDB container will configure itself using a TimescaleDB tune script. You can adjust various
parameters. 

* Adjust memory available using ```TS_TUNE_MEMORY=<n>GB``` environment variable
* Adjust cpus available using ```TS_TUNE_NUM_CPUS=<n>``` environment variable
* See [README](https://github.com/timescale/timescaledb-docker/blob/master/README.md) for more details on tuning





