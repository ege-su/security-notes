# OSI and TCP/IP — In My Own Words

The OSI (Open Systems Interconnection) model divides network communication into seven layers. Each layer is responsible for a different part of getting data from one device to another.

## The 7 OSI Layers

### 1. Physical

The actual physical transmission of data. This includes things like Ethernet cables, connectors, radio signals, and the hardware used to move bits between devices.

_Security view:_ An attacker could physically tamper with or disconnect network equipment, tap a cable, or jam a wireless signal.

### 2. Data Link

Communication between devices on the same local network.

This is where MAC addresses are used. Network Interface Cards (NICs) have MAC addresses associated with them, and switches mainly operate at this layer.

_Security view:_ An attacker could use techniques such as ARP spoofing to impersonate another device on the local network and intercept traffic.

### 3. Network

Responsible for moving packets between different networks and deciding where they should go. IP addresses belong here, as do routing protocols such as OSPF and RIP.

Routers mainly operate at Layer 3.

_Security view:_ An attacker could spoof IP addresses or manipulate/reroute network traffic.

### 4. Transport

Responsible for communication between applications on different hosts. The two main protocols at this layer are TCP and UDP.

TCP prioritizes reliable delivery over speed. Packets are tracked and missing data can be retransmitted.

UDP has less overhead and does not wait for acknowledgements or retransmissions, which makes it useful when speed/low latency matters more than perfect delivery. For example, a Zoom call generally cares a lot about receiving current information quickly. 

_Security view:_ An attacker could abuse TCP connection handling, for example with a SYN flood that leaves many half-open connections and exhausts resources.

### 5. Session

Creates, maintains, and ends connections ("sessions") between applications. Basically keeping track of an ongoing conversation between two systems.

_Security view_: An attacker could hijack an established session and act as if they were the legitimate user.

### 6. Presentation

Deals with how data is represented so that the application can use it. This can include translating between data formats, encoding/decoding, compression, and encryption/decryption.

_Security view_: An attacker could target the way data is encoded, encrypted, or decoded, including weaknesses in encryption or malformed data handling.

### 7. Application

The layer closest to the user and the applications we actually interact with. Protocols such as HTTP, DNS, SMTP, and FTP are commonly associated with this layer.

_Security view:_ This is where many attacks against user-facing services happen, such as SQL injection, cross-site scripting (XSS), or malicious HTTP requests.

## TCP/IP vs OSI

The OSI model has seven layers, but real-world networking is more commonly described using the TCP/IP model, which I've liked better since CS408.

The TCP/IP model combines several OSI layers:

- **Application** --> OSI Application + Presentation + Session
- **Transport** --> OSI Transport
- **Internet** --> OSI Network
- **Network Access** --> OSI Data Link + Physical
