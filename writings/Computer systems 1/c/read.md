 
The setup was simple: a small RPC server. The client connects, sends a request, the server reads it, then sends a response.


```c
while ((n = read(client, buf, 1024)) > 0) {
//The mistake was reading until `read()` returned `0`:
}
```

It seems logical: keep reading until there is nothing left. But `read()` returning `0` does **not** mean "the sender finished sending this message." It means **the connection was closed**.

TCP is just a byte stream. It does not know where your messages start or end. The kernel only knows:

* bytes are available to read
* the connection was closed

It does not know that the client sent a complete RPC request and is now waiting for a response.

What happens:

1. Client sends request.
2. Client waits for response but keeps the connection open.
3. Server reads the request.
4. Server calls `read()` again expecting `0`.
5. Nothing arrives, but the connection is still open.
6. `read()` blocks forever waiting for more data or a close.
7. Client waits forever for the response.

Both sides are stuck.

The lesson: `read()` cannot tell you when a message is complete. It only tells you when bytes arrive or when the connection closes.

If you want persistent connections, your protocol needs message boundaries. Common solutions:

* **Length prefixing**

  ```
  [message size][message data]
  ```

  The receiver reads the size first, then knows exactly how many bytes to read.

* **Delimiters**

  ```
  {"method":"add","a":1}\n
  ```

  The receiver reads until it finds the delimiter.

HTTP, RPC systems, and most network protocols solve this by defining their own message framing instead of relying on `read()` returning `0`.
