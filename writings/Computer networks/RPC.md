---
Title: Understanding RPC
date: 2026-07-19
---
Programs are build with functions. When people started build computer networks they eventually found a way tried to run programs through them.
Why not call functions over the network ryt. Also make the transition seemless so those remote calls feels native to the caller. This is the start for Remote Procedual Call.

>Then when computer nextworks start to grow and become the internet, servers needed be open to clients and not tightly coupled to them, so Representative State Transfer Protocal was born. These are universally agreed upon patterns that are used in networks to communicate and are implemented through serveral transport mediums (mostly RPC, REST only works with TCP versions). 

<iframe title="RPC from scratch in C" src="https://www.youtube.com/embed/PIqHAythNO4?feature=oembed" height="113" width="200" allowfullscreen="" allow="fullscreen" style="aspect-ratio: 1.76991 / 1; width: 40%; height: 40%;"></iframe>

| Part                  | Timestamp | Description                                          |
| :-------------------- | :-------: | :--------------------------------------------------- |
| Serialization         |   02:31   | Converting complex data structures into byte streams |
| Server Implementation |   19:11   | Setting up state, mutexes, and request endpoints     |
| Client Implementation |   44:55   | Connecting and executing remote procedure calls      |
### GPT roadmap to build a RPC 
A simple request response RPC connection to connect C backend to a typescript tui frontend.
#qustion can we use gRPC

> How do we hide the network part in RPC
#### Transport medium
Common choices are TCP sockets, Unix sockets, HTTP, WebSockets.

> TCP is apperently best for learning.
#### Message format
When we pass bytes to each other there should be a format for the computers in the network to read those bytes and get the correct info. yk to make sense of the big data blob. You can make your own or use something like JSON for this. Also we should decide the schema for this protocal. This is called a contract.

- what messages can be sent
- what they look like
- what methods exist
- what data types are expected
- what responses will come back

> gRPC uses protocoal buffers (protobuffs) 

A real RPC framework is mostly a system with defined format that encords and decodes data and handle errors to transfer data on the network.

| Style | Medium Type                 | Commonly Used In                      | Advantages                                                            | Disadvantages                                                         |
| ----- | --------------------------- | ------------------------------------- | --------------------------------------------------------------------- | --------------------------------------------------------------------- |
| RPC   | TCP                         | Custom binary RPC, internal services  | Fast, low overhead, full control over protocol                        | Must implement protocol, serialization, authentication, etc. yourself |
| RPC   | HTTP/1.1                    | JSON-RPC, XML-RPC                     | Easy to debug, widely supported, firewall-friendly                    | Higher overhead than binary protocols, no multiplexing                |
| RPC   | HTTP/2                      | gRPC and modern RPC frameworks        | Multiplexing, streaming, efficient binary transport, high performance | More complex, not human-readable, requires HTTP/2 support             |
| RPC   | Unix Domain Socket          | Local IPC (same machine)              | Very fast, lower latency than TCP, no network stack                   | Only works between processes on the same machine                      |
| RPC   | Message Queues              | RabbitMQ, ZeroMQ, Kafka request/reply | Asynchronous, resilient, decouples services                           | Higher latency, added infrastructure, more complex request tracking   |
| REST  | HTTP/1.1                    | Traditional web APIs                  | Simple, universal, cacheable, easy to test and debug                  | Higher overhead, one request per connection unless keep-alive is used |
| REST  | HTTP/2                      | Modern REST APIs                      | Multiplexing, header compression, improved performance                | More complex than HTTP/1.1, requires HTTP/2 support                   |
| REST  | HTTP/3 (QUIC)               | High-performance web APIs             | Faster connection setup, better performance on unreliable networks    | Newer technology, not universally supported                           |
| REST  | CoAP                        | IoT and embedded systems              | Lightweight, optimized for constrained devices                        | Limited ecosystem, not suitable for general web APIs                  |
| REST  | WebSockets (REST-like APIs) | Real-time applications                | Full-duplex communication, low latency                                | Not truly RESTful, stateful connection, more complex to manage        |

| Style            | Transport                | Data format      |
| ---------------- | ------------------------ | ---------------- |
| gRPC             | HTTP/2                   | Protocol Buffers |
| JSON-RPC         | HTTP/TCP                 | JSON             |
| XML-RPC          | HTTP                     | XML              |
| REST API         | HTTP/1.1, HTTP/2, HTTP/3 | JSON/XML         |
| Local RPC        | Unix socket              | Binary/JSON      |
| Microservice RPC | HTTP/2, TCP, queues      | Binary           |
#### Basic RPC server  architecture

![[Network Coms-1784464414719.webp]]

```
start program

create socket

bind to port 8080

listen()

wait for client

accept connection

while connected:

    read request

    parse request

    find function

    execute function

    send response
```

```
while(true)
{
    request = receive();

    if request.method == "add"
        result = add();

    send(result);
}
```

#### Tui client

```
start

connect to localhost:8080

send request

wait for response

display result
```

Example flow:

User presses:

```
5 + 3
```

OpenTUI receives input.

Your TS code creates:

```
{
 method:"add",
 args:[5,3]
}
```

Send it.

Wait.

Receive:

```
8
```

Render:

```
Result: 8
```

---

#### RPC function router

Your C server needs a data struct that maps the functions....what?

#### App logic through RPC 

#### Serialization
Whats that?

Beginner:

* JSON

Advanced:

* MessagePack
* Protobuf
* FlatBuffers

#### Add message boundaries

Important TCP lesson:

TCP is a stream.

If you send:

```
hello
world
```

the receiver might get:

```
helloworld
```

or:

```
hel
lo
world
```

You need framing.

Common solution:

Prefix every message with size:

```
[20][{"method":"add"...}]
```

Meaning:

```
20 bytes coming
```

Then read exactly 20 bytes.

#### Handle errors

Your protocol needs errors.

Example:

Request:

```
{
 method:"divide",
 args:[5,0]
}
```

Response:

```
{
 error:"division by zero"
}
```

Do not just crash the server.

#### Async behavior

Later your backend might:

* download songs
* query databases
* play audio
* run long tasks

You don't want:

```
client
 |
 request
 |
 server freezes 10 seconds
 |
 response
```

Eventually add:

```
client
 |
 request id:123
 |
 server starts task
 |
 immediately returns

{
 status:"started",
 id:123
}

later:

{
 id:123,
 result:"done"
}
```

This is how real RPC systems handle jobs.

