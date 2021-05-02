---
layout: post
title: "gRPC SSL Security"
---

gRPC has SSL/TLS integration and promotes the use of SSL/TLS to authenticate the server, and to encrypt all the data exchanged between the client and the server. Optional mechanisms are available for clients to provide certificates for mutual authentication.

## Certificate files by openssl

| ca.key     | Certificate Authority `private key` file (this shouldn't be shsare in real-life)                 |
| ca.crt     | Certificate Authority trust `certificate` (this should be share with user in real-life)          |
| server.key | Server private key, passwrod protected (this shouldn't b share)                                |
| server.csr | Server certificate signing request (thils should be shared with the CA owner)                  |
| server.crt | Server certificate signed by the CA (this would be sent back by the CA owner) - keep on server |
| server.pem | Conversion for server.key into a format gRRPC like (this shouldn't be share)                   |

### Caution

- Private files: ca.key, server.key, sever.pem, server.crt
- Share files: ca.crt (need by the client), server.csr (need by th CA)

## Openssl

{% highlight bash %}
# Changes these CN's to match your hosts in your envrioment if need.
SERVER_CN=localhost

# Step 1: Generate Certificate Authority + Trust Certificate (ca.crt)
openssl genrsa -passout pass:1111 -des3 -out ca.key 4096
openssl req -passin pass:1111 -new -x509 -days 365 -key ca.key -out ca.crt -subj "/CN=${SERVER_CN}"

# Step 2: Generate the Server Private Key (server.key)
openssl genrsa -passout pass:1111 -des3 -out server.key 4096

# Step 3: Get a certificate signing request from the CA (server.csr)
openssl req -passin pass:1111 -new -key server.key -out server.csr -subj "/CN=${SERVER_CN}"

# Step 4: Sign the certificate with the CA we created (it's called self signing) - server.crt
openssl x509 -req -passin pass:1111 -days 365 -in server.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out server.crt

# Step 5: Convert the server certificate to .pem format (server.pem) - usable by gRPC
openssl pkcs8 -topk8 -nocrypt -passin pass:1111 -in server.key -out server.pem
{% endhighlight %}

## Example Code

## Sever

{% highlight go %}
lis, err := net.Listen("tcp", "0.0.0.0:50051")
if err != nil {
	log.Fatalf("Faild to listen: %v", err)
}

certFile := "server.crt"
keyFile := "server.pem"
creds, err := credentials.newServerTLSFromFIle(certFile, keyFile)
if err != nil {
	log.Fatalf("Faild loading certificates: %v", err)
}
opts := grpc.Creds(creds)

s := grpc.NewServer(opts)

if err := s.Serve(lis); err != nil {
	log.Fatalf("faild to serve: %v", err)
}
{% endhighlight %}

### Client

{% highlight golang %}
certFile := "ca.cert" // Certification Authority Trust certificate
creds, err := credentials.newClientTLSFromFile(certFile, "")
if err != nil {
	log.Fatalf("Error while loading CA trust certificate: %v", err)
}
opts := grpc.WithTransportCredentials(creds)

conn, err := grpc.Dial("localhost:50051", opts)
if err != nil {
	log.Fatalf("could not connect: %v", err)
}
defer conn.Close()

c := pb.NewGreetClient(conn)
// ...
{% endhighlight %}


Check out the [gRPC Docs](https://grpc.io/docs/guides/auth/#go) for more info.