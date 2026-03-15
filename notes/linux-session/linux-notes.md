# Linux Networking — Concepts & Theory
> Session 3 | Phase 1 | DevOps Mentorship
> Date: 2026-03-15
> EC2: Ubuntu 24.04, ap-south-1, 172.31.3.215

---

## 1. Network Interfaces (`ip addr show`)

A **network interface** is a point of connection between your machine and a network. Linux assigns each interface a name.

### Common Interface Types

| Interface | Name Pattern | Description |
|-----------|-------------|-------------|
| Loopback | `lo` | Virtual interface, never leaves the machine |
| Ethernet (physical/virtual) | `eth0`, `ens5` | Real or virtualised NIC |
| Docker bridge | `docker0` | Created by Docker daemon |
| Virtual ethernet | `veth*` | Container networking pairs |

### Key Fields in `ip addr show`

| Field | Meaning |
|-------|---------|
| `inet` | IPv4 address assigned to interface |
| `inet6` | IPv6 address |
| `mtu` | Maximum Transmission Unit — max packet size in bytes |
| `scope global` | Reachable from outside the machine |
| `scope host` | Only reachable within this machine |
| `dynamic` | Address assigned by DHCP |
| `brd` | Broadcast address — reaches all hosts in subnet |

### EC2-Specific: Why No Public IP on Interface?
On AWS EC2, the **public IP is not assigned to the network interface**. It lives at the Internet Gateway (IGW) as a 1:1 NAT mapping. The EC2 only knows its private IP. This is why `ip addr show` never shows your public IP.

---

## 2. CIDR and Subnetting

**CIDR (Classless Inter-Domain Routing)** notation expresses an IP address and its subnet mask together.

### Formula
```
Total IPs in subnet = 2^(32 - prefix)
```

### Examples

| CIDR | Host bits | Total IPs | Usable IPs (AWS) |
|------|-----------|-----------|------------------|
| /20  | 12        | 4096      | 4091             |
| /24  | 8         | 256       | 251              |
| /28  | 4         | 16        | 11               |

### AWS Reserved IPs (5 per subnet)

| Address | Reserved for |
|---------|-------------|
| x.x.x.0 | Network address |
| x.x.x.1 | VPC router |
| x.x.x.2 | AWS DNS resolver |
| x.x.x.3 | AWS future use |
| x.x.x.255 | Broadcast |

### Example: 172.31.3.215/20
- Network address: `172.31.0.0`
- Broadcast: `172.31.15.255` (172.31.0.0 + 4095)
- VPC router: `172.31.0.1`
- AWS DNS: `172.31.0.2`
- Total IPs: 4096 | Usable: 4091

---

## 3. Routing Table (`ip route show`)

A **routing table** is a signpost that tells the kernel where to send packets based on destination IP.

### Route Types

| Entry | Meaning |
|-------|---------|
| `default via x.x.x.x` | Default gateway — used when no specific route matches |
| `x.x.x.x/prefix dev ens5` | Local subnet — handled directly on this interface |
| `proto dhcp` | Route learned via DHCP |
| `proto kernel` | Route added automatically by kernel |

### Two-Layer Routing on AWS

```
Layer 1 — Linux routing table (inside EC2, seen with ip route show)
  → decides: "send to default gateway 172.31.0.1"

Layer 2 — AWS VPC route table (outside EC2, in AWS infrastructure)
  → decides: "0.0.0.0/0 → Internet Gateway"
```

Both layers must have correct rules for internet traffic to flow.

### Public vs Private Subnet (the key difference)

| Subnet Type | VPC Route Table Rule | Result |
|-------------|---------------------|--------|
| Public | `0.0.0.0/0 → igw-xxx` | Traffic escapes to internet |
| Private | No `0.0.0.0/0` rule | Traffic dies at VPC router |

---

## 4. Listening Ports (`ss -tulnp`)

`ss` (socket statistics) shows all active network sockets on the system.

### Flag Meanings

