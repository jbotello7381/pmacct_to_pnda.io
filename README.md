PMACCT to PNDA.IO Communication
===============================

## Problem
PNDA.IO use avro as part of how data is serialized/encoded/decoded in and out of kafka.  Currently, it use kafka
ByteArraySerialization.  PMACCT is a multipurpose of passive monitoring tools that can collect, aggregate, replicate
and export forwarding plane data (ipv4/ipv6) as flow records.  As part of its capabilities, it can forward these
flow records via kafka messages with avro encoding.   However, current kafka plugin doesn’t seems to support byte array 
serialization of the data inside the kafka messages.

In order to demonstrate this, we configure pmacct to use kafka messages and avro serialization.  

IPFix (netflow ver 10) records are collected by pmacct and sent it to pnda kafka bus.   After these messages are
collected and then stored in pnda hdfs, if we try to decode the information inside those files, we clearly see that
pnda can’t deserialize the messages and store then, it move them into a quarantine folder.


<pre>
ubuntu@ip-10-180-221-47:~/datasets$ java -jar avro-tools-1.8.2.jar tojson f0f01acf-5011-42ec-90b0-c18f21e4e2ab.avro
log4j:WARN No appenders could be found for logger (org.apache.hadoop.metrics2.lib.MutableMetricsFactory).
log4j:WARN Please initialize the log4j system properly.
log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
{"topic":"netflow","timestamp":1521676850049,"reason":{"string":<b>"Unable to deserialize data"</b>},"payload":
"{\"event_type\": \"purge\", \"as_src\": 65000, \"as_dst\": 65100, \"peer_ip_src\": \"REMOVED\", \"peer_ip_dst\":
\"\", \"iface_in\": 654, \"iface_out\": 659, \"ip_src\": \"REMOVED\", \"net_src\": \"REMOVED\", \"ip_dst\":
\"REMOVED\", \"net_dst\": \"REMOVED\", \"mask_src\": 32, \"mask_dst\": 16, \"port_src\": 2099, \"port_dst\":
55764, \"tcp_flags\": \"24\", \"ip_proto\": \"tcp\", \"tos\": 0, \"sampling_rate\": 1000, \"timestamp_start\":
\"2018-03-16 22:17:29.856000\", \"timestamp_end\": \"2018-03-16 22:17:29.856000\", \"timestamp_arrival\": \"2018-03-16
22:18:30.893573\", \"stamp_inserted\": \"2018-03-16 22:17:00\", \"stamp_updated\": \"2018-03-16 22:19:01\", \"packets\"
: 1, \"bytes\": 667, \"writer_id\": \"default_kafka/2971\"}"}
</pre>

However, the flow record information is still there.


##Solution

Directly from Logstash website.

Logstash is an open source data collection engine with real-time pipelining capabilities. Logstash can dynamically unify
data from disparate sources and normalize the data into destinations of your choice. Cleanse and democratize all your
data for diverse advanced downstream analytics and visualization use cases.

The idea here is to use logstash to translate from pmacct netflow/ipfix (json encoded) to pnda.io avro via a kafka
message bus.


###Putting things together

