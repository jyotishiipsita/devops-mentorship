# Linux Networking — Lab Commands & Outputs
> Session 3 | Phase 1 | DevOps Mentorship
> Date: 2026-03-15
> EC2: ip-172-31-3-215 | Ubuntu 24.04.4 LTS | Kernel 6.17.0-1007-aws

---

## Environment Verification

```bash
uname -a && whoami && hostname -I
```
```
Linux ip-172-31-3-215 6.17.0-1007-aws #7~24.04.1-Ubuntu SMP Thu Jan 22 21:04:49 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux
root
172.31.3.215
```

```bash
cat /etc/os-release
```
```
PRETTY_NAME="Ubuntu 24.04.4 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
VERSION="24.04.4 LTS (Noble Numbat)"
```

---

## 1. Network Interfaces

```bash
ip addr show
```
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 06:c2:5f:21:0f:f3 brd ff:ff:ff:ff:ff:ff
    inet 172.31.3.215/20 metric 100 brd 172.31.15.255 scope global dynamic ens5
       valid_lft 3062sec preferred_lft 3062sec
    inet6 fe80::4c2:5fff:fe21:ff3/64 scope link
       valid_lft forever preferred_lft forever
```

Key observations:
- Two interfaces: `lo` (loopback) and `ens5` (ethernet)
- Private IP only: `172.31.3.215` — public IP lives at IGW, not on interface
- MTU 9001 — AWS jumbo frames for better throughput
- `/20` subnet → 4096 IPs, broadcast at `172.31.15.255`
- `dynamic` → IP assigned by DHCP

```bash
ifconfig
```
```
ens5: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 9001
        inet 172.31.3.215  netmask 255.255.240.0  broadcast 172.31.15.255
        ...
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        ...
```

Note: `netmask 255.255.240.0` = `/20` in CIDR notation

---

## 2. Routing Table

```bash
ip route show
```
```
default via 172.31.0.1 dev ens5 proto dhcp src 172.31.3.215 metric 100
172.31.0.0/20 dev ens5 proto kernel scope link src 172.31.3.215 metric 100
172.31.0.1 dev ens5 proto dhcp scope link src 172.31.3.215 metric 100
172.31.0.2 dev ens5 proto dhcp scope link src 172.31.3.215 metric 100
```

Key observations:
- `default via 172.31.0.1` — default gateway, catches all unknown destinations
- `172.31.0.0/20` — local subnet handled directly on ens5
- `172.31.0.1` — VPC router (always .1 in AWS)
- `172.31.0.2` — AWS DNS resolver (always .2 in AWS VPC)

---

## 3. Listening Ports

```bash
ss -tulnp
```
```
Netid  State   Recv-Q Send-Q  Local Address:Port  Peer Address:Port  Process
udp    UNCONN  0      0       127.0.0.54:53        0.0.0.0:*         users:(("systemd-resolve",pid=424,fd=16))
udp    UNCONN  0      0       127.0.0.53%lo:53     0.0.0.0:*         users:(("systemd-resolve",pid=424,fd=14))
udp    UNCONN  0      0       172.31.3.215%ens5:68 0.0.0.0:*         users:(("systemd-network",pid=493,fd=23))
udp    UNCONN  0      0       127.0.0.1:323        0.0.0.0:*         users:(("chronyd",pid=682,fd=5))
tcp    LISTEN  0      511     0.0.0.0:80           0.0.0.0:*         users:(("nginx",pid=630,fd=6)...)
tcp    LISTEN  0      4096    0.0.0.0:22           0.0.0.0:*         users:(("sshd",pid=1012,fd=3)...)
tcp    LISTEN  0      511     0.0.0.0:443          0.0.0.0:*         users:(("nginx",pid=630,fd=5)...)
tcp    LISTEN  0      4096    127.0.0.53%lo:53     0.0.0.0:*         users:(("systemd-resolve",pid=424,fd=15))
tcp    LISTEN  0      4096    127.0.0.54:53        0.0.0.0:*         users:(("systemd-resolve",pid=424,fd=17))
tcp    LISTEN  0      4096    [::]:22              [::]:*            users:(("sshd",pid=1012,fd=4)...)
```

Key observations:
- Port 22 (sshd) — our active SSH connection
- Port 80 + 443 (nginx) — web server running on this EC2
- Port 53 (systemd-resolve) — local DNS resolver on loopback only
- Port 68 (systemd-network) — DHCP client
- Port 323 (chronyd) — NTP time synchronization

---

## 4. DNS Resolution

### Basic query
```bash
dig google.com
```
```
;; ANSWER SECTION:
google.com.     78    IN    A    142.250.205.46
;; Query time: 2 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
```

### Second query (TTL reset + different IP = DNS load balancing)
```bash
dig google.com
```
```
;; ANSWER SECTION:
google.com.     203   IN    A    142.251.223.14
;; Query time: 1 msec
```

Observations:
- Different IP returned → DNS-based load balancing
- TTL reset (not decremented) → local cache expired, fresh query to AWS DNS
- Query time 1ms → extremely fast, served from cache or very close DNS

### Full DNS trace
```bash
dig google.com +trace
```
```
.                    85983  IN  NS  a.root-servers.net.   ← root servers
...
;; Received 239 bytes from 127.0.0.53#53 in 1 ms

