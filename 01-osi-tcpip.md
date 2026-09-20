# OSI and TCP/IP — In My Own Words

The OSI (Open Systems Interconnection) model divides network communication into seven layers. Each layer is responsible for a different part of getting data from one device to another.

## The 7 OSI Layers

### 1. Physical

The actual physical transmission of data. This includes things like Ethernet cables, connectors, radio signals, and the hardware used to move bits between devices.

_Security view:_ 
	An attacker could physically tamper with or disconnect network equipment, tap a cable, or jam a wireless signal.
	A defender can reduce these risks by restricting physical access to network equipment, securing cables and network cabinets, and monitoring important hardware for tampering or outages.

### 2. Data Link

Communication between devices on the same local network.

This is where MAC addresses are used. Network Interface Cards (NICs) have MAC addresses associated with them, and switches mainly operate at this layer.

_Security view:_ 
	An attacker could use techniques such as ARP spoofing to impersonate another device on the local network and intercept traffic.
	A defender can use protections such as switch port security, network segmentation, and Dynamic ARP Inspection to make local-network impersonation and interception more difficult.

### 3. Network

Responsible for moving packets between different networks and deciding where they should go. IP addresses belong here, as do routing protocols such as OSPF and RIP.

Routers mainly operate at Layer 3.

_Security view:_ 
	An attacker could spoof IP addresses or manipulate/reroute network traffic.
	A defender can use firewall and access-control rules, anti-spoofing filters, and secure routing configurations to restrict unwanted traffic and reduce the risk of traffic being impersonated/rerouted.

### 4. Transport

Responsible for communication between applications on different hosts. The two main protocols at this layer are TCP and UDP.

TCP prioritizes reliable delivery over speed. Packets are tracked and missing data can be retransmitted.

UDP has less overhead and does not wait for acknowledgements or retransmissions, which makes it useful when speed/low latency matters more than perfect delivery. For example, a Zoom call generally cares a lot about receiving current information quickly. 

_Security view:_
	An attacker could abuse TCP connection handling, for example with a SYN flood that leaves many half-open connections and exhausts resources.
	A defender can use measures such as rate limiting, SYN cookies, connection limits, and stateful firewalls to reduce the impact of attacks that abuse TCP or UDP communication.

### 5. Session

Creates, maintains, and ends connections ("sessions") between applications. Basically keeping track of an ongoing conversation between two systems.

_Security view_: 
	An attacker could hijack an established session and act as if they were the legitimate user.
	A defender can protect sessions by using unpredictable session identifiers, expiring inactive sessions, invalidating sessions after logout, and asking users to authenticate again before sensitive actions.

### 6. Presentation

Deals with how data is represented so that the application can use it. This can include translating between data formats, encoding/decoding, compression, and encryption/decryption.

_Security view:_ 
	An attacker could exploit weaknesses in how data is encoded or decoded, for example by sending a malformed image or document that triggers a bug in a parser, or by taking advantage of weak encryption settings.
	A defender can reduce this risk by keeping parsing and cryptographic libraries updated, validating input formats, and disabling outdated encryption protocols.

### 7. Application

The layer closest to the user and the applications we actually interact with. Protocols such as HTTP, DNS, SMTP, and FTP are commonly associated with this layer.

_Security view:_ 
	This is where many attacks against user-facing services happen, such as SQL injection, cross-site scripting (XSS), or malicious HTTP requests.
	These risks can be reduced through practices such as validating input, using parameterized queries, encoding output correctly, keeping applications patched, and limiting what requests the application accepts.

## TCP/IP vs OSI

The OSI model has seven layers, but real-world networking is more commonly described using the TCP/IP model, which I've liked better since CS408.

The TCP/IP model combines several OSI layers:

- **Application** --> OSI Application + Presentation + Session
- **Transport** --> OSI Transport
- **Internet** --> OSI Network
- **Network Access** --> OSI Data Link + Physical