![High Level](https://github.com/jbotello7381/pmacct_to_pnda.io/blob/master/Diagram.PNG)

In order to do this, we need to populate the data defined by pnda avro schema using the json input message coming
from pmacct.

This is pnda avro schema:
<pre>
{  
  "namespace": "pnda.entity",
  "type": "record",
  "name": "event",
  "fields": [
     {"name": "timestamp", "type": "long"},
     {"name": "src",       "type": "string"},
     {"name":<b>"host_ip"</b>,   "type": "string"},
     {"name": "rawdata",   "type": "bytes"}
  ]
}
</pre>

Namespace, type and name are part of avro envelop, and we don’t need to prepare the data to match those fields.   
However, timestamp, src, host_ip and rawdata need to exist in order for logstash to transform the input to avro
encoding. 

PNDA.IO documentation is not reliable and we have experience this in different ways.  In this case is not exception.  

Using above schema, logstash keep exiting with the error “The datum” is not an example of the schema.

<pre>
[2018-03-26T06:43:07,617][ERROR][logstash.agent] Failed to execute action {:action=>LogStash::PipelineAction::
Stop/pipeline_id:main,:exception=>"Avro::IO::AvroTypeError", :message=>"The datum {\"src\"=>\"netflow\",\"host\"=>
\"ip-10-180-221-47\", \"timestamp\"=>1522046587325, \"rawdata\"=>\"{\\\"event_type\\\": \\\"purge\\\", 
\\\"as_src\\\": <<<removed>>, \\\"as_dst\\\":<<<removed>>>, \\\"peer_ip_src\\\": \\\"<<<removed>>>\\\", \\\"peer_ip_dst
\\\": \\\"\\\", \\\"iface_in\\\": 658, \\\"iface_out\\\": 656, \\\"ip_src\\\": \\\<<<removed>>>\\\", \\\"net_src\\\": 
\\\<<<removed>>>\\\", \\\"ip_dst\\\": \\\<<<removed>>>\\\", \\\"net_dst\\\": \\\<<<removed>>>\\\", \\\"mask_src\\\": 
16, \\\"mask_dst\\\": 24, \\\"port_src\\\": 64267, \\\"port_dst\\\": 5220, \\\"tcp_flags\\\": \\\"0\\\", \\\"ip_proto\\\
": \\\"udp\\\", \\\"tos\\\": 0, \\\"sampling_rate\\\": 1000, \\\"timestamp_start\\\": \\\"2018-03-25 20:36:09.600000\\\"
, \\\"timestamp_end\\\": \\\"2018-03-25 20:36:49.792000\\\", \\\"timestamp_arrival\\\": \\\"2018-03-25 20:37:10.919446\\
\", \\\"stamp_inserted\\\": \\\"2018-03-25 20:36:00\\\", \\\"stamp_updated\\\": \\\"2018-03-25 20:37:11\\\", \\\"packets
\\\": 3, \\\"bytes\\\": 169, \\\"writer_id\\\": \\\"default_kafka/17164\\\"}\", \"@version\"=>\"1\", \"@timestamp\
"=>2018-03-26T06:43:07.325Z} <b>is not an example of schema</b> 
</pre>

Normally this problem can mean a couple of things.  Data type doesn’t match (bytes vs string, etc) or one of the names
inside the fields array.

If we look closer, host is what logstash generated doesn’t match what  pnda avro schema expect.

There are two ways to fix this, 
- Fix the schema to match what logstash set or
- Change the field or add it using logstash configuration (preferable).


###Option A
<pre>
{  
  "namespace": "pnda.entity",
  "type": "record",
  "name": "event",
  "fields": [
     {"name": "timestamp", "type": "long"},
     {"name": "src",       "type": "string"},
     {"name": "host",   "type": "string"},
     {"name": "rawdata",   "type": "bytes"}
  ]
}
</pre>

Here we changed the field "name": "host_ip" to "host".  Please be aware making this change may break something on the
PNDA side.  This is why this is not preferable.


####Logstash Configuration
<pre>
# Working configuration with avro output and kafka input
input {
  kafka {
    bootstrap_servers => "localhost:9092" # point to the kafka instance
    topics => "ipfix_json"
    # Used by pnda in as one way to identify the data
    add_field => [ "src", "netflow" ]
    # This field is normally inserted by logstash, however when using kafka as input plugin, 
    # the field was not present, so we manually insert it to comply with the avro schema
    add_field => [ "host", "ip_10.180.221.190" ]
  }
}
filter {
   mutate {
      # Put the content of the message to the PNDA Avro 'rawdata' field
      rename => { "message" => "rawdata" } 
      }
   ruby {
      # Put @timestamp (logstash reference) to timestamp field
      # pnda.io documentation is wrong in this section.  We have to fix it since
      # pnda example generate a floating number instead of a long integer making
      # logstash to complain and exit with not matching schema error
      code => "event.set('timestamp', (event.get('@timestamp').to_f * 1000).to_i)"
    }
}
output {
    kafka {
      bootstrap_servers => "10.180.221.130:9092" # change the broker list IPs
      topic_id => "netflow_json"
      compression_type => "none" # "none", "gzip", "snappy", "lz4"
      value_serializer => 'org.apache.kafka.common.serialization.ByteArraySerializer'
      codec => pnda-avro { schema_uri => "/home/ubuntu/logstash-5.3.3/pmacct.avsc" }
    }
}
</pre>


###Option B (Preferable)
Translate or add host_ip field

<pre>
# Working configuration with avro output and kafka input
input {
  kafka {
    bootstrap_servers => "localhost:9092" # point to the kafka instance
    topics => "ipfix_json"
    # Used by pnda in as one way to identify the data
    add_field => [ "src", "netflow" ]
    # This field is normally inserted by logstash, however when using kafka as input plugin, 
    # the field was not present, so we manually insert it to comply with the avro schema
    add_field => [ "host_ip", "ip_10.180.221.190" ]
  }
}
filter {
   mutate {
      # Put the content of the message to the PNDA Avro 'rawdata' field
      rename => { "message" => "rawdata" } 
      # if host is already inserted by logstash we can rename it
      # rename => { “host” => “host_ip” }
      }
   ruby {
      # Put @timestamp (logstash reference) to timestamp field
      # pnda.io documentation is wrong in this section.  We have to fix it since
      # pnda example generate a floating number instead of a long integer making
      # logstash to complain and exit with not matching schema error
      code => "event.set('timestamp', (event.get('@timestamp').to_f * 1000).to_i)"
    }
}
output {
    kafka {
      bootstrap_servers => "10.180.221.130:9092" # change the broker list IPs
      topic_id => "netflow_json"
      compression_type => "none" # "none", "gzip", "snappy", "lz4"
      value_serializer => 'org.apache.kafka.common.serialization.ByteArraySerializer'
      codec => pnda-avro { schema_uri => "/home/ubuntu/logstash-5.3.3/pmacct.avsc" }
    }
}
</pre>

Validation via a kafka consumer

<pre>
jbotello@Fa7h3rWin10:~/$ python consumer.py -s true
avro serialization required
start Consumer
on topic netflow_json
------------------------------------------------------------
ConsumerRecord(topic=u'netflow_json', partition=0, offset=1855861, timestamp=1522042861193, timestamp_type=0, key=None,
value='\xfc\xd9\x97\x8d\xccX\x0enetflow"ip_10.180.221.190\xd0\n{"event_type": "purge", "as_src": <<<removed>>>, 
"as_dst": <<<removed>>>, "peer_ip_src": <<<removed>>>, "peer_ip_dst": "", "iface_in": 654, "iface_out": 670, "ip_src": 
<<<removed>>>, "net_src": <<<removed>>>, "ip_dst": <<<removed>>>, "net_dst": <<<removed>>>, "mask_src": 24, "mask_dst":
17, "port_src": 5242, "port_dst": 54764, "tcp_flags": "0", "ip_proto": "udp", "tos": 0, "sampling_rate": 1000, 
"timestamp_start": "2018-03-26 05:39:58.592000", "timestamp_end": "2018-03-26 05:40:36.992000", "timestamp_arrival": 
"2018-03-26 05:40:59.288540", "stamp_inserted": "2018-03-26 05:39:00", "stamp_updated": "2018-03-26 05:41:01", 
"packets": 2, "bytes": 193, "writer_id": "default_kafka/24194"}', checksum=-72371146, serialized_key_size=-1, 
serialized_value_size=714) {u'timestamp': 1522042861182, u'host': u'ip_10.180.221.190', u'rawdata': '{"event_type":
"purge", "as_src": <<<removed>>>, "as_dst": <<<removed>>>, "peer_ip_src": <<<removed>>>, "peer_ip_dst": "", "iface_in":
654, "iface_out": 670, "ip_src": <<<removed>>>, "net_src": <<<removed>>>, "ip_dst": <<<removed>>>, "net_dst":
<<<removed>>>, "mask_src": 24, "mask_dst": 17, "port_src": 5242, "port_dst": 54764, "tcp_flags": "0", "ip_proto": 
"udp", "tos": 0, "sampling_rate": 1000, "timestamp_start": "2018-03-26 05:39:58.592000", "timestamp_end": "2018-03-26
05:40:36.992000", "timestamp_arrival": "2018-03-26 05:40:59.288540", "stamp_inserted": "2018-03-26 05:39:00", 
"stamp_updated": "2018-03-26 05:41:01", "packets": 2, "bytes": 193, "writer_id": "default_kafka/24194"}', u'src': 
u'netflow'}
</pre>