com.                 172800 IN  NS  a.gtld-servers.net.   ← TLD servers
...
;; Received 1170 bytes from 198.41.0.4#53(a.root-servers.net) in 46 ms

google.com.          172800 IN  NS  ns1.google.com.       ← authoritative
...
;; Received 644 bytes from 192.5.6.30#53(a.gtld-servers.net) in 45 ms

google.com.          300    IN  A   142.251.222.174       ← final answer
;; Received 55 bytes from 216.239.36.10#53(ns3.google.com) in 28 ms
```

DNS hierarchy proven: local resolver → root → TLD → authoritative → IP

### DNS configuration files
```bash
cat /etc/resolv.conf
```
```
nameserver 127.0.0.53
options edns0 trust-ad
search ap-south-2.compute.internal
```

```bash
ls -la /etc/resolv.conf
```
Result: symlink → managed dynamically by systemd-resolved

```bash
cat /etc/nsswitch.conf | grep hosts
```
```
hosts: files dns
```
Resolution order: `/etc/hosts` first, then DNS

```bash
cat /etc/hosts
```
```
127.0.0.1  localhost
127.0.1.1  ip-172-31-3-215
::1        localhost ip6-localhost ip6-loopback
```

---

## 5. Network Path Analysis

```bash
traceroute google.com
```
```
traceroute to google.com (142.250.193.174), 30 hops max
 1  240.2.196.13     15ms  ← AWS internal backbone
 2  242.6.253.7      14ms  ← AWS internal backbone
 3  * * *                  ← silent hop (ICMP blocked)
 4  99.82.178.53     14ms  ← AWS edge router (exits AWS here)
 5  * * *                  ← silent hop
 6  142.250.208.220  15ms  ← Google backbone entry
 7  142.250.213.100  14ms  ← Google backbone
 8  192.178.254.237  16ms  ← Google backbone
 9  216.239.49.46    34ms  ← Google backbone
