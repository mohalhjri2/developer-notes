# Networking — The Complete Field Guide

> "The network is the computer." — John Gage, Sun Microsystems

---

## Table of Contents

1. [OSI & TCP/IP Models](#osi--tcpip-models)
2. [IP Addressing & Subnetting](#ip-addressing--subnetting)
3. [TCP & UDP](#tcp--udp)
4. [DNS](#dns)
5. [HTTP & HTTPS](#http--https)
6. [TLS/SSL](#tlsssl)
7. [Load Balancing](#load-balancing)
8. [Firewalls & iptables](#firewalls--iptables)
9. [NAT & Port Forwarding](#nat--port-forwarding)
10. [VPNs & Tunneling](#vpns--tunneling)
11. [Network Tools](#network-tools)
12. [Common Ports](#common-ports)
13. [Performance Tuning](#performance-tuning)
14. [Troubleshooting Methodology](#troubleshooting-methodology)

---

## OSI & TCP/IP Models

```
OSI Model                   TCP/IP Model      Examples
─────────────────────────────────────────────────────────
7 Application               Application       HTTP, HTTPS, DNS, SMTP, FTP
6 Presentation              ↑                 TLS/SSL, encoding
5 Session                   ↑                 Session management
4 Transport                 Transport         TCP, UDP
3 Network                   Internet          IP, ICMP, routing
2 Data Link                 Network Access    Ethernet, MAC, ARP, switches
1 Physical                  ↑                 Cables, fiber, WiFi, hubs

Encapsulation:
[Ethernet Header | IP Header | TCP Header | HTTP | Data]
                  └── Packet ───────────────────────────┘
     └── Frame ──────────────────────────────────────────┘
```

### How a Request Works (HTTP)
```
1. DNS lookup: "google.com" → 142.250.x.x
2. TCP 3-way handshake: SYN → SYN-ACK → ACK
3. TLS handshake (HTTPS): ClientHello → ServerHello → Certificate → Keys
4. HTTP GET request sent
5. Server processes, sends response
6. TCP FIN → FIN-ACK → ACK (or keep-alive)
```

---

## IP Addressing & Subnetting

### IPv4 Classes (legacy, now CIDR)
```
Class A: 1.0.0.0   – 126.0.0.0    /8  (~16M hosts)
Class B: 128.0.0.0 – 191.255.0.0  /16 (~65K hosts)
Class C: 192.0.0.0 – 223.255.255.0 /24 (254 hosts)
```

### Private Address Ranges
```
10.0.0.0/8         10.0.0.0 – 10.255.255.255  (16.7M addresses)
172.16.0.0/12      172.16.0.0 – 172.31.255.255 (1M addresses)
192.168.0.0/16     192.168.0.0 – 192.168.255.255 (65K addresses)
127.0.0.0/8        Loopback
169.254.0.0/16     Link-local (APIPA — no DHCP found)
```

### CIDR Notation
```
/32  = 1 host         255.255.255.255
/31  = 2 hosts        255.255.255.254  (point-to-point links)
/30  = 4 hosts (2 usable)
/29  = 8 hosts (6 usable)
/28  = 16 hosts (14 usable)
/27  = 32 hosts (30 usable)
/26  = 64 hosts (62 usable)
/25  = 128 hosts (126 usable)
/24  = 256 hosts (254 usable)     ← common LAN
/23  = 512 hosts (510 usable)
/22  = 1024 hosts
/16  = 65536 hosts (65534 usable)  ← common VPC
/8   = 16.7M hosts

Formula: hosts = 2^(32-prefix) - 2 (network + broadcast)
```

### Subnet Calculation
```
IP:      192.168.1.100/26
Binary:  11000000.10101000.00000001.01100100
Mask:    11111111.11111111.11111111.11000000  (/26)

Network: 192.168.1.64    (all host bits = 0)
Broadcast: 192.168.1.127  (all host bits = 1)
Hosts:   192.168.1.65 - 192.168.1.126 (62 hosts)

Quick way: /26 = 64 addresses per subnet
  192.168.1.0/26   → 0–63
  192.168.1.64/26  → 64–127 ← our subnet
  192.168.1.128/26 → 128–191
  192.168.1.192/26 → 192–255
```

### IPv6
```
Format: 2001:0db8:85a3:0000:0000:8a2e:0370:7334
Short:  2001:db8:85a3::8a2e:370:7334 (:: = consecutive zeros)

Special addresses:
::1               loopback (127.0.0.1 equivalent)
fe80::/10         link-local (auto-assigned)
fc00::/7          unique local (private, like RFC1918)
2000::/3          global unicast (public internet)
ff00::/8          multicast

/128  = single host
/64   = standard subnet (LAN segment)
/48   = site allocation
/32   = ISP allocation
```

---

## TCP & UDP

### TCP (Transmission Control Protocol)
```
Characteristics:
  - Connection-oriented (3-way handshake)
  - Reliable (acknowledgments, retransmission)
  - Ordered delivery
  - Flow control (receiver window)
  - Congestion control (slow start, CWND)

3-Way Handshake:
  Client → SYN (seq=x)                    → Server
  Client ← SYN-ACK (seq=y, ack=x+1)      ← Server
  Client → ACK (ack=y+1)                  → Server

Connection Termination (4-way):
  Client → FIN                            → Server
  Client ← ACK                            ← Server
  Client ← FIN                            ← Server
  Client → ACK                            → Server

TCP States:
  LISTEN      waiting for connection
  SYN_SENT    sent SYN, waiting for SYN-ACK
  ESTABLISHED connection open
  FIN_WAIT_1  sent FIN, waiting for ACK
  FIN_WAIT_2  received ACK, waiting for FIN
  TIME_WAIT   sent last ACK, waiting 2*MSL before closing
  CLOSE_WAIT  received FIN, waiting for app to close
  CLOSING     both sent FIN simultaneously

# View TCP states
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
netstat -an | grep TCP | awk '{print $6}' | sort | uniq -c | sort -rn
```

### UDP (User Datagram Protocol)
```
Characteristics:
  - Connectionless
  - No reliability (fire and forget)
  - No ordering
  - Low overhead (8-byte header vs TCP's 20+)
  - Supports multicast/broadcast

Use cases: DNS, DHCP, streaming, gaming, VoIP, QUIC/HTTP3
```

### Key TCP Concepts
```
MSS (Max Segment Size):  max payload per segment (typically 1460 bytes for Ethernet)
MTU (Max Transmission Unit):  max frame size (1500 bytes for Ethernet)
Window Size:  how much data receiver can buffer (flow control)
Nagle's Algorithm:  buffers small packets (disable with TCP_NODELAY for low-latency)
Keep-alive:  periodic probes to detect dead connections
```

---

## DNS

### Record Types
```
A       hostname → IPv4 address
AAAA    hostname → IPv6 address
CNAME   hostname → hostname (alias)
MX      mail server for domain
NS      authoritative nameservers for domain
TXT     arbitrary text (SPF, DKIM, verification)
PTR     IPv4/IPv6 → hostname (reverse DNS)
SOA     start of authority (zone metadata)
SRV     service location (host + port + priority)
CAA     certificate authority authorization
```

### Resolution Process
```
Browser cache → OS cache → /etc/hosts → Recursive resolver (ISP/8.8.8.8)
  → Root nameserver (.) → TLD nameserver (.com) → Authoritative NS → A record
```

### dig
```bash
dig google.com                       basic lookup
dig google.com A                     IPv4 only
dig google.com AAAA                  IPv6
dig google.com MX                    mail servers
dig google.com NS                    nameservers
dig google.com TXT                   text records
dig google.com +short                just the answer
dig @8.8.8.8 google.com             use specific resolver
dig @1.1.1.1 google.com +trace      trace full resolution
dig -x 8.8.8.8                      reverse DNS lookup
dig google.com +noall +answer       only answer section
dig google.com +dnssec              check DNSSEC

# Useful for debugging:
dig google.com +stats               timing info
dig google.com SOA                  zone authority info
dig axfr @ns1.example.com example.com  zone transfer (if allowed)
```

### DNS Debugging
```bash
# Check /etc/resolv.conf
cat /etc/resolv.conf

# Check /etc/hosts
cat /etc/hosts

# Test DNS resolution from app
docker exec container nslookup google.com

# Check DNSSEC
dig google.com +dnssec +multi

# Propagation check
for ns in 8.8.8.8 1.1.1.1 9.9.9.9; do echo "$ns:"; dig @$ns example.com A +short; done

# SPF check (anti-spam)
dig example.com TXT | grep "v=spf"

# DMARC
dig _dmarc.example.com TXT
```

---

## HTTP & HTTPS

### HTTP Methods
```
GET     retrieve resource (safe, idempotent)
POST    create resource (neither)
PUT     replace resource (idempotent)
PATCH   partially update (not idempotent generally)
DELETE  remove resource (idempotent)
HEAD    like GET but no body (safe, idempotent)
OPTIONS preflight, discover supported methods
```

### Status Codes
```
1xx  Informational
  100 Continue
  101 Switching Protocols

2xx  Success
  200 OK
  201 Created
  204 No Content
  206 Partial Content

3xx  Redirection
  301 Moved Permanently (update bookmarks)
  302 Found (temporary)
  304 Not Modified (use cache)
  307 Temporary Redirect (keep method)
  308 Permanent Redirect (keep method)

4xx  Client Errors
  400 Bad Request
  401 Unauthorized (not authenticated)
  403 Forbidden (authenticated, no permission)
  404 Not Found
  405 Method Not Allowed
  409 Conflict
  410 Gone (permanently removed)
  422 Unprocessable Entity (validation failed)
  429 Too Many Requests (rate limited)

5xx  Server Errors
  500 Internal Server Error
  501 Not Implemented
  502 Bad Gateway (upstream error)
  503 Service Unavailable
  504 Gateway Timeout
```

### HTTP Headers
```
# Request
Host: example.com
Authorization: Bearer <token>
Authorization: Basic <base64(user:pass)>
Content-Type: application/json
Content-Length: 1234
Accept: application/json, text/html
Accept-Encoding: gzip, deflate, br
Cookie: session=abc123
User-Agent: Mozilla/5.0 ...
X-Forwarded-For: 203.0.113.1
X-Real-IP: 203.0.113.1
Origin: https://example.com
Referer: https://example.com/page

# Response
Content-Type: application/json; charset=utf-8
Content-Length: 512
Cache-Control: max-age=3600, public
Cache-Control: no-store
ETag: "abc123"
Last-Modified: Mon, 01 Jan 2024 00:00:00 GMT
Set-Cookie: session=abc; HttpOnly; Secure; SameSite=Strict
Access-Control-Allow-Origin: https://example.com
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'
```

### HTTP/2 vs HTTP/3
```
HTTP/1.1   text protocol, head-of-line blocking, 6 connections/domain
HTTP/2     binary, multiplexed (many requests per connection), header compression, server push
HTTP/3     UDP-based (QUIC), better on lossy networks, faster 0-RTT reconnect

Check version:
curl -I --http2 https://example.com
curl -I --http3 https://example.com   (if curl supports it)
```

---

## TLS/SSL

### How TLS Works
```
TLS 1.3 Handshake:
1. ClientHello: supported versions, cipher suites, random, key share
2. ServerHello: chosen version/cipher, random, key share, certificate
3. Server derives session keys
4. Client verifies certificate (checks CA chain, hostname, expiry)
5. Client derives session keys
6. Handshake complete — encrypted data flows

TLS 1.2 needs 2 round trips. TLS 1.3 needs only 1. 0-RTT possible for resumptions.
```

### Certificates
```bash
# View certificate
openssl x509 -in cert.pem -text -noout
openssl x509 -in cert.pem -noout -subject -issuer -dates

# Check remote certificate
openssl s_client -connect example.com:443
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -text -noout

# Check expiry date
echo | openssl s_client -connect example.com:443 2>/dev/null | \
  openssl x509 -noout -dates

# Verify certificate chain
openssl verify -CAfile chain.pem cert.pem

# Generate self-signed cert
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes \
  -subj "/CN=example.com"

# Generate CSR
openssl req -newkey rsa:2048 -nodes -keyout domain.key -out domain.csr \
  -subj "/C=US/ST=CA/L=SF/O=MyOrg/CN=example.com"

# Generate ED25519 key (modern)
openssl genpkey -algorithm Ed25519 -out key.pem

# Check if private key matches certificate
openssl x509 -noout -modulus -in cert.pem | md5sum
openssl rsa  -noout -modulus -in key.pem  | md5sum
# Must match!

# Test TLS version support
openssl s_client -connect example.com:443 -tls1_3
nmap --script ssl-enum-ciphers -p 443 example.com
```

### Common Certificate Formats
```
.pem    base64 encoded (most common on Linux)
.crt    usually PEM format certificate
.key    usually PEM private key
.csr    certificate signing request
.p12/.pfx  PKCS#12 — contains cert + key (Windows, Java)
.der    binary DER format

Convert:
openssl pkcs12 -in cert.p12 -out cert.pem -nodes
openssl pkcs12 -export -out cert.p12 -inkey key.pem -in cert.pem -certfile chain.pem
openssl x509 -in cert.der -inform DER -out cert.pem -outform PEM
```

---

## Load Balancing

### Algorithms
```
Round Robin           requests cycle through all servers equally
Weighted Round Robin  some servers get more traffic (capacity-aware)
Least Connections     send to server with fewest active connections
IP Hash               client IP always goes to same server (session stickiness)
Random                random selection
Least Response Time   to fastest responding server
Resource Based        to server with most available resources
```

### Layer 4 vs Layer 7
```
L4 (Transport):
  - Routes based on IP + port
  - Doesn't look at content
  - Faster (less processing)
  - Cannot make routing decisions based on HTTP headers/path
  - Examples: HAProxy TCP mode, AWS NLB

L7 (Application):
  - Routes based on HTTP headers, path, cookies, content
  - Can modify requests/responses
  - SSL termination
  - Content-based routing (/api → service A, / → service B)
  - Examples: Nginx, HAProxy HTTP mode, AWS ALB, Envoy
```

### Nginx Load Balancer
```nginx
upstream backend {
    least_conn;                    # algorithm
    server 10.0.0.1:8080 weight=3;
    server 10.0.0.2:8080 weight=1;
    server 10.0.0.3:8080 backup;  # only used when others fail

    keepalive 32;                  # connection pool to upstream
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";       # keepalive
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;

        # Health checks (Nginx Plus: active; open-source: passive)
        proxy_next_upstream error timeout invalid_header http_500 http_502;
        proxy_next_upstream_tries 3;
    }
}
```

### HAProxy
```
frontend http
    bind *:80
    bind *:443 ssl crt /etc/ssl/cert.pem
    redirect scheme https unless { ssl_fc }
    default_backend webservers

backend webservers
    balance leastconn
    option httpchk GET /health
    server web1 10.0.0.1:8080 check inter 5s rise 2 fall 3
    server web2 10.0.0.2:8080 check inter 5s rise 2 fall 3
    server web3 10.0.0.3:8080 check inter 5s rise 2 fall 3 backup
```

---

## Firewalls & iptables

### iptables concepts
```
Tables:  filter | nat | mangle | raw
Chains:  INPUT | OUTPUT | FORWARD (filter) | PREROUTING | POSTROUTING (nat)
Actions: ACCEPT | DROP | REJECT | LOG | MASQUERADE | DNAT | SNAT

Packet flow:
  Incoming → PREROUTING → (FORWARD or INPUT) → OUTPUT → POSTROUTING
```

```bash
# View rules
iptables -L -n -v --line-numbers
iptables -L INPUT -n -v
iptables -t nat -L -n -v

# Allow/block
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -j DROP         # default deny (put this last!)

# Block an IP
iptables -A INPUT -s 203.0.113.1 -j DROP
iptables -I INPUT -s 203.0.113.1 -j DROP   # insert at top (higher priority)

# Port forwarding (DNAT)
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 10.0.0.1:8080
iptables -t nat -A POSTROUTING -j MASQUERADE

# Save/restore (Debian/Ubuntu)
iptables-save > /etc/iptables/rules.v4
iptables-restore < /etc/iptables/rules.v4

# Save/restore (RHEL/CentOS)
service iptables save
iptables-restore < /etc/sysconfig/iptables
```

### ufw (simpler frontend)
```bash
ufw status verbose
ufw enable / disable
ufw allow 22
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow from 10.0.0.0/24 to any port 5432
ufw allow from 203.0.113.1
ufw deny 23
ufw delete allow 80/tcp
ufw reset                     reset all rules
ufw logging on
```

### nftables (modern iptables replacement)
```bash
nft list ruleset
nft list tables
nft add table inet filter
nft add chain inet filter input '{ type filter hook input priority 0; policy drop; }'
nft add rule inet filter input ct state established,related accept
nft add rule inet filter input iif lo accept
nft add rule inet filter input tcp dport { 22, 80, 443 } accept
```

---

## NAT & Port Forwarding

```bash
# Enable IP forwarding (required for routing/NAT)
echo 1 > /proc/sys/net/ipv4/ip_forward
sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf

# Masquerade (SNAT for internet sharing)
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Port forwarding: forward :80 to internal server
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to 192.168.1.10:8080
iptables -A FORWARD -p tcp -d 192.168.1.10 --dport 8080 -j ACCEPT

# Static SNAT (fixed IP)
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -j SNAT --to-source 203.0.113.1
```

---

## VPNs & Tunneling

### WireGuard (modern VPN)
```bash
# Server setup
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee client_private.key | wg pubkey > client_public.key

# /etc/wireguard/wg0.conf (server)
cat > /etc/wireguard/wg0.conf << 'EOF'
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <server_private_key>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <client_public_key>
AllowedIPs = 10.8.0.2/32
EOF

# Start
wg-quick up wg0
systemctl enable wg-quick@wg0

# Status
wg show

# Client config
cat > /etc/wireguard/wg0.conf << 'EOF'
[Interface]
Address = 10.8.0.2/24
PrivateKey = <client_private_key>
DNS = 1.1.1.1

[Peer]
PublicKey = <server_public_key>
Endpoint = server_ip:51820
AllowedIPs = 0.0.0.0/0   # route all traffic through VPN
PersistentKeepalive = 25
EOF
```

### SSH Tunneling
```bash
# Local forward (access remote service locally)
ssh -L 5432:db.internal:5432 user@bastion
# Now connect to localhost:5432 to reach db.internal:5432

# Remote forward (expose local service to remote)
ssh -R 8080:localhost:3000 user@remote
# Remote machine can access localhost:8080 to reach your local:3000

# Dynamic SOCKS proxy
ssh -D 1080 user@proxy-server
# Configure browser to use SOCKS5 proxy at localhost:1080

# Background + no shell
ssh -fN -L 5432:db:5432 user@bastion
```

---

## Network Tools

### curl
```bash
curl https://example.com
curl -v https://example.com              verbose
curl -I https://example.com             headers only
curl -o file.html https://example.com   save to file
curl -L https://example.com            follow redirects
curl -X POST -H "Content-Type: application/json" \
     -d '{"key":"value"}' https://api.example.com
curl -u user:pass https://example.com   basic auth
curl -H "Authorization: Bearer token" https://api.example.com
curl --cert client.crt --key client.key https://example.com
curl -k https://example.com            skip TLS verification (dev only)
curl --resolve example.com:443:1.2.3.4 https://example.com  override DNS
curl -w "\n%{http_code} %{time_total}s\n" -s -o /dev/null https://example.com
```

### tcpdump
```bash
tcpdump -i eth0                         capture on interface
tcpdump -i any                          all interfaces
tcpdump -n                              no DNS resolution
tcpdump -v                              verbose
tcpdump -X                              hex + ASCII
tcpdump -w capture.pcap                write to file
tcpdump -r capture.pcap                read from file

# Filters
tcpdump host 203.0.113.1               traffic to/from host
tcpdump src 203.0.113.1               only from host
tcpdump dst 203.0.113.1               only to host
tcpdump port 443                       specific port
tcpdump port 80 or port 443           multiple ports
tcpdump 'tcp[tcpflags] & tcp-syn != 0'  SYN packets
tcpdump 'tcp[tcpflags] == tcp-rst'    RST packets (connection resets)

# Real examples
tcpdump -i eth0 -n port 80 -X | head -100    HTTP traffic
tcpdump -i eth0 -n 'host 8.8.8.8 and port 53'  DNS to Google
tcpdump -i eth0 -w /tmp/debug.pcap port 443  capture HTTPS for later analysis
```

### nmap
```bash
nmap 192.168.1.0/24                    ping sweep + basic scan
nmap -p 80,443,8080 192.168.1.1       specific ports
nmap -p- 192.168.1.1                  all 65535 ports
nmap -sV 192.168.1.1                  version detection
nmap -sV -sC 192.168.1.1             version + default scripts
nmap -O 192.168.1.1                   OS detection
nmap -A 192.168.1.1                   aggressive (OS, version, scripts)
nmap -sU -p 53,161 192.168.1.1       UDP scan
nmap -Pn 192.168.1.1                 skip ping (host up assumed)
nmap --script ssl-cert 192.168.1.1 -p 443  check TLS cert
nmap --script http-title 192.168.1.0/24   get page titles
```

### Other Essential Tools
```bash
# Connectivity
ping -c 4 8.8.8.8
ping -i 0.2 -c 20 host              rapid ping
traceroute 8.8.8.8
mtr --report 8.8.8.8               real-time traceroute report
pathping 8.8.8.8                   Windows equivalent

# Bandwidth
iperf3 -s                           server mode
iperf3 -c server_ip                 TCP throughput test
iperf3 -c server_ip -u -b 100M     UDP test at 100Mbps

# Socket statistics
ss -tlnp                            listening TCP
ss -tlnpu                           listening TCP + UDP
ss -tnp state established           established connections
ss -s                               summary statistics
ss -tnp 'dport = :443'             connections to port 443

# Routing
ip route get 8.8.8.8               which route is used
ip route add 10.0.0.0/8 via 192.168.1.1   add route
ip route del 10.0.0.0/8            delete route
route -n                            routing table

# ARP
arp -n                              ARP cache
ip neighbor                         modern ARP
arping 192.168.1.1                 ARP ping

# Network namespace (Linux)
ip netns list
ip netns add myns
ip netns exec myns ip addr
```

---

## Common Ports

```
20/21   FTP data/control
22      SSH
23      Telnet (insecure — disable)
25      SMTP
53      DNS
67/68   DHCP
80      HTTP
110     POP3
143     IMAP
443     HTTPS
465/587 SMTP with TLS
636     LDAPS
993     IMAPS
995     POP3S
3306    MySQL
5432    PostgreSQL
6379    Redis
27017   MongoDB
2181    ZooKeeper
5601    Kibana
9200    Elasticsearch
9090    Prometheus
3000    Grafana (default)
8080    Common alt HTTP
8443    Common alt HTTPS
2375    Docker daemon (unencrypted — never expose!)
2376    Docker daemon (TLS)
6443    Kubernetes API
10250   kubelet
2379    etcd client
2380    etcd peer
4789    VXLAN
8472    Flannel VXLAN
179     BGP
```

---

## Performance Tuning

### Kernel TCP Settings
```bash
# View current settings
sysctl -a | grep net.ipv4.tcp

# Time Wait — speed up connection reuse
echo 1 > /proc/sys/net/ipv4/tcp_tw_reuse
# Or: sysctl -w net.ipv4.tcp_tw_reuse=1

# Connection backlog (for busy servers)
sysctl -w net.core.somaxconn=65535
sysctl -w net.ipv4.tcp_max_syn_backlog=65535

# File descriptors (connections)
ulimit -n 1000000
echo "* soft nofile 1000000" >> /etc/security/limits.conf
echo "* hard nofile 1000000" >> /etc/security/limits.conf

# Buffer sizes (for high-throughput)
sysctl -w net.core.rmem_max=134217728     # 128MB
sysctl -w net.core.wmem_max=134217728
sysctl -w net.ipv4.tcp_rmem="4096 87380 134217728"
sysctl -w net.ipv4.tcp_wmem="4096 65536 134217728"

# BBR congestion control (Linux 4.9+, highly recommended)
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p
sysctl net.ipv4.tcp_congestion_control    # verify

# Persist settings
cat >> /etc/sysctl.d/99-network-performance.conf << 'EOF'
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_fin_timeout = 15
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
EOF
sysctl -p /etc/sysctl.d/99-network-performance.conf
```

---

## Troubleshooting Methodology

### The 7-Step Method
```
1. Define the problem     → what exactly is failing?
2. Gather information     → logs, metrics, packet captures
3. Consider causes        → hardware, software, config, external
4. Formulate hypothesis   → most likely cause
5. Test hypothesis        → isolate, test one thing at a time
6. Fix + verify           → apply fix, confirm problem solved
7. Document               → what happened, what fixed it
```

### Layered Troubleshooting (bottom-up)
```bash
# Layer 1-2: Physical/Link
ip link show                    is the interface up?
ethtool eth0                    cable, speed, duplex
ip -s link show eth0            packet counts, errors, drops

# Layer 3: Network
ping 192.168.1.1                can you reach the gateway?
ip route show                   routing table correct?
arp -n                          ARP resolving correctly?

# Layer 4: Transport
nc -zv host port                can you connect?
ss -tlnp | grep port            is the service listening?
curl -v http://host:port        HTTP connectivity

# Layer 7: Application
curl -v https://api.example.com  full request trace
openssl s_client -connect host:443  TLS handshake
dig host                         DNS resolution
journalctl -u service -f         application logs
```

### Common Problems
```
Can't connect to host:
  → ping works? (L3 OK)
  → traceroute stops where? (find the hop)
  → port listening? (ss -tlnp)
  → firewall blocking? (iptables, ufw, security groups)
  → DNS resolving? (dig hostname)

Intermittent drops:
  → mtr --report — shows packet loss per hop
  → check interface errors: ip -s link show eth0
  → check for retransmits: ss -i | grep retrans

Slow:
  → bandwidth: iperf3 test
  → latency: ping, mtr
  → TCP retransmits: ss -ti
  → DNS slow? (dig shows query time)
  → traceroute: which hop is slow?

High latency:
  → check MTU: ping -M do -s 1472 host (fragmentation?)
  → check for bufferbloat: mtr + simultaneous download
  → enable BBR congestion control
  → check for half-duplex negotiation: ethtool eth0
```