##How to setup a simple test environment
Best way to validate the complete workflow, we can perform the following task


###Collect a sample of the data sent by pmacct

<pre>
ubuntu@ip-10-180-221-190:~/pmacct$ nfacctd -f netflow_kafka.conf -d
...snip…
DEBUG ( default_kafka/kafka ): {"event_type": "purge", "as_src": <<<removed>>>, "as_dst": <<<removed>>>, "peer_ip_src":
<<<removed>>>, "peer_ip_dst": "", "iface_in": 658, "iface_out": 656, "ip_src": <<<removed>>>, "net_src": <<<removed>>>,
"ip_dst": <<<removed>>>, "net_dst": <<<removed>>>, "mask_src": 14, "mask_dst": 24, "port_src": 4532, "port_dst": 5100,
"tcp_flags": "0", "ip_proto": "udp", "tos": 0, "sampling_rate": 1000, "timestamp_start": "2018-03-26 07:06:54.80000",
"timestamp_end": "2018-03-26 07:07:35.552000", "timestamp_arrival": "2018-03-26 07:07:55.63327", "stamp_inserted":
"2018-03-26 07:06:00", "stamp_updated": "2018-03-26 07:08:01", "packets": 3, "bytes": 220, "writer_id": 
"default_kafka/25167"}
</pre>

