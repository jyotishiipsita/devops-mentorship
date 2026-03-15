# Linux Networking — Interview Q&A
> Session 3 | Phase 1 | DevOps Mentorship
> Date: 2026-03-15
> Topic: Linux Networking Fundamentals for DevOps

---

## Core Networking

**Q1. What is the difference between `lo` and `ens5` interfaces on a Linux machine?**

`lo` is the loopback interface — a virtual interface that exists entirely in software. Traffic sent to `127.0.0.1` never leaves the machine; it loops back to itself. It's used for inter-process communication on the same host.

`ens5` is the actual ethernet interface — connected to the physical (or virtualised) network card. Traffic sent through `ens5` leaves the machine and travels to the network. On AWS EC2, `ens5` is the virtualised NIC connected to the VPC.

---

**Q2. On an AWS EC2 instance, `ip addr show` only shows the private IP (172.31.x.x) but never the public IP. Why?**

The public IP is not assigned to the EC2 network interface. It lives at the AWS Internet Gateway layer as a 1:1 NAT mapping. When outbound traffic leaves the EC2 with private source IP `172.31.x.x`, the IGW replaces it with the public IP. When inbound traffic arrives at the public IP, the IGW translates it to the private IP and forwards it to the EC2. The EC2 itself is completely unaware of its public IP — it only knows its private IP.

---

**Q3. What does `/20` mean in `172.31.3.215/20` and how many usable IP addresses does this subnet have?**

`/20` means the first 20 bits are the network (fixed) part and the remaining 12 bits are the host part. Total IPs = 2^12 = 4096. On AWS, 5 IPs are reserved per subnet (network address, VPC router, DNS, future use, broadcast), leaving 4091 usable IPs.

The subnet range is `172.31.0.0` (network) to `172.31.15.255` (broadcast). Calculation: `172.31.0.0 + 4095 = 172.31.15.255`.

---

**Q4. What are the 5 reserved IPs in every AWS subnet and what are they used for?**

| Address | Reserved for |
|---------|-------------|
| x.x.x.0 | Network address — identifies the subnet |
| x.x.x.1 | VPC router — all traffic goes here first |
| x.x.x.2 | AWS DNS resolver |
| x.x.x.3 | AWS future use |
| x.x.x.255 | Broadcast address |

So a `/28` subnet (16 IPs) has only 11 usable addresses.

---

**Q5. What is the difference between a public subnet and a private subnet in AWS?**

The only technical difference is the VPC route table entry:

- **Public subnet**: route table has `0.0.0.0/0 → Internet Gateway`. Traffic with unknown destinations escapes to the internet.
- **Private subnet**: no `0.0.0.0/0 → IGW` rule. Traffic hits the VPC router and has nowhere to go for internet-bound packets — they are dropped.

A private subnet can still reach the internet via a NAT Gateway placed in a public subnet.

---

**Q6. Explain the two-layer routing that happens when an EC2 instance sends a packet to the internet.**

Layer 1 — Linux routing table (inside EC2, seen with `ip route show`):
The EC2 kernel checks its own route table. The destination IP doesn't match any local subnet, so it matches `default via 172.31.0.1`. The packet is sent to the VPC router at `172.31.0.1`.

Layer 2 — AWS VPC route table (outside EC2, in AWS infrastructure):
The VPC router checks the subnet's route table. It finds the rule `0.0.0.0/0 → igw-xxxxxxxx`. The packet is forwarded to the Internet Gateway, which performs NAT (private IP → public IP) and sends the packet to the internet.

Both route tables must have correct rules. If either is missing the right entry, the packet is dropped silently.

---

## DNS

**Q7. Walk me through the complete DNS resolution chain when a Linux machine resolves `google.com`.**

1. Application calls `gethostbyname("google.com")`
2. OS checks `/etc/hosts` first — if entry exists, return immediately
3. No entry → query goes to `127.0.0.53` (systemd-resolved, local caching proxy)
4. Cache hit → return cached IP immediately
5. Cache miss → systemd-resolved forwards to `172.31.0.2` (AWS DNS resolver)
6. AWS DNS performs recursive resolution:
   - Asks root servers (`a-m.root-servers.net`) → "who handles `.com`?"
   - Asks TLD servers (`a.gtld-servers.net`) → "who handles `google.com`?"
   - Asks Google's authoritative DNS (`ns1-4.google.com`) → "what is `google.com`'s IP?"
7. Answer returned through the chain, cached at each layer with TTL
8. Application receives IP address

---

