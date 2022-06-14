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
                                         "openbmp.parsed.unicast_prefix",
                                         "openbmp.parsed.ls_node",
                                         "openbmp.parsed.ls_link",
                                         "openbmp.parsed.ls_prefix",
                                         "openbmp.parsed.l3vpn",
                                         "openbmp.parsed.evpn"
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
                      "is_next_hop_IPv4","originator_id", "path_id", "labels", "is_pre_policy", "is_adj_in",
                      "large_community_list" ]
    
            convert => {
                "sequence"      => "integer"
                "peer_asn"      => "integer"    
                "origin_as"     => "integer"                            
                "prefix_len"    => "integer"
                "path_id"       => "integer"                
                "as_path_count" => "integer"
                "MED"           => "integer"
                "local_pref"    => "integer"
            }
            separator    => "\t"
            source       => "row"
            remove_field => ["message", "original", "event", "row" ]
        }

    } else if [TYPE] == "l3vpn" {
        csv {
            columns=>["action","sequence","hash","router_hash","router_ip","base_attr_hash",
                      "peer_hash","peer_ip","peer_asn","timestamp","prefix","prefix_len",
                      "is_IPv4","origin","as_path","as_path_count","origin_as","next_hop","MED","local_pref",
                      "aggregator","community_list","ext_community_list","cluster_list","is_atomic_agg",
                      "is_next_hop_IPv4","originator_id", "path_id", "labels", "is_pre_policy", "is_adj_in",
                      "rd", "rd_type", 
                      "large_community_list" ]
    
            convert => {
                "sequence"      => "integer"
                "peer_asn"      => "integer"    
                "origin_as"     => "integer"      
                "path_id"       => "integer"                                      
                "prefix_len"    => "integer"
                "as_path_count" => "integer"
                "MED"           => "integer"
                "local_pref"    => "integer"
                "rd_type"       => "integer"
            }
            separator    => "\t"
            source       => "row"
            remove_field => ["message", "original", "event", "row" ]
        }

    } else if [TYPE] == "ls_node" {
        csv {
            columns=>["action","sequence","hash","base_attr_hash","router_hash","router_ip",
                      "peer_hash","peer_ip","peer_asn","timestamp",
                      "igp_router_id","router_id","routing_id","ls_id","mt_id",
                      "ospf_area_id","isis_area_id","protocol","flags", 
                      "as_path","local_pref","MED","next_hop",
                      "node_name",
                      "is_pre_policy", "is_adj_in",
                      "rd", "rd_type", 
                      "sr_capabilities" ]
    
            convert => {
                "sequence"      => "integer"
                "MED"           => "integer"
                "peer_asn"      => "integer"                
                "local_pref"    => "integer"
                "mt_id"         => "integer"
                "ls_id"         => "integer"
                "routing_id"    => "integer"                                
            }
            separator    => "\t"
            source       => "row"
            remove_field => ["message", "original", "event", "row" ]
        }

    } else if [TYPE] == "ls_link" {
        csv {
            columns=>["action","sequence","hash","base_attr_hash","router_hash","router_ip",
                      "peer_hash","peer_ip","peer_asn","timestamp",
                      "igp_router_id","router_id","routing_id","ls_id",
                      "ospf_area_id","isis_area_id","protocol", 
                      "as_path","local_pref","MED","next_hop",
                      "mt_id","local_link_id","remote_link_id","interface_ip",
                      "neighbor_ip","igp_metric","admin_group","max_link_bw",
                      "max_resv_bw","unreserved_bw","te_default_metric",
                      "link_protection","mpls_proto_mask","srlg",
                      "link_name","remote_node_hash","local_node_hash",
                      "remote_igp_router_id","remote_router_id",
                      "local_node_asn","remote_node_asn",
                      "epe_peer_ndoe_sid",
                      "is_pre_policy", "is_adj_in",
                      "adj_segment_id"
                      ]
    
            convert => {
                "sequence"          => "integer"
                "MED"               => "integer"                
                "local_pref"        => "integer"
                "peer_asn"          => "integer"                
                "routing_id"        => "integer"                
                "mt_id"             => "integer"
                "ls_id"             => "integer"
                "local_link_id"     => "integer"
                "remote_link_id"    => "integer"
                "igp_metric"        => "integer"
                "admin_group"       => "integer"
                "max_link_bw"       => "integer"
                "max_resv_bw"       => "integer"
                "te_default_metric" => "integer"
                "local_node_asn"    => "integer"
                "remote_node_asn"   => "integer"
            }
            separator    => "\t"
            source       => "row"
            remove_field => ["message", "original", "event", "row" ]
        }


    } else if [TYPE] == "ls_prefix" {
        csv {
            columns=>["action","sequence","hash","base_attr_hash","router_hash","router_ip",
                      "peer_hash","peer_ip","peer_asn","timestamp",
                      "igp_router_id","router_id","routing_id","ls_id",
                      "ospf_area_id","isis_area_id","protocol", 
                      "as_path","local_pref","MED","next_hop",
                      "local_node_hash","mt_id","ospf_route_type",
                      "igp_flags","route_tag","ext_route_tag",
                      "ospf_fwd_addr","igp_metric",
                      "prefix","prefix_len",
                      "is_pre_policy", "is_adj_in",
                      "prefix_sid"
                      ]
    
            convert => {
                "sequence"          => "integer"
                "MED"               => "integer"
                "local_pref"        => "integer"
                "peer_asn"          => "integer"
                "prefix_len"        => "integer"                
                "routing_id"        => "integer"                
                "mt_id"             => "integer"                
                "ls_id"             => "integer"
                "route_tag"         => "integer"
                "ext_route_tag"     => "integer"
                "igp_metric"        => "integer"
                "te_default_metric" => "integer"
                "local_node_asn"    => "integer"
                "remote_node_asn"   => "integer"
            }
            separator    => "\t"
            source       => "row"
            remove_field => ["message", "original", "event", "row" ]
        }

    } else if [TYPE] == "evpn" {
        csv {
            columns=>["action","sequence","hash","router_hash","router_ip","base_attr_hash",
                      "peer_hash","peer_ip","peer_asn","timestamp",
                      "origin","as_path","as_path_count","origin_as","next_hop","MED","local_pref",
                      "aggregator","community_list","ext_community_list","cluster_list","is_atomic_agg",
                      "is_next_hop_IPv4","originator_id", "path_id","is_pre_policy", "is_adj_in",
                      "rd","rd_type","origin_router_ip_len","origin_router_ip",
                      "eth_tag_id","eth_segment_id","mac_len","mac",
                      "ip_len", "ip", "mpls_label_1","mpls_label_2",
                      "large_community_list" ]
    
            convert => {
                "sequence"      => "integer"
                "peer_asn"      => "integer"
                "origin_as"     => "integer"
                "path_id"       => "integer"
                "as_path_count" => "integer"
                "MED"           => "integer"
                "local_pref"    => "integer"
                "rd_type"       => "integer"                
                "mac_len"       => "integer"
                "ip_len"        => "integer"
                "mpls_label_1"  => "integer"
                "mpls_label_2"  => "integer"                
                "origin_router_ip_len" => "integer"
            }
            separator    => "\t"
            source       => "row"
            remove_field => ["message", "original", "event", "row" ]
        }        

    } else if [TYPE] == "collector" {
        csv {
            columns => ["action","sequence","admin_id","hash","routers","router_count","timestamp"]
    
            convert => {
                "sequence"      => "integer"
                "router_count"  => "integer"                                      
            }
            separator    => "\t"
            source       => "row"
            remove_field => ["message", "original", "event", "row" ]
        }

    } else if [TYPE] == "router" {
        csv {
            columns => ["action","sequence","name","hash","ip","descr",
                        "term_code","term_reason","init_data",
                        "term_data","timestamp", "bgp_id"]
    
            convert => {
                "sequence"      => "integer"
                "term_code"     => "integer"                                      
            }
            separator    => "\t"
            source       => "row"
            remove_field => ["message", "original", "event", "row" ]
        }
        
    } else if [TYPE] == "peer" {
        csv {
            columns => ["action","sequence","hash","router_hash","name","remote_bgp_id",
                        "router_ip","timestamp","remote_asn","remote_ip","peer_rd",
                        "remote_port","local_asn","local_ip","local_port","local_bgp_id",
                        "info_data","adv_cap","recv_cap","remote_holddown","adv_holddown",
                        "bmp_reason","bgp_error_code","bgp_error_subcode","error_text",
                        "is_l3vpn","is_pre_policy","is_ipv4","is_loc_rib","is_loc_rib_filtered",
                        "table_name"
                        ]
    
            convert => {
                "sequence"          => "integer"
                "remote_asn"        => "integer"
                "remote_port"       => "integer"
                "local_asn"         => "integer"
                "local_port"        => "integer"
                "remote_holddown"   => "integer"
                "adv_holddown"      => "integer"
                "bmp_reason"        => "integer"
                "bgp_error_code"    => "integer"
                "bgp_error_subcode" => "integer"                                      
            }
            separator    => "\t"
            source       => "row"
            remove_field => ["message", "original", "event", "row" ]
        }        

    } else {
        drop {}
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