Save content of the sample (only the json part) into a file

Use the sample as input to logstash test configuration (**test_v1.conf**)

<pre>
input {
  stdin {
    add_field => [ "src", "netflow" ]
  }
}

filter {
   ruby {
      code => "event.set('timestamp', (event.get('@timestamp').to_f * 1000).to_i)"
    }
   mutate {
      # Put the content of the message to the PNDA Avro 'rawdata' field
      rename => { "message" => "rawdata" } 
      }
}

output {
  stdout {
     codec => rubydebug
     }
}
</pre>

Verify the output and tweek the test configuration as need it. 

<pre>
ubuntu@ip-10-180-221-47:~/logstash-6.2.3$ bin/logstash -f test_v1.conf < input.json
..snip...
{
    "@timestamp" => 2018-03-26T06:53:05.996Z,
          "host" => "ip-10-180-221-47",
       "rawdata" => "{\"event_type\": \"purge\", \"as_src\": REMOVED, \"as_dst\": REMOVED, \"peer_ip_src\": 
       \"REMOVED\", \"peer_ip_dst\": \"\", \"iface_in\": 658, \"iface_out\": 656, \"ip_src\": \"REMOVED\", 
       \"net_src\": \"REMOVED\", \"ip_dst\": \"REMOVED\", \"net_dst\": \"REMOVED\", \"mask_src\": 16, 
       \"mask_dst\": 24, \"port_src\": 64267, \"port_dst\": 5220, \"tcp_flags\": \"0\", \"ip_proto\": \"udp\",
       \"tos\": 0, \"sampling_rate\": 1000, \"timestamp_start\": \"2018-03-25 20:36:09.600000\", 
       \"timestamp_end\": \"2018-03-25 20:36:49.792000\", \"timestamp_arrival\": \"2018-03-25 20:37:10.919446\",
       \"stamp_inserted\": \"2018-03-25 20:36:00\", \"stamp_updated\": \"2018-03-25 20:37:11\", \"packets\": 3, 
       \"bytes\": 169, \"writer_id\": \"default_kafka/17164\"}",
     "timestamp" => 1522047185996,
      "@version" => "1",
           "src" => "netflow"
}
[2018-03-26T07:11:38,472][INFO ][logstash.pipeline] Pipeline has terminated {:pipeline_id=>"main", 
:thread=>"#<Thread:0x42eb2617 run>"}
</pre>

The above configuration take the json input (input.json) and use the configuration (test_v1.conf) and send the output 
of rubydebug codec (very useful) to stdout.

***Notice “host” field, which didn’t match pnda.io documented avro schema.***   

We can also test using avro or pnda-avro codec as output with the following configuration

<pre>
input {
  stdin {
    add_field => [ "src", "netflow" ]
  }
}
filter {
   mutate {
      rename => { "message" => "rawdata" }       }
   ruby {
      # Put the content of the message to the PNDA Avro 'rawdata' field
      code => "event.set('timestamp', (event.get('@timestamp').to_f * 1000).to_i)"
    }
}
output {
  stdout {
           # codec => pnda-avro {       
           codec => avro {
            schema_uri => "/home/ubuntu/logstash-6.2.3/pmacct.avsc"
           # base64_encoding => false
        }
     }
}
</pre>

