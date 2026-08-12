So [[serialization]] happens before anything. After that to actually send it through the wire, you call something like `write()` or `send()` on a socket file descriptor. This is a syscall. its encapsulated inside the kernal, and the kernel copies your bytes out of your process's memory into a kernel owned socket send buffer.
##### This changes based on the socket type
- IP/TCP
- Unix domain socket
###### TCP/IP
This is basically how TCP works

The kernel's TCP layer breaks your byte stream into segments and wraps each one in a TCP header  source port, destination port, sequence number, ack number, checksum, window size. TCP is a stream protocol, so it doesn't preserve your original message boundaries; it just guarantees the bytes arrive in order, eventually, or the connection errors out. This is exactly why your plan step 4 ("define your message format") matters — since TCP doesn't preserve message boundaries, you need something like a length-prefix or a delimiter so the receiving side knows where one RPC message ends and the next begins.

Each TCP segment then gets wrapped in an IP packet source IP, destination IP, TTL, protocol number, its own checksum. IP is responsible for routing, not delivery guarantees. The IP packet then gets wrapped in a link-layer frame — for Ethernet, that means source MAC, destination MAC, an Ethertype field, and a CRC. This is the layer that actually knows how to put bits on a wire (or radio waves, for Wi-Fi).

The NIC (network interface card) takes that frame and physically transmits it voltage transitions on copper, light pulses on fiber, or modulated radio on wireless. Bit by bit.
###### Unix Domain Sockets
With a Unix domain socket, there's no actual "network" involved at all. The kernel just moves bytes directly from one process's socket buffer to another process's socket buffer, entirely in memory. No packets, no headers, no addresses, no checksums, no physical transmission. It's essentially a fast, kernel-mediated pipe between two processes on the same machine. This is why Unix domain sockets are so much cheaper than TCP loopback — you skip the entire networking stack below the socket layer.
#### How TCP works
Https server takes bytes from tcp connections parses and makes the response according to the RFC. How does TCP works. It uses serialization.
#### TCP vs Unix domain sockets
We can use TCP for interprocess communication, we open 2 tcp sockets and talk, we get network overhead? why?
Why dont we get that from unix domain sockets. Why do they seem to replace each other. why is TCP a standard in learning network projects in scratch.
