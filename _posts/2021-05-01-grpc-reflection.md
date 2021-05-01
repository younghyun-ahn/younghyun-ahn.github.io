---
layout: post
title: "gRPC Reflection & Evans CLI"
---

gRPC Clients to connect to Server to have a `.proto` file which defines the service.
This is fine for proeuctin(you definitely want to know the API definition in advance). For development, when you hanve a gRPC server you don't know, simetimes you wish you could ask the server:
> "What APIs do you Have" ?

Reflection for two reaseons
1. Having server `expose` which endpoints are available
2. Allow `command line interfaces` (CLI) to talk to our server without have preliminary `.proto` file

Check out the [gRPC Reflection](https://github.com/grpc/grpc-go/tree/master/reflection) for more info.

## Server

{% highlight go %}
import "google.golang.org/grpc/reflection"

s := grpc.NewServer()
pb.RegisterYourOwnServer(s, &server{})

// Register reflection service on gRPC server.
reflection.Register(s)

s.Serve(lis)
{% endhighlight %}

## Installation

### macOS

```
$ brew tap ktr0731/evans
$ brew install evans
```

### go get

```
$ go get github.com/ktr0731/evans
```

## Usage (REPL)

```bash
> evans
> evans -p 50051 -r
> show package
+---------+
| PACKAGE |
+---------+
| api     |
+---------+
> package api
api@172.0.0.1:50051> show service
+---------+----------------------+-----------------------------+----------------+
| SERVICE |         RPC          |         REQUESTTYPE         |  RESPONSETYPE  |
+---------+----------------------+-----------------------------+----------------+
| Example | Unary                | SimpleRequest               | SimpleResponse |
|         | UnaryMessage         | UnaryMessageRequest         | SimpleResponse |
|         | UnaryRepeated        | UnaryRepeatedRequest        | SimpleResponse |
|         | UnaryRepeatedMessage | UnaryRepeatedMessageRequest | SimpleResponse |
|         | UnaryRepeatedEnum    | UnaryRepeatedEnumRequest    | SimpleResponse |
|         | UnarySelf            | UnarySelfRequest            | SimpleResponse |
|         | UnaryMap             | UnaryMapRequest             | SimpleResponse |
|         | UnaryMapMessage      | UnaryMapMessageRequest      | SimpleResponse |
|         | UnaryOneof           | UnaryOneofRequest           | SimpleResponse |
|         | UnaryEnum            | UnaryEnumRequest            | SimpleResponse |
|         | UnaryBytes           | UnaryBytesRequest           | SimpleResponse |
|         | ClientStreaming      | SimpleRequest               | SimpleResponse |
|         | ServerStreaming      | SimpleRequest               | SimpleResponse |
|         | BidiStreaming        | SimpleRequest               | SimpleResponse |
+---------+----------------------+-----------------------------+----------------+
api@172.0.0.1:50051> show message
+-----------------------------+
|           MESSAGE           |
+-----------------------------+
| SimpleRequest               |
| SimpleResponse              |
| Name                        |
| UnaryMessageRequest         |
| UnaryRepeatedRequest        |
| UnaryRepeatedMessageRequest |
| UnaryRepeatedEnumRequest    |
| UnarySelfRequest            |
| Person                      |
| UnaryMapRequest             |
| UnaryMapMessageRequest      |
| UnaryOneofRequest           |
| UnaryEnumRequest            |
| UnaryBytesRequest           |
+-----------------------------+
api@172.0.0.1:50051> desc SimpleRequest
+-------+-------------+
| FIELD |    TYPE     |
+-------+-------------+
| name  | TYPE_STRING |
+-------+-------------+

# Call a gRPC
api@172.0.0.1:50051> service Example
api.Example@172.0.0.1:50051> call Unary
name (TYPE_STRING) => ktr
{
  "message": "hello, ktr"
}
```

## Real world service (reference)

Google uses gRPC for some of ti's most important cloud service:
* [Google Pub/Sub](https://github.com/googleapis/googleapis/blob/master/google/pubsub/v1/pubsub.proto)
* [Google Spanner](https://github.com/googleapis/googleapis/blob/master/google/spanner/v1/spanner.proto)

[grpc.io](https://grpc.io/)

[RESTful HTTP API into gRPC](https://github.com/grpc-ecosystem/grpc-gateway)