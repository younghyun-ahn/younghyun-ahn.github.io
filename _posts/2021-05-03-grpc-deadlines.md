---
layout: post
title: "gRPC Deadlines"
---

Deadlines allow gRPC lcients to specify how long they ar willing to wait for and RPC to complete before the RPC is terminated with error DEADLINE_EXCEEDED. __The gRPC documentation recommends you set a deadline for all client RPC calls.__ Setting the deadline is up to you: how long do you feel your API should have to complete?

The server should check if the deadline has exceeded and cancel the work it is doing.

Note: Deadlines are propagated accross if gRPC calls are chained
* A => B => C (deadline for A is passed to B and thend passed to C)

## Server
```go
for i := 0; i < 3; i ++ {
	if ctx.Err() == context.Canceled {
		// the client canceled the request
		fmt.Println("The client canceled the request!")
		return nil, status.Error(codes.Canceled, "The client canceled the request")
	}
	time.Sleep(1* time.Second)
}
```

## Client
```go
ctx, cancel := context.WithTimeout(context.Background(), timeout)
defer cancel()

res, err := c.HelloWithDeadline(ctx, req)
if err != nil {
	statusErr, ok := status.FromError(err)
	if ok {
		if statusErr.Code() == codes.DeadlineExceeded {
			fmt.Println("Timeout was hit! Deadline was exceeded")
		} else {
			fmt.Printf("Unexpected error: %v", statusErr)
		}
	} else {
		log.Fatalf("Error while calling Hello RPC: %v", err)
	}
	return
}
log.Printf("Response from Hello: %v", res.Result)
```

## Console
```console
❯ go run hello/hello_server/server.go
Hello server
HelloWithDeadline function was invoked with hello:{first_name:"Younghyun" last_name:"Ahn"}
HelloWithDeadline function was invoked with hello:{first_name:"Younghyun" last_name:"Ahn"}

❯ go run hello/hello_client/client.go
Hello client
2021/05/03 22:46:56 Response from Hello: Hello Younghyun
Timeout was hit! Deadline was exceeded
```


Check out the [gRPC and Deadlines](https://grpc.io/blog/deadlines/) for more info.
