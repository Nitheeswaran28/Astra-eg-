# SOC-1 Networking Essentials

## 1. Lab Overview

Follow a connection from a Linux network interface to the logs and packets it produces. Use the host configuration, listening sockets and firewall records to explain what Ubuntu could see. The findings below describe expected evidence.

## 2. Lab Environment

Ubuntu and Kali Linux share an internal lab network. Ubuntu provides the observation point for authentication logs, Apache access logs, UFW and Wireshark. The exercises cover SSH (TCP 22), HTTP (TCP 80), a test TCP listener (4444) and a blocked connection example (TCP 23).

## 3. Workflow Diagram

Interface → route and DNS → listening service → traffic → packet capture → firewall decision

## 4. Tools Used

| Tool | Purpose |
|---|---|
| ip, hostname, resolvectl | Identify local addresses, routes and DNS servers |
| ss, Nmap | Compare local listeners with network-visible ports |
| ping, ssh, curl, Netcat | Illustrate ICMP, SSH, HTTP and raw TCP traffic |
| Wireshark | Inspect captured packets and protocol fields |
| tail, UFW | Observe service logs and firewall decisions |

## 5. Task-by-Task Walkthrough

### T1 — Identify network configuration

**Goal.** Determine how Ubuntu reaches local and remote destinations.

1. Identify the active interface and compare its address with the host address list.
2. Find the default gateway and the directly connected network.
3. Inspect the resolver file, then check which upstream DNS server serves the interface.

```bash
ip a
hostname -I
ip route
cat /etc/resolv.conf
resolvectl status
```

**Command explained.** `ip a` includes interface names and address prefixes. Uppercase `-I` lists host addresses. In the routing table, `default via` identifies the next hop for destinations without a more specific route.

**Expected finding.** A resolver file can point to the local stub at `127.0.0.53`. The interface details in `resolvectl status` identify its upstream resolver.

### T2 — Map ports to services

**Goal.** Connect a network port to the program that listens on it.

1. List Ubuntu's listening sockets.
2. Compare the address and port with the lab's network-side Nmap observations.

```bash
ss -tulnp
```

**Command explained.** `-t` and `-u` select TCP and UDP, `-l` selects listeners, `-n` keeps numeric addresses and ports, and `-p` includes process details where permissions allow.

**Expected finding.** A loopback listener serves local clients. A listener bound to a network interface may also accept remote clients, depending on firewall rules.

### T3 — Trace generated traffic

**Goal.** Relate a client action to a server-side record.

1. In the lab's connectivity check, replace `<PEER_IP>` with the assigned peer address. Four echo requests provide a short reachability sample.
2. Observe Ubuntu's authentication log during the SSH exercise and Apache's access log during the HTTP exercise. Use separate terminals or press Ctrl+C to stop following a file.
3. Compare an ordinary web request with a request for the lab's admin path. For the Netcat exercise, distinguish the listening endpoint from the connecting endpoint.

```bash
ping -c 4 <PEER_IP>
sudo tail -f /var/log/auth.log
sudo tail -f /var/log/apache2/access.log
```

**Command explained.** `-c 4` limits the packet count. With `tail`, `-f` displays new lines as the service writes them.

**Expected finding.** SSH records can identify a source and an authentication outcome. Apache records the requested path and response status. A TCP connection alone cannot establish a successful login.

### T4 — Inspect packets in Wireshark

**Goal.** Distinguish SSH connections from HTTP request patterns.

1. Capture on Ubuntu's active lab interface.
2. Apply each display filter separately. For SSH, inspect the endpoints and timing. For HTTP, inspect the sequence of requested paths and response codes.
3. Compare repeated connections or bursts of directory requests with the server logs from T3.

```wireshark
tcp.port == 22
http
```

**Command explained.** The TCP filter selects packets with source or destination port 22. The HTTP filter selects traffic decoded as HTTP.

**Expected finding.** SSH encryption hides the password and session contents. Multiple requests for different paths over a short period can indicate enumeration. Server logs provide context for that interpretation.

### T5 — Interpret firewall records

**Goal.** Determine how Ubuntu handled a connection attempt.

1. Check UFW's status and logging setting on the lab machine.
2. Inspect the log for the lab's blocked TCP connection to port 23.

```bash
sudo ufw status verbose
cat /var/log/ufw.log
```

**Command explained.** `verbose` shows policy and logging details. In a block record, `SRC` and `DST` identify endpoints, while `SPT` and `DPT` identify ports.

**Expected finding.** A record containing `[UFW BLOCK]`, `PROTO=TCP` and `DPT=23` shows a firewall block for that destination port.

## 6. Key Concepts (Beginner-Friendly)

A route chooses the next hop. DNS resolves a name to an address. A port identifies a service endpoint on that address. These are separate checks when a client cannot connect.

## 7. SOC / Real-World Relevance

Compare packet times, source addresses and server log entries when investigating repeated login attempts. Use firewall records to distinguish traffic blocked at the host from traffic that reached the application.

## 8. Quick Command Cheat Sheet

| Command | What It Does |
|---|---|
| `ip route` | Shows local routes and the default gateway |
| `resolvectl status` | Shows DNS settings by interface |
| `ss -tulnp` | Lists listening sockets and available process details |
| `sudo ufw status verbose` | Displays firewall policy and logging state |

## 9. Common Mistakes & Troubleshooting

- **Empty capture.** Match the capture interface to the address shown by `ip a`. Start capturing before the traffic occurs.
- **Unexpected DNS address.** Check whether the resolver file points to a local stub, then inspect the upstream settings.
- **Missing process name.** Check permissions before concluding a socket has no owner.

## 10. Conclusion

Interface and routing information explain where packets should travel.
Socket and application records identify the service involved.
Packet captures and UFW records show how far a connection progressed.