<pre>
ubuntu@ip-10-180-221-47:~/logstash-6.2.3$ bin/logstash -f test_v3.conf < input.json
...snip...
<b>
uKawk8xYDm5ldGZsb3cgaXAtMTAtMTgwLTIyMS00N8wKeyJldmVudF90eXBlIjoasdfasfHSDF$NyYyI6IDIwOSwgImFzX2RzdCI6IDY1MDcsICJwZWVyX2
lwX3NyYyI6ICIxMDQuMTYwLjEyOC4xIiwgInBlZXJfaXBfZHN0IjogIiIsICJpZmFjZV9pbiI6IDY1OCwgImlmYWNlX291d2RzdCI6ICIxOTIuNjQuMTczL
jAiLCAibWFza19zcmMiOiAxNiwgIm1hc2tfZHN0IjogMjQsICJwb3J0X3NyYyI6IDY0MjY3LCAicG9ydF9kc3QiOiA1MjIwLCAidGNwX2ZsYWdzIjogIjAi
LCAiaXBfcHJvdG8iOiAidWRwIiwgInRvcyI6IDAsICJzYW1wbGluZ19yYXRlIjogMTAwMCwgInRpbWVzdGFtcF9zdGFydCI6ICIyMDE4LTAzLTI1IDIwOjM
2OjA5LjYwMDAwMCIsICJ0aW1lc3RhbXBfZW5kIjogIjIwMTgtMDMtMjUgMjA6MzY6NDkuNzkyMDAwIiwgInRpbWVzdGFtcF9hcnJpdmFsIjogLTI1IDIwOj
M3OjExIiwgInBhY2tldHMiOiAzLCAiYnl0ZXMiOiAxNjksICJ3cml0ZXJfaWQiOiAiZGVmYXVsdF9rYWZrYS8xNzE2NCJ9
</b>
[2018-03-26T07:29:14,455][INFO ][logstash.pipeline] Pipeline has terminated {:pipeline_id=>"main", 
:thread=>"#<Thread:0x74899669 run>"}
</pre>

I haven’t try it yet, however in theory we could decode the avro message using avro tools available here and using the
pnda data preparation guide


##Configuration Files


###PMACCT Netflow producer
https://github.com/pmacct/pmacct.git

<pre>
ubuntu@ip-10-180-221-47:~/pmacct$ more netflow_kafka.conf 
! ..
plugins: kafka
!
aggregate: src_host, dst_host, src_port, dst_port, proto, tos, src_as, dst_as, peer_src_ip, peer_dst_ip, in_iface, out_iface, src_net, dst_net, src_mask, dst_mask, tcpflags, sampling_rate, timestamp_start, timestamp_end, timestamp_arrival
!
nfacctd_port: 2055
nfacctd_ip: 10.180.222.225
!
kafka_output: json
kafka_topic: ipfix_json
kafka_refresh_time: 60
kafka_history: 1m
kafka_history_roundoff: m
!kafka_broker_host: 10.180.221.152
kafka_broker_port: 9092
</pre>

###Logstash

https://artifacts.elastic.co/downloads/logstash/logstash-5.3.3.tar.gz
https://kafka.apache.org/quickstart

<pre>
# Working configuration with avro output and kafka input
input {
  kafka {
    bootstrap_servers => "localhost:9092" # point to the kafka instance
    topics => "ipfix_json"
    # Used by pnda in as one way to identify the data
    add_field => [ "src", "netflow" ]
    # This field is normally inserted by logstash, however when using kafka as input plugin, 
    # the field was not present, so we manually insert it to comply with the avro schema
    add_field => [ "host_ip", "ip_10.180.221.190" ]
  }
}
filter {
   mutate {
      # Put the content of the message to the PNDA Avro 'rawdata' field
      rename => { "message" => "rawdata" } 
      # if host is already inserted by logstash we can rename it
      # rename => { “host” => “host_ip” }
      }
   ruby {
      # Put @timestamp (logstash reference) to timestamp field
      # pnda.io documentation is wrong in this section.  We have to fix it since
      # pnda example generate a floating number instead of a long integer making
      # logstash to complain and exit with not matching schema error
      code => "event.set('timestamp', (event.get('@timestamp').to_f * 1000).to_i)"
    }
}
output {
    kafka {
      bootstrap_servers => "10.180.221.130:9092" # change the broker list IPs
      topic_id => "netflow_json"
      compression_type => "none" # "none", "gzip", "snappy", "lz4"
      value_serializer => 'org.apache.kafka.common.serialization.ByteArraySerializer'
      codec => pnda-avro { schema_uri => "/home/ubuntu/logstash-5.3.3/pmacct.avsc" }
    }
}
</pre>

##Reference

http://www.pmacct.net/

http://pnda.io

https://www.elastic.co/products/logstash

https://avro.apache.org/

https://kafka.apache.org/



