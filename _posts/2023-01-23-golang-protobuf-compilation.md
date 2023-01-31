---
layout: post
title: Compiling proto files in golang  
tags: linux
---

Follow the below steps in order to compile proto files in golang. 
Juniper has JET APIs which can be used to program BGP routes, firewall filters, any junos config over gRPC channels. The IDLs can be downloaded from Junipers download page. Follow the below steps to compile 


## Install gRPC protoc compiler 

use the below docker file 

```
FROM golang:1.19

RUN apt-get update && \
    apt-get install unzip

RUN curl -OL https://github.com/protocolbuffers/protobuf/releases/download/v3.9.1/protoc-3.9.1-linux-x86_64.zip && \
    unzip -o protoc-3.9.1-linux-x86_64.zip -d /usr/local bin/protoc && \
    unzip -o protoc-3.9.1-linux-x86_64.zip -d /usr/local include/* && \
    rm -rf protoc-3.9.1-linux-x86_64.zip

RUN go get -u github.com/golang/protobuf/protoc-gen-go && \
    go get -u github.com/gogo/protobuf/proto && \
    go get -u github.com/gogo/protobuf/gogoproto

COPY jet-idl-21.4R3.15 /app

WORKDIR "/app"

ENTRYPOINT ["/bin/bash"]
```

### Build the container using

```
docker build -t gojet:latest .
```

### Start the container 
```
docker run -itd --name gojet -v ${PWD}:/app gojet:latest
```

### Enter the container
```
docker exec -itd gojet bash
```

## mention the optional gopackage  in all the .proto files 

Below is a snippet of the Junos JET IDL. note that `option go_package` line. The path is relative path from root. The compiled proto file will be present in the mentioned directory. This directory would be automatically created in case it does not exist.
```
syntax = "proto3";
// [brief]: JET Routing Base Package.
package jnx.jet.routing.base;
option go_package = "/app/proto/jet/jnx/routing";
// [version]: 0.0.0
import "jnx_common_addr_types.proto";
// [version]: 0.0.0
import "jnx_common_base_types.proto";
```

## Compile
```
protoc --proto_path=<absolute path > <absolute path to .proto file> --go_out=plugins=grpc:. --plugin=protoc-gen-go=<path to protoc-gen-go>
```

Example 
```
protoc --proto_path=/app/proto/proto/ /app/proto/proto/jnx_routing_base_types.proto --go_out=plugins=grpc:. --plugin=protoc-gen-go=/go/bin/protoc-gen-go
```

## compile from the base followed by the ones hvaing dependencies 

For example compile jnx_common_base_types and jnx_common_addr before compiling jnx_authentication.

Example
```
protoc --proto_path=proto/ proto/jnx_routing_base_types.proto --go_out=plugins=grpc:. --plugin=protoc-gen-go=/go/bin/protoc-gen-go
protoc --proto_path=proto/ proto/jnx_routing_bgp_service.proto --go_out=plugins=grpc:. --plugin=protoc-gen-go=/go/bin/protoc-gen-go
protoc --proto_path=proto/ proto/jnx_authentication_service.proto --go_out=plugins=grpc:. --plugin=protoc-gen-go=/go/bin/protoc-gen-go
protoc --proto_path=proto/ proto/jnx_common_addr_types.proto --go_out=plugins=grpc:. --plugin=protoc-gen-go=/go/bin/protoc-gen-go
protoc --proto_path=proto/ proto/jnx_common_base_types.proto --go_out=plugins=grpc:. --plugin=protoc-gen-go=/go/bin/protoc-gen-go
protoc --proto_path=proto/ proto/jnx_management_service.proto --go_out=plugins=grpc:. --plugin=protoc-gen-go=/go/bin/protoc-gen-go
```  
