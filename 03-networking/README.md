
# BLACKFORGE Networking

This directory documents the networking architecture, addressing, routing, packet analysis, and traffic flow used throughout the BLACKFORGE cybersecurity laboratory.

Networking is the foundation of BLACKFORGE.

The lab intentionally separates public-facing infrastructure, private workloads, and administrative access so that offensive and defensive security scenarios can be tested across multiple network boundaries.

---

## Network Architecture

BLACKFORGE uses three logical networks:

```text
DMZ Network
10.50.10.0/24

Private Network
10.50.20.0/24

WireGuard Management Network
10.50.30.0/24
````

The first two are AWS subnet CIDRs (Classless Inter-Domain Routing blocks).

The WireGuard network is a VPN (Virtual Private Network) network and is not an AWS subnet.

---

## Network Topology

```text
                           INTERNET
                               │
                               │
                     Internet Gateway
                               │
                               ▼
                 ┌─────────────────────────┐
                 │          DMZ            │
                 │    10.50.10.0/24        │
                 │                         │
                 │ BLACKFORGE-DMZ-01       │
                 │ 10.50.10.116            │
                 │ Public: 13.229.212.104  │
                 │                         │
                 │ Nginx                   │
                 │ ModSecurity             │
                 │ OWASP CRS               │
                 │ WireGuard               │
                 │ nftables                │
                 │ Routing / NAT           │
                 └────────────┬────────────┘
                              │
                              │
                              ▼
                 ┌─────────────────────────┐
                 │        PRIVATE          │
                 │    10.50.20.0/24        │
                 │                         │
                 │ BLACKFORGE-PRIVATE-01   │
                 │ 10.50.20.32             │
                 │ No Public IP            │
                 │                         │
                 │ Docker                  │
                 │ OWASP Juice Shop        │
                 └─────────────────────────┘


                    MANAGEMENT NETWORK

                 ┌─────────────────────────┐
                 │       WireGuard         │
                 │      10.50.30.0/24       │
                 └────────────┬────────────┘
                              │
                ┌─────────────┴─────────────┐
                │                           │
                ▼                           ▼
           Kali Linux                    DMZ
           10.50.30.2                   10.50.30.1
```

---

# Network Components

## BLACKFORGE-VPC

```text
CIDR:
10.50.0.0/16
```

The VPC (Virtual Private Cloud) provides the overall AWS network boundary.

---

## DMZ

```text
Name:
BLACKFORGE-DMZ

CIDR:
10.50.10.0/24
```

The DMZ (Demilitarized Zone) contains the public-facing infrastructure.

Primary server:

```text
BLACKFORGE-DMZ-01

Private IP:
10.50.10.116

Public IP:
13.229.212.104
```

Roles:

* Nginx reverse proxy
* ModSecurity WAF (Web Application Firewall)
* OWASP CRS (Core Rule Set)
* WireGuard VPN endpoint
* IPv4 router
* NAT (Network Address Translation)
* nftables firewall

---

## Private Network

```text
Name:
BLACKFORGE-PRIVATE

CIDR:
10.50.20.0/24
```

Primary server:

```text
BLACKFORGE-PRIVATE-01

Private IP:
10.50.20.32

Public IP:
None
```

Roles:

* Docker host
* OWASP Juice Shop
* Internal application workload
* Vulnerable target

---

## WireGuard Management Network

```text
CIDR:
10.50.30.0/24
```

Endpoints:

```text
Kali:
10.50.30.2

DMZ:
10.50.30.1
```

This network is used for administrative access.

---

# Traffic Plan

## Public Application Traffic

```text
Internet
   ↓
DMZ
   ↓
Nginx
   ↓
ModSecurity
   ↓
OWASP CRS
   ↓
10.50.20.32:3000
```

---

## Administrative Traffic

```text
Kali
10.50.30.2
   ↓
WireGuard
   ↓
DMZ
10.50.30.1
   ↓
SSH
   ↓
Private Server
10.50.20.32
```

---

## Private Internet Traffic

```text
Private Server
10.50.20.32
      ↓
DMZ
10.50.10.116
      ↓
NAT
      ↓
Internet
```

The private server can access the Internet without requiring a public IP.

---

# Network Security Model

BLACKFORGE uses multiple layers of network security:

```text
AWS Security Groups
        ↓
Network Segmentation
        ↓
nftables
        ↓
Nginx
        ↓
ModSecurity
        ↓
OWASP CRS
        ↓
Application
```

This provides defense in depth.

---

# Key Networking Concepts Practiced

The BLACKFORGE networking phase covers:

* IPv4 addressing
* CIDR
* Subnetting
* AWS routing
* Linux routing
* IPv4 forwarding
* NAT
* Firewall forwarding
* Security Groups
* WireGuard
* VPN routing
* SSH ProxyJump
* Packet capture
* TCP connectivity testing
* Network troubleshooting

---

# Important Lessons

## 1. A private IP does not mean isolated from all communication

A private subnet can still communicate through explicitly configured routes and security controls.

---

## 2. Routing and firewalling are different

Routing determines:

```text
Where should the packet go?
```

Firewall rules determine:

```text
Should the packet be allowed?
```

Both must work for communication to succeed.

---

## 3. Ping is not a complete connectivity test

ICMP (Internet Control Message Protocol) may be blocked while TCP (Transmission Control Protocol) services remain reachable.

For example:

```bash
nc -vz -w 5 10.50.20.32 22
```

confirmed TCP/22 was reachable even when ICMP testing was not useful.

---

## 4. Packet capture is extremely useful

`tcpdump` was used to determine where traffic stopped.

Example:

```bash
sudo tcpdump -ni wg0 host 10.50.20.32
```

and:

```bash
sudo tcpdump -ni ens5 host 10.50.20.32
```

This allowed packet movement to be observed across interfaces.

---

# Verification Status

```text
VPC                         ✅
DMZ subnet                  ✅
Private subnet              ✅
IP addressing               ✅
Routing                     ✅
IPv4 forwarding             ✅
Private → Internet NAT      ✅
WireGuard                   ✅
VPN routing                 ✅
nftables                    ✅
SSH through VPN             ✅
Packet capture              ✅
Private application path    ✅
```