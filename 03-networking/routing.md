````markdown
# BLACKFORGE Routing

This document explains how traffic is routed between the BLACKFORGE networks.

Routing is a core part of the BLACKFORGE architecture because the DMZ (Demilitarized Zone) server performs several networking roles:

- Router
- NAT (Network Address Translation) gateway
- Firewall
- WireGuard VPN endpoint
- Reverse proxy

---

# Network Layout

BLACKFORGE uses the following networks:

```text
VPC:
10.50.0.0/16

DMZ:
10.50.10.0/24

Private:
10.50.20.0/24

WireGuard:
10.50.30.0/24
````

The DMZ and Private networks are AWS subnets.

The WireGuard network is a logical VPN (Virtual Private Network) network.

---

# AWS Route Tables

## DMZ Route Table

The DMZ subnet uses:

```text
10.50.0.0/16
    → local

0.0.0.0/0
    → Internet Gateway
```

This provides:

* Communication with other resources inside the VPC
* Internet access for the DMZ

---

# Private Route Table

The Private subnet uses:

```text
10.50.0.0/16
    → local

0.0.0.0/0
    → BLACKFORGE-DMZ-01
```

The important part is the default route.

Instead of sending Internet-bound traffic directly to an Internet Gateway, the private subnet sends it toward the DMZ instance.

---

# Private → Internet

The complete traffic path is:

```text
Private Server
10.50.20.32
       │
       ▼
Private Route Table
       │
       ▼
DMZ
10.50.10.116
       │
       ▼
nftables
       │
       ▼
NAT
       │
       ▼
Internet Gateway
       │
       ▼
Internet
```

This allows the private server to access the Internet without having a public IP address.

---

# IPv4 Forwarding

The Linux operating system on the DMZ must be able to forward packets between interfaces.

This is controlled by IPv4 (Internet Protocol version 4) forwarding.

Verification:

```bash
sysctl net.ipv4.ip_forward
```

Result:

```text
net.ipv4.ip_forward = 1
```

A value of `1` means IPv4 forwarding is enabled.

---

# Persistent IPv4 Forwarding

The setting was made persistent using:

```text
/etc/sysctl.d/99-blackforge-router.conf
```

Configuration:

```text
net.ipv4.ip_forward=1
```

This ensures forwarding remains enabled after a reboot.

---

# DMZ Routing

The DMZ was tested to determine how it reaches the private server.

Command:

```bash
ip route get 10.50.20.32
```

Result:

```text
10.50.20.32 via 10.50.10.1 dev ens5 src 10.50.10.116
```

This means the DMZ sends traffic destined for the private server through:

```text
Gateway:
10.50.10.1

Interface:
ens5

Source:
10.50.10.116
```

---

# Private Server Routing

The private server's routing table included:

```text
default via 10.50.20.1 dev ens5
```

and:

```text
10.50.20.0/24 dev ens5
```

This means the private server uses:

```text
10.50.20.1
```

as its AWS subnet gateway for traffic outside its local subnet.

---

# Private → Internet NAT

The DMZ performs NAT for private-subnet traffic.

The nftables rule is:

```text
ip saddr 10.50.20.0/24
oifname "ens5"
masquerade
```

Conceptually:

```text
10.50.20.32
     │
     │ Private source address
     ▼
DMZ
     │
     │ Source NAT
     ▼
10.50.10.116
     │
     ▼
Internet
```

The external destination sees the DMZ's public address rather than the private server's internal address.

---

# NAT Verification

The private server was tested with:

```bash
curl -4 https://ifconfig.me
```

The observed public address was:

```text
13.229.212.104
```

This demonstrated that the private server's Internet traffic was exiting through the DMZ.

---

# WireGuard Routing

WireGuard provides the management network:

```text
10.50.30.0/24
```

Endpoints:

```text
DMZ:
10.50.30.1

Kali:
10.50.30.2
```

Kali has routes for:

```text
10.50.30.0/24
```

and:

```text
10.50.20.0/24
```

through:

```text
wg0
```

---

# Kali → Private Routing

The intended path is:

```text
Kali
10.50.30.2
      │
      │ wg0
      ▼
DMZ
10.50.30.1
      │
      │ forwarding
      ▼
Private
10.50.20.32
```

This allows Kali to reach the private server without giving the private server a public IP.

---

# WireGuard → Private Forwarding

Traffic entering the DMZ through:

```text
wg0
```

must be forwarded toward:

```text
ens5
```

The packet flow is:

```text
Kali
10.50.30.2
      │
      ▼
wg0
      │
      ▼
DMZ
      │
      ▼
nftables
      │
      ▼
ens5
      │
      ▼
10.50.20.32
```

---

# WireGuard → Private NAT

The DMZ also performs source NAT for traffic from the WireGuard network to the private subnet.

The rule is:

```text
Source:
10.50.30.0/24

Destination:
10.50.20.0/24

Interface:
ens5