10  142.251.50.59    25ms  ← Google backbone
11  142.250.235.107  25ms  ← Google backbone
12  142.250.193.174  22ms  ← Google server ✅
```

Three zones visible: AWS backbone (1-4) → Internet (5) → Google backbone (6-12)

```bash
curl ipinfo.io
```
```json
{
  "ip": "18.61.44.227",
  "hostname": "ec2-18-61-44-227.ap-south-2.compute.amazonaws.com",
  "city": "Hyderabad",
  "region": "Telangana",
  "country": "IN",
  "org": "AS16509 Amazon.com, Inc."
}
```

EC2 public IP: `18.61.44.227` | AWS AS number: AS16509

---

## 6. HTTP/TLS Handshake

```bash
curl -v https://google.com 2>&1 | head -50
```
```
* Host google.com:443 was resolved.
* IPv4: 142.251.43.78
*   Trying 142.251.43.78:443...
* Connected to google.com (142.251.43.78) port 443       ← TCP handshake done
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1)         ← TLS starts
* TLSv1.3 (IN),  TLS handshake, Server hello (2)
* TLSv1.3 (IN),  TLS handshake, Encrypted Extensions (8)
* TLSv1.3 (IN),  TLS handshake, Certificate (11)         ← Google's cert received
* TLSv1.3 (IN),  TLS handshake, CERT verify (15)
* TLSv1.3 (IN),  TLS handshake, Finished (20)
* TLSv1.3 (OUT), TLS handshake, Finished (20)            ← TLS done
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / X25519 / id-ecPublicKey
* ALPN: server accepted h2
* subject: CN=*.google.com
* start date: Feb  2 08:36:42 2026 GMT
* expire date: Apr 27 08:36:41 2026 GMT
* subjectAltName: host "google.com" matched cert's "google.com"
* issuer: C=US; O=Google Trust Services; CN=WE2
* SSL certificate verify ok.                             ← certificate chain verified
* using HTTP/2
* [:method: GET]
* [:path: /]
```

---

## 7. Firewall

```bash
iptables -L -n --line-numbers
```
```
Chain INPUT (policy ACCEPT)
num  target  prot  opt  source  destination

Chain FORWARD (policy ACCEPT)
num  target  prot  opt  source  destination

Chain OUTPUT (policy ACCEPT)
num  target  prot  opt  source  destination
```

Observation: Empty iptables — EC2 protected by AWS Security Groups at infrastructure level. iptables becomes critical when Docker/Kubernetes manage container networking.

---

## 8. Network Namespaces

```bash
ip netns list
```
```
(empty — no containers running)
```

### Manual namespace lab (simulating Docker internals)

```bash
# Create isolated namespace
ip netns add test-ns

# Verify creation
ip netns list
```
```
test-ns
```

```bash
# Show host interfaces (normal view)
ip addr show
```
```
1: lo: ... inet 127.0.0.1/8
2: ens5: ... inet 172.31.3.215/20
```

```bash
# Show interfaces INSIDE the namespace (completely isolated)
ip netns exec test-ns ip addr show
```
```
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN
    link/loopback 00:00:00:00:00:00
```

Observation: Namespace has ONLY a bare loopback, no ens5, no IP. Completely isolated from host. This is exactly how Docker starts a container's network.

```bash
# Clean up
ip netns delete test-ns
ip netns list
```
```
(empty again)
```

---

## Quick Reference — Commands Learned This Session

| Command | Purpose |
|---------|---------|
| `ip addr show` | Show all network interfaces and IPs |
| `ip route show` | Show routing table |
| `ss -tulnp` | Show all listening ports + processes |
| `dig <domain>` | DNS lookup |
| `dig <domain> +trace` | Full DNS resolution chain |
| `cat /etc/resolv.conf` | Show configured DNS server |
| `cat /etc/hosts` | Show local DNS overrides |
| `cat /etc/nsswitch.conf` | Show name resolution order |
| `traceroute <host>` | Show network path hop by hop |
| `curl ipinfo.io` | Show EC2's public IP and metadata |
| `curl -v https://<host>` | Show full TCP/TLS/HTTP handshake |
| `iptables -L -n --line-numbers` | Show firewall rules |
| `ip netns list` | List network namespaces |
| `ip netns add <name>` | Create network namespace |
| `ip netns exec <name> <cmd>` | Run command inside namespace |
| `ip netns delete <name>` | Delete network namespace |
