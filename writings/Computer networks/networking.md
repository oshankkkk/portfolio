
## The Basics

Two or more computers talking to each other is a network. A switch connects computers inside a local network. A router connects networks together and handles getting traffic out to the internet. A dialog router is basically a switch and a router combined in one box.

Ethernet cables are layer 1. MAC addresses and frames are layer 2. Routers are layer 3. The router replaces the private IP with a public one when sending traffic out.

## MAC Addresses

A MAC address (Media Access Control) is a unique 12-digit hexadecimal identifier burned into the NIC of a device. It is used as a physical address to reliably transport data in a local network, whether that's ethernet or wifi. Each NIC in your device has its own MAC address, and your device probably has multiple NICs, each with their own private IP address assigned by the router.

A switch takes packets and routes them to the MAC address of the node it's meant for. It stores the MAC addresses of all connected devices in a content-addressable memory (CAM) table.

### Why Both MAC and IP?

MAC addresses are globally unique numbers given by the manufacturer, just like IP addresses are unique. But there can only be one IP address per request, so you can't send two IPs. Because of that, the two addresses serve different roles: the MAC address identifies the device physically on the local wire, and the IP address identifies the logical destination. You need both to send an ARP frame.

### Reserved Addresses

The first IP in a subnet is the gateway address, which is the router's address. The last IP is the broadcast address, used to reach all devices on the local network at once. When a sender's MAC address is not known, the packet is sent to the broadcast address so all devices receive it.

## ARP (Address Resolution Protocol)

ARP is how a device gets the MAC address for a known IP. When you ping a device, it first sends an ARP request to get the MAC address, and then sends the actual packet. ARP is sent out to all devices connected to the switch using the broadcast MAC address `FF:FF:FF:FF:FF:FF`. The device that owns the target IP sends back a reply with its MAC address, while all other devices ignore it.

ARP is not a ping. Ping is ICMP and tests connectivity. ARP is specifically for resolving a MAC address from an IP.

ARP is basically the sender yelling on the local network: "who has IP address 192.168.x.x? Tell me your MAC address."

### What's Inside an ARP Request

- sender MAC address
- sender IP address
- target IP address
- target MAC address (left as empty / all zeroes, because that's literally what you're asking for)

When the router sends an ARP message to a server, it sends its own MAC address so the server knows where to reply.

## How Data Moves on a Local Network

Say computer A wants to send a message to computer C on the same local network. A has the destination IP but no MAC address. A sends an ARP request out to the broadcast MAC address `FF:FF:FF:FF:FF:FF`. Every computer on the network gets it, but only C, which owns that destination IP, sends an ARP reply back to A with its MAC address. Now A knows C's MAC and can build the frame.

The frame looks like this:

> Source MAC: A's MAC | Destination MAC: C's MAC

And inside that frame, the IP packet is:

> Source IP: A's IP | Destination IP: C's IP

The IP says who the message is logically for. The MAC says who gets this frame on this local wire right now. Two different jobs.

## How Data Moves to a Different Network

When A wants to reach a computer E somewhere outside the local network, the same problem shows up first. A has the destination IP but needs a MAC to build a frame. But ARP broadcasts don't travel across the internet, so ARPing for E's MAC directly won't work.

First A has to figure out whether E is even local. It does this using its own IP address and subnet mask. If A's IP is `192.168.8.153` and the subnet mask is `255.255.255.0`, then A's local network is `192.168.8.0/24`. That means any address from `192.168.8.1` to `192.168.8.254` is local. If E's IP is something like `8.8.8.8`, A knows that's not in its LAN, and it sends the packet to the default gateway instead.

So A sends an ARP request for the default gateway IP, which is usually `192.168.x.1`. The router replies with its MAC. Then A builds the frame like this:

> Source MAC: A's MAC | Destination MAC: Router's MAC

But inside that frame, the IP packet is still:

> Source IP: A's IP | Destination IP: E's IP

MAC addresses change hop by hop. IP addresses stay the same from start to end.

## How Routers Forward Packets

When the router gets the frame from A, it looks at the destination MAC, sees it's addressed to itself, and accepts the frame. Then it unwraps the ethernet part and looks at the IP packet inside. The source IP is A and the destination IP is E. The router knows this packet isn't for it, but it knows where to send it next.

It checks its routing table to figure out the next hop. A routing table looks roughly like this:

| Destination Network | Next Hop   |
|---------------------|------------|
| 192.168.1.0/24      | Local      |
| 10.0.0.0/8          | Router 2   |
| 172.16.0.0/12       | Router 5   |
| 0.0.0.0/0           | ISP Router |

The router checks which network E's IP belongs to and picks the best matching route. This is called longest prefix match. Once it knows the next hop IP, it has the exact same local problem again: it knows the destination IP it wants to move the packet toward, but on this next local link it needs a MAC address. So the router ARPs for the next hop's MAC, then creates a brand new ethernet frame for that link and sends the packet out.

The router does not edit the old frame and forward it. It removes the old layer 2 frame entirely and creates a fresh one for every hop. The IP packet inside stays untouched the whole time. That is why MAC addresses are a local delivery system, not a global one.

Routing protocols like OSPF, RIP, EIGRP, and BGP are how routers share network topology and best paths with each other, which is how routing tables get built and stay up to date.

### The Full Hop by Hop

Say A wants to send to E and the path goes through home router R1, then ISP router R2, then R3, then R4, then finally E.

A builds a frame with destination MAC = R1's MAC and sends it. R1 receives it, strips the frame, looks at the IP packet, checks its routing table, ARPs for R2's MAC, builds a new frame with destination MAC = R2's MAC, and forwards the packet. R2 does the exact same thing toward R3. R3 to R4. Then R4 is finally on E's local network, so it ARPs for E's MAC if it doesn't have it already, then sends the frame directly to E.

The destination IP has been E the entire time. The destination MAC changed at every single hop.

## DNS

How does a computer get the destination IP in the first place? Still to cover.

## OSI Model

| Layer | Name         |
|-------|--------------|
| 7     | Application  |
| 6     | Presentation |
| 5     | Session      |
| 4     | Transport    |
| 3     | Network      |
| 2     | Data Link    |
| 1     | Physical     |

## TCP/IP Model

| Layer       | PDU     |
|-------------|---------|
| Application |         |
| Transport   | Segment |
| Network     | Packet  |
| Data Link   | Frame   |
| Physical    |         |

## Common Ports

- 80 — HTTP
- 443 — HTTPS
- 22 — SSH
- 5432 — PostgreSQL
- 25 — SMTP

## Still To Cover

- DNS
- UDP
- Switch vs Router
- WAP (Wireless Access Point)
- Upper OSI layers — Application, Presentation, Session jobs
- Routing protocols in depth