| Flag | Meaning |
|------|---------|
| `-t` | Show TCP sockets |
| `-u` | Show UDP sockets |
| `-l` | Show only listening sockets |
| `-n` | Show port numbers (don't resolve to names) |
| `-p` | Show process name and PID |

### Socket States

| State | Protocol | Meaning |
|-------|----------|---------|
| `LISTEN` | TCP | Waiting to accept incoming connections |
| `UNCONN` | UDP | Ready to receive — UDP has no connection concept |
| `ESTABLISHED` | TCP | Active connection in progress |

### TCP vs UDP

| | TCP | UDP |
|--|-----|-----|
| Delivery guarantee | Yes | No |
| Connection required | Yes (3-way handshake) | No |
| Speed | Slower | Faster |
| Use cases | SSH, HTTP, HTTPS | DNS, NTP, video streaming |

---

## 5. DNS Resolution

### Resolution Chain

```
Application
    ↓
/etc/hosts (checked first — local override)
    ↓ not found
127.0.0.53 — systemd-resolved (local caching proxy)
    ↓ cache miss
172.31.0.2 — AWS DNS resolver
    ↓
Root servers (13 globally, a-m.root-servers.net)
    ↓
TLD servers (e.g. a.gtld-servers.net for .com)
    ↓
Authoritative DNS (e.g. ns1-4.google.com)
    ↓
IP address returned ✅
```

### Key DNS Files

| File | Purpose |
|------|---------|
| `/etc/resolv.conf` | Points to the DNS server to use (nameserver 127.0.0.53) |
| `/etc/hosts` | Local overrides — checked before DNS |
| `/etc/nsswitch.conf` | Resolution order: `files dns` |

### Important DNS Concepts

| Concept | Meaning |
|---------|---------|
| TTL | Time To Live — how long a DNS answer is cached (in seconds) |
| A record | Maps domain → IPv4 address |
| NS record | Points to authoritative nameserver |
| DNS-based load balancing | Multiple A records returned in rotation |
| DNSSEC | Cryptographic signatures on DNS responses (prevents spoofing) |

### systemd-resolved: Two Local Addresses

| Address | Role |
|---------|------|
| `127.0.0.53` | Main resolver — caching, forwarding to AWS DNS |
| `127.0.0.54` | Stub resolver — handles only local `.local` hostnames |

The `%lo` suffix (e.g. `127.0.0.53%lo`) means the socket is bound specifically to the loopback interface — unreachable from outside the machine.

---

## 6. Network Path Analysis (`traceroute`)

`traceroute` sends packets with incrementing TTL values. Each router decrements TTL and when it hits 0, sends back an ICMP "time exceeded" message — revealing its IP.

### `* * *` in traceroute output
These are **silent hops** — routers that block ICMP responses. The packet still passed through; the router just didn't reply. Not a sign of failure.

### Reading traceroute for AWS → Internet

```
Hops 1-2:  240.x.x.x / 242.x.x.x  → AWS internal backbone
Hop 4:     99.82.x.x               → AWS edge router (packet leaves AWS here)
Hops 6-11: 142.250.x.x             → Google backbone
Hop 12:    destination              → Google server
```

---

## 7. HTTP/TLS Handshake (`curl -v`)

### Full Connection Sequence

```
Phase 1: DNS resolution
Phase 2: TCP 3-way handshake (SYN → SYN-ACK → ACK)
Phase 3: TLS handshake
  → Client hello (offers supported cipher suites)
  → Server hello (chooses cipher suite)
  → Certificate exchange
  → Certificate verification against CA store
  → Key exchange (X25519)
  → Finished
Phase 4: HTTP request (GET /)
Phase 5: HTTP response
```

### TLS Certificate Chain of Trust

```
Root CA (trusted, built into /etc/ssl/certs/ca-certificates.crt)
    ↓ signed
Intermediate CA (e.g. Google Trust Services WE2)
    ↓ signed
Server certificate (*.google.com)
```

If any link in this chain is broken or unrecognised → connection rejected.

### TLS 1.3 Parameters (from curl -v output)

| Parameter | Value | Meaning |
|-----------|-------|---------|
| Protocol | TLSv1.3 | Latest, most secure TLS version |
| Cipher | AES_256_GCM | Symmetric encryption algorithm |
| Hash | SHA384 | Integrity verification |
| Key exchange | X25519 | Elliptic curve Diffie-Hellman |

---

## 8. Firewall (`iptables`)

### Three Default Chains

| Chain | Controls |
|-------|---------|
| INPUT | Incoming traffic to this machine |
| FORWARD | Traffic passing through (routing/NAT) |
| OUTPUT | Outgoing traffic from this machine |

### Default Policy
`ACCEPT` = allow everything unless a rule explicitly drops it (default on EC2)
`DROP` = block everything unless a rule explicitly allows it (hardened servers)

### iptables vs AWS Security Groups

| | AWS Security Group | iptables |
|--|-------------------|---------|
| Lives | AWS infrastructure (outside EC2) | Linux kernel (inside EC2) |
| Managed by | AWS console/CLI | OS commands |
| Granularity | Port + source IP/CIDR | Port, IP, protocol, state, interface |
| Docker/K8s | Not aware of containers | Manages all container NAT/routing |

**Why iptables matters for containers:** Docker and Kubernetes write iptables rules automatically to manage container networking, port forwarding, and NAT — even when you don't configure iptables yourself.

---

## 9. Network Namespaces

A **network namespace** is an isolated copy of the Linux networking stack. Each namespace has its own:
- Network interfaces
- Routing table
- iptables rules
- IP addresses

### Why This Matters for Containers

```
Host machine
├── Default namespace → ens5, 172.31.3.215
├── Container 1 namespace → veth0, 172.17.0.2
├── Container 2 namespace → veth0, 172.17.0.3
└── Container 3 namespace → veth0, 172.17.0.4
```

Each container lives in its own isolated namespace — it cannot see other containers' interfaces or traffic.

### Docker's Network Setup Per Container
1. Create new network namespace
2. Create veth (virtual ethernet) pair
3. One end in container namespace, one end on host
4. Assign IP to container interface
5. Set up routes inside namespace
6. Container has isolated, functional network

---

## 10. Complete Packet Journey: `curl google.com`

```
PHASE 1 — DNS RESOLUTION
1. curl asks OS: "what is google.com's IP?"
2. OS checks /etc/hosts → not found
3. Query → 127.0.0.53 (systemd-resolved)
4. Cache hit? return IP | Cache miss? forward to 172.31.0.2
5. 172.31.0.2 → root servers → TLD servers → ns1-4.google.com
6. Returns 142.251.x.x, cached at 127.0.0.53

PHASE 2 — ROUTING
7. OS checks routing table: 142.251.x.x → default → 172.31.0.1
8. Packet → VPC router (172.31.0.1)
9. VPC route table: 0.0.0.0/0 → Internet Gateway
10. IGW NAT: 172.31.3.215 → 18.61.44.227 (public IP)

PHASE 3 — TCP HANDSHAKE
11. SYN → Google
12. SYN-ACK ← Google
13. ACK → Google (connection established)

PHASE 4 — TLS HANDSHAKE
14. Client hello → Google
15. Server hello ← Google
16. Certificate ← Google
17. Certificate verified against CA store
18. Key exchange, encryption established

PHASE 5 — HTTP REQUEST
19. GET / HTTP/2 → Google
20. 200 OK + HTML ← Google
```

---
