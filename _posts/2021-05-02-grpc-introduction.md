---
layout: post
title: "gRPC Introduction"
---

At the core of gRPC, you need to define the messages and services using `Protocol Buffers`. The rest of the gRPC code will be generated for you and you'll have to provide and implementatin for it. one `.proto` file works for over 12 programming languages (server and client), and allow you to use a framework that scales to millions of RPC per seconds.

* gRPC is a `free` and `open-source` framework developed by Google
* gRPC is part of the Cloud Native Computation Foundation (CNCF) - like Docker & Kubernetes for example
* At a high level, it allows you to define REQUEST and RESPONSE for RPC (Remote Procedure Calls) and handles all the rest for you
* On top of it, it's `modern`, `fast and efficient`, build on top of `HTTP/2`, `low latency`, supports `streaming`, `language independent`, and makes it supper easy to plug in `authentication`, `load balancing`, `logging` and `monitoring`.

## What's an RPC?

* It's not a new concept (CORBA had this berore). 
* With gRPC, it's implemented very cleanly and solves a lot of problems.

![screenshot](../../../assets/images/landing-2.svg)

## Protocol Buffers?

* gRPC use Protocol Buffers for communications.
* Let's measure the payload size vs JSON:

```go
// JSON: 55 bytes
{
	"age": 25,
	"first_name": "Younhghyun",
	"last_name": "Ahn"
}

// same in Protocol BUffers: 20 bytes
message Person {
	int32 age = 1;
	string first_name = 2;
	string last_name = 3;
}
```
We save in `Network Bandwidth`.

#### Efficiency of Protocol Buffers over JSON

* Parsing JSON is actually CPU intensive (because toe format is human readable)
* Parsing Protocol Buffers (binary format) is less CPU intensive because it's closer to how a machine represents data
* By using gRPC, the use of Protocol Buffers means `faster` and more `effcient` communication, friendly with mobile device that have a slower CPU

#### Summary: Why Protocol Buffers?

* Easy to write message definition
* The definition of the API is independent from the implementation
* A huge amount of code can be generated, in andy language, from a simpe .proto file
* The payload is binary, therefor very efficient to send / receive on a network and serialize / de-serializer on a CPU
* Protocol Buffers defines rules to make an API evolve without breaking existing clients, with is helpful for microservices

## Language Interoperability

gRPC can be used by any language. Because the code can be generated for any language, it makes it super simple to create micro-services in any language that interact with each other.

![screenshot](../../../assets/images/language_interoperability.png)

## 4 Types of API in gRPC

![screenshot](../../../assets/images/grpc_4type.png)

* Unary is what a traditional API looks like (HTTP REST)
* HTTP/2 enables APIs to now have streaming capabilities.
* The server and client can push multiple messages as part of one request!
* In gRPC it's very easy to define thes APIs as we'll see

## Scalability in gRPC

* gRPC Server are asynchronous by default
* This means they do not block threads on request
* Therefore each gRPC server can serve millions of requests in parallel

* gRPC Clients can be asynchronous or synchronous (blocking)
* The client decides which model works best for the performance nees
* gRPC Client can perform client side load balancing

## gRPC vs REST

| gRPC                                                                         | REST                                                                        |
|------------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| Protocol Buffers - smaller, faster                                           | JSON - text based, slower, bigger                                           |
| HTTP/2 (lower latency) - from 2015                                           | HTTP1.1 (high latency) - from 1997                                          |
| Bidirectional & Async                                                        | Client => Server requests only                                              |
| Stream Support                                                               | Request / Response support only                                             |
| API Oriented-"WHAT" (no constraints - free design)                           | CRUD Oriented (Create-Retrieve-Update-Delete / POST GET PUT DELETE)         |
| Code Generation through Protocol Buffers in any language - 1st class citizen | Code generation through OpenAPI / Swagger(add-on) - 2nd class citizen       |
| RPC Based - gRPC does the plumbing for us                                    | HTTP verbs based - we have to write the plumbing or use a 3rd party library |

## Summary: Why use gRPC

 * Easy code defiition in over 11 languages
 * Uses a mordern, low latency HTTP/2 transport mechanism
 * SSL Security is built in
 * Support for streaming APIs for maximum performance
 * gRPC is API oriented, insted of Resource Oriented like REST


Check out the [gRPC and gRPC Reflection](https://medium.com/ruangguru-engineering/grpc-and-grpc-reflection-8cc1e395adb3), [REST v. gRPC](https://husobee.github.io/golang/rest/grpc/2016/05/28/golang-rest-v-grpc.html) for more info.