Action:
masquerade
```

This causes the private server to see the traffic as originating from the DMZ rather than directly from the WireGuard address.

This works with the existing AWS Security Group trust between the DMZ and Private server.

---

# nftables Forwarding

The DMZ uses nftables as the Linux firewall.

The forwarding chain uses:

```text
policy drop
```

This creates a default-deny forwarding model.

Traffic must match an allowed rule before it can be forwarded.

Relevant paths include:

```text
Private → Internet
Internet → Private return traffic
WireGuard → Private
Private → WireGuard return traffic
```

---

# Routing and Firewalling

Routing and firewalling are separate functions.

Routing answers:

```text
Where should the packet go?
```

Firewalling answers:

```text
Should the packet be allowed?
```

Therefore:

```text
Correct Route
      +
Allowed Firewall Traffic
      +
Allowed Security Group Traffic
      +
Listening Service
      =
Successful Connection
```

A correct route alone does not guarantee connectivity.

---

# Troubleshooting Example

During WireGuard testing, Kali attempted to reach:

```text
10.50.20.32
```

through:

```text
10.50.30.2
```

The first question was whether the packet actually entered the DMZ.

Packet capture on WireGuard:

```bash
sudo tcpdump -ni wg0 host 10.50.20.32
```

showed:

```text
10.50.30.2 > 10.50.20.32: ICMP echo request
```

This confirmed that the packet entered the DMZ through WireGuard.

---

# Checking the AWS Interface

Traffic was then observed on:

```bash
sudo tcpdump -ni ens5 host 10.50.20.32
```

The request was seen leaving the DMZ toward the private server.

Therefore:

```text
Kali
  ↓
WireGuard
  ↓
DMZ
  ↓
ens5
  ↓
Private
```

was functioning.

---

# Why the Ping Still Failed

No ICMP (Internet Control Message Protocol) reply was observed.

This did not necessarily mean the routing path was completely broken.

The private Security Group did not allow ICMP, while TCP/22 was permitted.

Therefore the actual SSH service was tested instead.

```bash
nc -vz -w 5 10.50.20.32 22
```

Result:

```text
10.50.20.32:22 open
```

SSH connectivity was then successfully established.

---

# Important AWS Lesson

The WireGuard network:

```text
10.50.30.0/24
```

is not an AWS subnet.

It therefore should not be treated as an AWS subnet in the AWS route table.

The AWS route table handles the real VPC networks:

```text
10.50.10.0/24
10.50.20.0/24
```

The DMZ Linux host handles routing for the logical WireGuard network.

---

# Complete Routing Model

```text
                         INTERNET
                            │
                            ▼
                    Internet Gateway
                            │
                            ▼
                ┌────────────────────┐
                │        DMZ         │
                │   10.50.10.0/24    │
                │                    │
                │ 10.50.10.116      │
                │                    │
                │ IPv4 Forwarding   │
                │ nftables          │
                │ NAT               │
                └─────────┬──────────┘
                          │
              ┌───────────┴───────────┐
              │                       │
              ▼                       ▼
        PRIVATE NETWORK          WIREGUARD
        10.50.20.0/24            10.50.30.0/24
              │                       │
              ▼                       ▼
        10.50.20.32              10.50.30.2
```

---

# Verification Commands

## Check routing table

```bash
ip route
```

---

## Check route to private server

```bash
ip route get 10.50.20.32
```

---

## Check IPv4 forwarding

```bash
sysctl net.ipv4.ip_forward
```

---

## Check nftables rules

```bash
sudo nft list ruleset
```

---

## Capture private traffic

```bash
sudo tcpdump -ni ens5 host 10.50.20.32
```

---

## Test SSH connectivity

```bash
nc -vz -w 5 10.50.20.32 22
```

---

# Routing Lessons Learned

## 1. AWS and Linux routing work together

AWS determines how traffic moves between VPC resources.

Linux routing determines how the DMZ handles traffic between its interfaces.

Both layers are required.

---

## 2. The DMZ is the central network control point

The DMZ performs:

```text
Routing
    +
Forwarding
    +
NAT
    +
Firewalling
    +
VPN termination
```

This makes it the central security and routing point of BLACKFORGE.

---

## 3. NAT can solve return-path and trust problems

The WireGuard → Private NAT rule causes the private server to see the traffic as coming from the trusted DMZ address.

This works with the existing Security Group design.

---

## 4. Packet capture is valuable when routing is unclear

Instead of assuming where a packet went:

```text
Capture it.
```

Observe:

```text
Ingress interface
       ↓
Forwarding
       ↓
Egress interface
       ↓
Destination
       ↓
Return traffic
```

---

# Verification Status

```text
AWS Route Tables             ✅
DMZ Routing                  ✅
Private Routing              ✅
IPv4 Forwarding              ✅
Private → Internet NAT       ✅
WireGuard Routing            ✅
WireGuard → Private NAT      ✅
nftables Forwarding          ✅
Packet Flow Verification     ✅
SSH Connectivity             ✅
```

---