**Q8. What is TTL in DNS and what happens when it reaches zero?**

TTL (Time To Live) is the number of seconds a DNS answer can be cached. Each caching resolver counts down the TTL. When it reaches zero, the cached answer is discarded and the next query triggers a fresh DNS lookup upstream.

TTL is set by the domain owner. Low TTL (e.g. 60s) allows fast DNS changes but increases DNS load. High TTL (e.g. 86400s = 1 day) reduces DNS load but means changes propagate slowly.

---

**Q9. What is DNS-based load balancing and how does Google use it?**

DNS-based load balancing works by having multiple A records for the same domain — each pointing to a different server IP. The DNS server rotates which IP it returns, distributing traffic across multiple servers.

When you run `dig google.com` twice, you may get different IPs each time (`142.250.205.46` then `142.251.223.14`). Google has hundreds of servers globally. The DNS resolver returns different IPs based on geographic proximity, server load, and rotation policies — spreading billions of requests across their infrastructure.

---

**Q10. What is the purpose of `/etc/hosts` and how does it interact with DNS?**

`/etc/hosts` is a local file that maps hostnames to IP addresses. It is checked **before** DNS — the resolution order is defined in `/etc/nsswitch.conf` as `hosts: files dns`.

If `google.com` is listed in `/etc/hosts` with IP `1.2.3.4`, all requests to `google.com` from that machine will go to `1.2.3.4` regardless of what DNS says. DNS is never even queried.

Use cases: local development overrides, blocking domains (map to `127.0.0.1`), Kubernetes pod DNS injection.

---

**Q11. What is `systemd-resolved` and why does DNS listen on `127.0.0.53` instead of directly using `172.31.0.2`?**

`systemd-resolved` is a local DNS caching stub resolver that acts as an intermediary between applications and the real DNS server. It listens on `127.0.0.53`.

Benefits:
- **Caching**: repeated queries are answered instantly from cache without hitting the network
- **Centralised management**: one place to configure DNS for the entire machine
- **DNSSEC validation**: validates DNS responses cryptographically before passing to apps

`127.0.0.53` is the stub resolver (main cache + forwarder). `127.0.0.54` is a secondary stub for local `.local` hostname resolution only.

---

## TCP/TLS

**Q12. What is the TCP 3-way handshake and why is it needed?**

Before any data can be exchanged over TCP, both sides must establish a connection:

1. **SYN**: Client sends synchronisation packet → "I want to connect, my sequence starts at X"
2. **SYN-ACK**: Server acknowledges → "OK, my sequence starts at Y, acknowledged yours"
3. **ACK**: Client acknowledges server's sequence → "Connection established"

This ensures both sides are alive, ready, and have agreed on sequence numbers for reliable ordered delivery. UDP skips this entirely — fire and forget.

---

**Q13. When you run `curl -v https://google.com`, how does your machine verify it's talking to the real Google and not an imposter?**

Through the TLS certificate chain of trust:

1. Google sends its certificate (`CN=*.google.com`)
2. The certificate is signed by `Google Trust Services WE2` (intermediate CA)
3. Your machine checks `/etc/ssl/certs/ca-certificates.crt` — a file containing ~100 trusted root CAs
4. Google Trust Services chains up to a trusted root CA in that file
5. The `subjectAltName` on the certificate matches the domain (`google.com`)
6. `SSL certificate verify ok` — identity confirmed

If an attacker intercepts the connection (MITM), they cannot forge a certificate signed by Google Trust Services without access to the private key — so the verification would fail and the connection would be rejected.

---

**Q14. What is the difference between TCP and UDP? Give examples of when each is used.**

| | TCP | UDP |
|--|-----|-----|
| Full name | Transmission Control Protocol | User Datagram Protocol |
| Connection | Requires 3-way handshake | Connectionless |
| Delivery | Guaranteed, ordered, error-checked | Best effort, no guarantee |
| Speed | Slower (overhead) | Faster (no overhead) |
| Use cases | SSH, HTTP/S, databases | DNS, NTP, video streaming, VoIP |

TCP = registered mail. UDP = dropping a letter in a postbox.

For DevOps: Kubernetes health checks often use both. DNS uses UDP (small fast queries), but falls back to TCP for large responses. Container overlay networks often use UDP for performance.

---

## Firewall & Security

**Q15. What is the difference between AWS Security Groups and Linux `iptables`?**

