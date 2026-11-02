<p align="right"><sub>November 1, 2026</sub></p>

# Listening Ports on macOS: All Interfaces vs. Localhost

## Introduction

In this experiment, I used `lsof` and Python's built-in HTTP server to examine how binding a service to different network interfaces affects its accessibility. My goal was to compare a service listening on all IPv4 interfaces (`0.0.0.0`) with one restricted to the local loopback interface (`127.0.0.1`).

**Environment:** macOS, Terminal (zsh), Python 3  
**Tools:** `lsof`, `python3`, `curl`

## Commands Used

### 1. Inspect existing listening services

	lsof -nP -iTCP -sTCP:LISTEN

This command lists TCP sockets in the listening state.

- `-n`: Disable hostname resolution.
- `-P`: Display numeric port numbers.
- `-iTCP`: Select TCP sockets.
- `-sTCP:LISTEN`: Show only listening TCP sockets.

The output contained services listening on both wildcard (`*`) and loopback (`127.0.0.1`) addresses.

### 2. Create a test HTTP server

First, I created a directory containing a harmless test file:

	mkdir -p ~/listener-lab
	echo Hello-from-test-server > ~/listener-lab/index.html

Then I started an HTTP server bound to all IPv4 interfaces:

	python3 -m http.server 8000 --bind 0.0.0.0 --directory ~/listener-lab

In another terminal window, I inspected the listening socket:

	lsof -nP -iTCP:8000 -sTCP:LISTEN

**Relevant output (other columns omitted):**

```
COMMAND    NAME
Python     TCP *:8000 (LISTEN)
```

The wildcard address indicated that the server was listening on all available IPv4 interfaces.

### 3. Restrict the server to localhost

After stopping the first server with `Ctrl+C`, I restarted it using the loopback address:

	python3 -m http.server 8000 --bind 127.0.0.1 --directory ~/listener-lab

I repeated the inspection:

	lsof -nP -iTCP:8000 -sTCP:LISTEN

**Relevant output:**

```
COMMAND    NAME
Python     TCP 127.0.0.1:8000 (LISTEN)
```

The listening address had changed from `*` to `127.0.0.1`.

### 4. Test connectivity

I tested the localhost-only server with:

	curl -s http://127.0.0.1:8000

**Output:**

```
Hello-from-test-server
```

The server successfully responded to a connection originating from the same machine (my Mac). However, I wanted to understand whether the binding address actually affected which connections the server would accept.

I therefore performed an additional test using my Mac's Wi-Fi IP address.

**Find the active network interface:**

	route -n get default | grep interface

The active interface was `en0`.

**Retrieve the Wi-Fi IP address:**

	ipconfig getifaddr en0

I kept the actual IP address private.
Then, with the HTTP server still bound to `127.0.0.1`, I ran two connectivity tests.

**Test A — Connect using the Wi-Fi IP:**

	curl --noproxy '*' -s -o /dev/null --max-time 3 -w '%{http_code}\n' http://$(ipconfig getifaddr en0):8000

Output:

```
000
```

**Test B — Connect using localhost:**

	curl --noproxy '*' -s -o /dev/null --max-time 3 -w '%{http_code}\n' http://127.0.0.1:8000

Output:

```
200
```

So, the results were:

|Destination|curl output|Observation|
|---|---|---|
|Wi-Fi IP, port 8000|`000`|No HTTP response received|
|`127.0.0.1:8000`|`200`|HTTP OK|

`200` indicates that the server returned a successful HTTP response. `000` indicates that curl did not receive an HTTP response, but it does not identify the exact connection error.

These results were consistent with the server listening exclusively on the IPv4 loopback address. My Mac could access the service through `127.0.0.1`, but the test using its own Wi-Fi address did not succeed.

I stopped the server after completing the experiment.

## What I Learned

The address a service binds to decides which local network interfaces can accept connections to it.

| Binding address | Meaning                                           |
| --------------- | ------------------------------------------------- |
| `0.0.0.0`       | Listens on all IPv4 interfaces                    |
| `127.0.0.1`     | Listens only on the local IPv4 loopback interface |

Binding a service to all interfaces can make it reachable to other devices on the network, depending on firewall rules and network configuration. Binding it to localhost limits connections to the same machine.

Connecting to `0.0.0.0` locally is not the same as proving that a service is reachable from elsewhere, though. In my experiment, a local connection to this address succeeded, but it does not show that another device could reach the service. This matters when configuring development servers, administrative interfaces, databases, and any other service that should not be exposed more than necessary.

### Limitations

The connectivity tests I performed were all local, using the same machine as both client and server.

I verified the difference between the listening addresses and tested the localhost-only configuration through two destination addresses. However, I did not test the all-interfaces configuration using the Wi-Fi IP, or attempt a connection from a second device.

Therefore, this experiment demonstrated the listening configurations and their observed local behavior, but did not establish actual remote reachability.

## Where I Got Stuck

I initially encountered an unexpected shell quoting issue while creating the test file. My original command had a message in the format `"Hello from [Mac's name]!"`, and instead of executing it, zsh displayed the continuation prompt `cmdand dquote>`.

I later changed the message to `Hello-from-test-server` to avoid including my machine's name in the experiment. Since this had also removed the exclamation mark, I unknowingly eliminated the quoting issue without any shell configuration changes.

Later, after searching about the issue online, I learned that zsh has a history expansion feature (`BANG_HIST`) which treats exclamation marks specially, even inside double quotes. In this case, the `!"` sequence caused the shell to interpret the closing quotation mark differently, leaving it waiting for additional input.

My first connectivity test showed me that the server responded locally, but it couldn’t tell the two binding configurations apart, because both accept connections from localhost. To separate them, I added a test using my Mac’s Wi-Fi IP address.

Finally, I decided against publishing raw terminal screenshots because they contained my local username and device information. The outputs included here are reduced to the relevant fields.

## Conclusion

A service can listen on port 8000 in both the `0.0.0.0` and `127.0.0.1` configurations, but the binding address determines which interfaces accept connections. I used `lsof` to inspect these bindings and `curl` to verify how the server behaved, and the experiment showed why checking listening addresses is a good first step when investigating potentially exposed services.
