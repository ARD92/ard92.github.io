---
layout: post
title: Ingesting telemetry from Junos devices 
tags: linux kubernetes junos
---

Juniper platforms have various ways to gather statistics such as SNMP, telemetry, netconf. This blog talks about how one can leverage telemetry to subscribe to various sensors to gather information. Additionally you can pass the subscribed information on a message bus to process further. Either to modify/parse data and store it into the database. 

This gives an example on spinning up the below on kubernetes to achieve the same 
1. jtimon ( used for subscribing to telemetry sensor)
2. kafka (publish the subscribed msgs on kafka for further processing)
3. consumer app (app to process msgs on kafka and parse to store on db)
3. influxdb(timeseries db used to store data) 

The reason for using kafka is it is easily extensible if we need to add more features/sensors/vendor devices 

![architecture](/images/telemetry_ingest.png)

## Install 
Install kubernetes. you can find information [here](https://ard92.github.io/2022/04/23/setting-up-k8s-in-ubuntu.html) on how to bring up the cluster

The manifest files for jtimon, influx and kafka are present [here](https://github.com/ARD92/telemetry-ingest)

```
kubectl apply -f namespace.yaml

kubectl apply -f kafka/bw-zookeeper.yaml 
kubectl apply -f kafka/bw-kafka.yaml

kubectl apply -f influx/secrets.yaml
kubectl apply -f influx/influx-pvc.yaml
kubectl apply -f influx/influx-deploy.yaml
kubectl apply -f influx/influx-service.yaml

# ensure you copy subscribe.json to the respective PV path
kubectl apply -f jtimon/jtimon-pvc.yaml
kubectl apply -f jtimon/jtimon.yaml 
scp subscribe.json <path to pvc>
```

## Verify 

### Check if sensor is subscribed on vSRX
```
root@vsrx> show agent sensors

Sensor Information :

    Name                                    : sensor_1000
    Resource                                : /interfaces/interface[name='ge-0/0/0']/state/counters/
    Version                                 : 1.0
    Sensor-id                               : 539528115
    Subscription-ID                         : 1000
    Parent-Sensor-Name                      : Not applicable
    Component(s)                            : PFE,PFE,mib2d,xmlproxyd

    Profile Information :

        Name                                : export_1000
        Reporting-interval                  : 30
        Payload-size                        : 5000
        Format                              : GPB

```

### Check if data is being received on kafka

enter the k8s kafka pod 

```
root@kafka-6cf4f5656d-vbml7:/opt/kafka/bin# ./kafka-console-consumer.sh --bootstrap-server=localhost:9092 --topic bw-account --from-beginning


## output as below. snipped.. 

vsrx"Psensor_1000_1_1:/junos/system/linecard/interface/logical/usage/:/interfaces/:PFE0�����0:
__timestamp__8�����0:[

__prefix__RM/interfaces/interface[name='ge-0/0/0']/subinterfaces/subinterface[index='0']/:
	init-time8���:
state/counters/out-octets8��6:
state/counters/out-pkts8�P:
state/counters/in-octets8��@:
                             state/counters/in-pkts8�`:[

__prefix__RM/interfaces/interface[name='ge-0/0/1']/subinterfaces/subinterface[index='0']/:
	init-time8����:
state/counters/out-octets8��%:
state/counters/out-pkts8�9:
state/counters/in-octets8��:
```

## Debug tips 

### Jtimon config used 
This is used on the pvc. 
```
root@k8s-worker1:/opt/aprabh/jtimon/config# more subscribe.json
{
    "host": "192.167.1.4",
    "port": 32767,
    "user": "root",
    "password": "juniper123",
    "cid": "bw_account_payg",
    "kafka": {
        "brokers": ["10.85.47.165:31093"],
        "topic": "bw-account",
        "client-id": "bw-account-vsrx"
    },
    "paths": [{
        "path": "/interfaces",
        "freq": 30000
    }],
    "log": {
        "file": "log.txt",
        "periodic-stats": 0,
        "verbose": false
    }
}
```

### Stream to kafka topics 

#### kafka debug

```
root@kafka-6cf4f5656d-vbml7:/opt/kafka/bin# ./kafka-topics.sh --bootstrap-server=localhost:9092 --list
bw-account
```

#### Listing data on the topic
```
root@kafka-6cf4f5656d-vbml7:/opt/kafka/bin# ./kafka-console-consumer.sh --bootstrap-server=localhost:9092 --topic bw-account --from-beginning


## output as below. snipped.. 

vsrx"Psensor_1000_1_1:/junos/system/linecard/interface/logical/usage/:/interfaces/:PFE0�����0:
__timestamp__8�����0:[

__prefix__RM/interfaces/interface[name='ge-0/0/0']/subinterfaces/subinterface[index='0']/:
	init-time8���:
state/counters/out-octets8��6:
state/counters/out-pkts8�P:
state/counters/in-octets8��@:
                             state/counters/in-pkts8�`:[

__prefix__RM/interfaces/interface[name='ge-0/0/1']/subinterfaces/subinterface[index='0']/:
	init-time8����:
state/counters/out-octets8��%:
state/counters/out-pkts8�9:
state/counters/in-octets8��:
```

#### Store to Influx DB
```
root@influxdb-56b97dff7c-gfxch:/usr/local/bin# influx v1 shell
InfluxQL Shell 2.5.0
Connected to InfluxDB OSS v2.5.0
>
>
> SHOW DATABASES
> quit
```
