---
title: Using Logstash
---

# Logstash
Logstash can be used to log into ElasticSearch, plain file, InfluxDB, etc.  It can consume the 
OpenBMP parsed messages from Kafka and directly import them into another store/log/file.  The Postgres
container does not use logstash because it does a lot more than logging.  

If the use-case is to log all BGP udpates into some DB store, then Logstash is a good option.

## Installing
See the [Logstash getting started](https://www.elastic.co/guide/en/logstash/current/getting-started-with-logstash.html) instructions
to install directly or use [Logstash Docker](https://hub.docker.com/_/logstash) container.   

   
## Configuration
Below is an example

### openbmp.conf
```
input {
	kafka {
		bootstrap_servers   => "obmp-kafka:29092"
                topics              => [ "openbmp.parsed.collector",
                                         "openbmp.parsed.router",
                                         "openbmp.parsed.peer",
                                         "openbmp.parsed.bmp_stat",
                                         "openbmp.parsed.base_attribute",
                                         "openbmp.parsed.unicast_prefix",
                                         "openbmp.parsed.ls_node",
                                         "openbmp.parsed.ls_link",
                                         "openbmp.parsed.ls_prefix",
                                         "openbmp.parsed.l3vpn"
                                    ]
		group_id            => "openbmp-logstash"
		codec               => plain
		decorate_events     => true
	}
}

filter {
    # Split the message into header and body
    mutate {
        split       => [ "message", "\n\n" ]
        add_field   => { "TOPIC" => "%{[@metadata][kafka][topic]}"}
    }

    # Set the type of message
    if "T: unicast_prefix" in [message][0] {
        mutate { add_field => { "TYPE" => "unicast_prefix" } }

    } else if "T: l3vpn" in [message][0] {
        mutate { add_field => { "TYPE" => "l3vpn" } }

    } else if "T: evpn" in [message][0] {
        mutate { add_field => { "TYPE" => "evpn" } }

    } else if "T: ls_prefix" in [message][0] {
        mutate { add_field => { "TYPE" => "ls_prefix" } }

    } else if "T: ls_link" in [message][0] {
        mutate { add_field => { "TYPE" => "ls_link" } }

    } else if "T: ls_node" in [message][0] {
        mutate { add_field => { "TYPE" => "ls_node" } }

    } else if "T: collector" in [message][0] {
        mutate { add_field => { "TYPE" => "collector" } }

    } else if "T: peer" in [message][0] {
        mutate { add_field => { "TYPE" => "peer" } }

    } else if "T: router" in [message][0] {
        mutate { add_field => { "TYPE" => "router" } }

    } else if "T: bmp_stat" in [message][0] {
        mutate { add_field => { "TYPE" => "bmp_stat" } }

    } else {
        drop {}
    }

    # Split the message body into rows
    split {
	field       => "[message][1]"
	terminator  => "\n"
	target      => "row"
    }

    # Parse message based on type
    if [TYPE] == "unicast_prefix" {
        csv {
            columns=>["action","sequence","hash","router_hash","router_ip","base_attr_hash",
                      "peer_hash","peer_ip","peer_asn","timestamp","prefix","prefix_len",
                      "is_IPv4","origin","as_path","as_path_count","origin_as","next_hop","MED","local_pref",
                      "aggregator","community_list","ext_community_list","cluster_list","is_atomic_agg",
                      "is_next_hop_IPv4","originator_id", "path_id", "labels", "is_pre_policy", "is_adj_int",
                      "large_community_list" ]
    
            convert => {
                "sequence"      => "integer"
                "prefix_len"    => "integer"
                "as_path_count" => "integer"
                "MED"           => "integer"
                "local_pref"    => "integer"
            }
            separator    =>"\t"
            source       => "row"
            remove_field => ["message", "original", "event", "row" ]
        }
        
    }

    date {
	    match => ["timestamp", "YYYY-MM-dd HH:mm:ss.SSSSSS"]
	    remove_field => ["timestamp"]
    }
}

output{
  stdout {
    id => "openbmp"
  }

}
```


## Example
```danger
You **MUST** set ```config.support_escapes: true``` in **logstash.yml** or via docker environment
variable **CONFIG_SUPPORT_ESCAPES=true**
```

Copy/create ```openbmp.conf``` from the above config file. Modify it as needed.  As needed, change the
output to go to file(s), Elasticsearch, InfluxDB, etc.

```
docker run -it --rm --network obmp_default \
    -v $(pwd):/work -e CONFIG_SUPPORT_ESCAPES=true -e XPACK_MONITORING_ENABLED=false\
    logstash:8.2.2 -f /work/openbmp.conf
```
