# OSI and TCP/IP — In My Own Words

The OSI (Open Systems Interconnection) model divides network communication into seven layers. Each layer is responsible for a different part of getting data from one device to another.

## The 7 OSI Layers

### 1. Physical

The actual physical transmission of data. This includes things like Ethernet cables, connectors, radio signals, and the hardware used to move bits between devices.

### 2. Data Link

Communication between devices on the same local network.

This is where MAC addresses are used. Network Interface Cards (NICs) have MAC addresses associated with them, and switches mainly operate at this layer.

### 3. Network

Responsible for moving packets between different networks and deciding where they should go. IP addresses belong here, as do routing protocols such as OSPF and RIP.

Routers mainly operate at Layer 3.

### 4. Transport

Responsible for communication between applications on different hosts. The two main protocols at this layer are TCP and UDP.

TCP prioritizes reliable delivery over speed. Packets are tracked and missing data can be retransmitted.

UDP has less overhead and does not wait for acknowledgements or retransmissions, which makes it useful when speed/low latency matters more than perfect delivery. For example, a Zoom call generally cares a lot about receiving current information quickly. 

### 5. Session

Creates, maintains, and ends connections ("sessions") between applications. Basically keeping track of an ongoing conversation between two systems.

### 6. Presentation

Deals with how data is represented so that the application can use it. This can include translating between data formats, encoding/decoding, compression, and encryption/decryption.

### 7. Application

The layer closest to the user and the applications we actually interact with. Protocols such as HTTP, DNS, SMTP, and FTP are commonly associated with this layer.

## TCP/IP vs OSI

The OSI model has seven layers, but real-world networking is more commonly described using the TCP/IP model, which I've liked better since CS408.

The TCP/IP model combines several OSI layers:

- **Application** --> OSI Application + Presentation + Session
- **Transport** --> OSI Transport
- **Internet** --> OSI Network
- **Network Access** --> OSI Data Link + Physical