---
layout: post
title: Kafka producer and consumer
tags: python
---

Example of a kafka producer and consumer. Producers are used when you want to add data into kafka message bus on a specific topic and consumers when you want to read data on a specific topic 

### Kafka Producer
```
import json
import random
import string
#import ipaddress
import logging
import time

from kafka import KafkaProducer
from kafka import KafkaConsumer
from kafka.errors import KafkaError

producer = KafkaProducer(bootstrap_servers='10.143.64.140:59092', value_serializer=lambda v: json.dumps(v).encode('utf-8'))
topicName = 'ard'
#producer = KafkaProducer(value_serializer=lambda v: json.dumps(v).encode('utf-8'))

def main():
    # Dummy Bgp Data
    bgpData = {
    	"bgpGroup": " ",
    	"bgpNeighbor": " ",
            "bgpAsn": " ",
            "bgpNlri": " ",
            "bgpComm": " "
    }

    bgp_group = ["group1", "group2", "group3", "group4", "group5"]
    bgp_comm_start = "100:"
    bgp_nlri = ["inet-vpn-unicast", "inet-unicast", "inet6-unicast", "inet6-vpn-unicast", "evpn", "inet-flow", "inet6-flow"]

    # Generate Random data
    for i in range(0,10):
        bgpData["bgpGroup"] = random.choice(bgp_group)
        bgpData["bgpNeighbor"] = "101.101."+str(i)+".1/24"
        bgpData["bgpAsn"] = random.randint(64512,65534)
        bgpData["bgpNlri"] = random.choice(bgp_nlri)
        bgpData["bgpComm"] = bgp_comm_start+str(i)
        print(bgpData)
        producer.send(topicName, bgpData)

    producer.close()

if __name__ == "__main__":
    main()
```

### Kafka Consumer
```
import kafka
import json
import concurrent.futures
from kafka import KafkaConsumer
from kafka import KafkaProducer

KAFKA_BOOTSTRAP_SERVERS_CONS = '10.143.64.140:59092'
kafka_topics = ["ard"]

def thread_kafka_topic_handler(message):
    if (message.topic == 'ard'):
        print(json.loads(message.value))


def main():
    kafka_connected = False
    while True:
        if kafka_connected == False:
            try:
                consumer = KafkaConsumer(*kafka_topics,
                        max_poll_records=100000,
                        bootstrap_servers=KAFKA_BOOTSTRAP_SERVERS_CONS)
                print("kafka connected")
                kafka_connected = True
            except:
                print("Kafka not conneccted to: ", KAFKA_BOOTSTRAP_SERVERS_CONS)
                kafka_connected = False

        with concurrent.futures.ThreadPoolExecutor(max_workers=1) as executor:
            for message in consumer:
                executor.submit(thread_kafka_topic_handler, message=message,)


if __name__ == "__main__":
    main()
```