| | AWS Security Groups | iptables |
|--|--------------------|----|
| Location | AWS infrastructure (outside EC2) | Linux kernel (inside EC2) |
| Awareness | Knows EC2 instances, VPCs | Knows network interfaces, IPs |
| Container-aware | No | Yes (Docker writes iptables rules) |
| Stateful | Yes | Configurable (conntrack) |
| Managed by | AWS console/CLI | Linux commands |

Security Groups protect the EC2 at the infrastructure level — before traffic even reaches the OS. iptables operates inside the OS and becomes critical for container networking: Docker and Kubernetes automatically write iptables rules to handle container-to-container routing, port mapping, and NAT.

---

## Network Namespaces & Containers

**Q16. What is a Linux network namespace and how does Docker use it?**

A network namespace is an isolated copy of the entire Linux networking stack. Each namespace has its own network interfaces, routing table, iptables rules, and IP addresses. Processes in one namespace cannot see or interact with network resources in another namespace.

Docker creates a new network namespace for every container:
1. Create namespace (isolated blank slate — only loopback, no IP)
2. Create a veth (virtual ethernet) pair — two virtual interfaces linked together
3. Place one end inside the container namespace, one end on the host
4. Assign IP to the container interface (e.g. `172.17.0.2`)
5. Set up routes inside the namespace
6. Container now has isolated, functional network

This is why 100 containers can each have their own IP on the same host with one physical NIC.

---

**Q17. Why does `ip netns exec test-ns ip addr show` only show a bare loopback with no IP?**

A freshly created network namespace starts completely empty — it has no interfaces except a bare loopback (`lo`) which is DOWN with no IP assigned. This is the blank slate before container networking is wired up.

In Docker, after creating the namespace, it immediately:
- Creates and attaches a veth pair
- Assigns an IP from the Docker bridge subnet (`172.17.0.0/16`)
- Brings the interface UP
- Adds a default route inside the namespace

Without these steps, the container would have no network connectivity at all.

---

## The Big Picture

**Q18. An interviewer asks: "What happens step by step when you run `curl google.com` on a Linux EC2 instance?" Give a complete answer.**

**Phase 1 — DNS Resolution:**
1. curl asks the OS to resolve `google.com`
2. OS checks `/etc/hosts` — not found
3. Query goes to `127.0.0.53` (systemd-resolved)
4. Cache miss → forwarded to `172.31.0.2` (AWS DNS)
5. Recursive resolution: root servers → `.com` TLD servers → `ns1-4.google.com`
6. Returns `142.251.x.x`, cached at systemd-resolved

**Phase 2 — Routing:**
7. curl has the IP — OS checks routing table
8. Doesn't match local subnet → default gateway `172.31.0.1`
9. Packet reaches VPC router → VPC route table says `0.0.0.0/0 → IGW`
10. IGW performs NAT: `172.31.3.215 → 18.61.44.227` (public IP)

**Phase 3 — TCP Handshake:**
11. SYN → Google server
12. SYN-ACK ← Google server
13. ACK → Google server (connection established on port 80)

**Phase 4 — HTTP Request:**
14. `GET / HTTP/1.1` sent
15. `200 OK` + HTML received
16. curl displays response

For HTTPS, TLS handshake is inserted between steps 13 and 14: Client hello → Server hello → Certificate exchange → Certificate verification against CA store → Key exchange → Encrypted tunnel established → HTTP request sent inside the tunnel.

---

**Q19. You create a `/26` subnet in AWS. How many usable IPs do you have?**

`/26` → host bits = 32 - 26 = 6 → 2^6 = 64 total IPs
Minus 5 AWS reserved = **59 usable IPs**

Subnet range: x.x.x.0 to x.x.x.63

---

**Q20. A junior engineer says "I opened port 8080 in the Security Group but I still can't reach my app." What are the possible reasons and how would you debug?**

Security Group is only one layer. Check all of these:

1. **Is the app actually listening on port 8080?** → `ss -tulnp | grep 8080`
2. **Is it listening on the right interface?** → `0.0.0.0:8080` means all interfaces, `127.0.0.1:8080` means localhost only (unreachable from outside)
3. **Does iptables have a blocking rule?** → `iptables -L -n | grep 8080`
4. **Is the Security Group inbound rule correct?** → check source CIDR, port range, protocol
5. **Is the app process running at all?** → `ps aux | grep <appname>`
6. **Are you connecting to the right IP?** → public IP for internet, private IP for VPC-internal
7. **Is the VPC route table correct?** → for internet access, needs `0.0.0.0/0 → IGW`

Systematic debugging: start from the application layer and work outward — app → OS → firewall → AWS.

---
