````markdown
# BLACKFORGE IP Addressing

This document defines the IP addressing scheme used throughout the BLACKFORGE cybersecurity laboratory.

IP addressing provides the foundation for communication between the DMZ (Demilitarized Zone), private workloads, and the WireGuard management network.

---

# Addressing Overview

BLACKFORGE uses the following networks:

| Network | CIDR | Purpose |
|---|---|---|
| BLACKFORGE VPC | `10.50.0.0/16` | Overall AWS network |
| DMZ | `10.50.10.0/24` | Public-facing infrastructure |
| Private | `10.50.20.0/24` | Internal workloads |
| WireGuard | `10.50.30.0/24` | Management VPN |

---

# VPC

```text
Name:
BLACKFORGE-VPC

CIDR:
10.50.0.0/16
````

The VPC (Virtual Private Cloud) is the main AWS network boundary for BLACKFORGE.

The `10.50.0.0/16` address space contains the AWS subnets used by the lab.

---

# DMZ Network

```text
Name:
BLACKFORGE-DMZ

CIDR:
10.50.10.0/24
```

The DMZ contains the public-facing infrastructure.

Primary server:

```text
BLACKFORGE-DMZ-01

Private IP:
10.50.10.116

Public IP:
13.229.212.104
```

The private IP is used for communication inside the AWS VPC.

The public IP provides Internet-facing access to the services intentionally exposed by the DMZ.

Primary roles include:

* Nginx reverse proxy
* ModSecurity WAF (Web Application Firewall)
* OWASP CRS (Core Rule Set)
* WireGuard VPN endpoint
* Router
* NAT (Network Address Translation)
* nftables firewall

---

# Private Network

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

The private server intentionally has no public IP address.

It contains the internal application workload:

```text
10.50.20.32:3000
```

which hosts OWASP Juice Shop.

The application is accessed through the DMZ rather than directly from the Internet.

---

# WireGuard Management Network

```text
CIDR:
10.50.30.0/24
```

The WireGuard network is a logical VPN (Virtual Private Network) network.

It is not an AWS subnet.

The two primary endpoints are:

```text
Kali Linux:
10.50.30.2

DMZ:
10.50.30.1
```

The network is used for secure administrative access.

---

# Complete Addressing Layout

```text
BLACKFORGE-VPC
10.50.0.0/16
│
├── DMZ
│   10.50.10.0/24
│   │
│   └── BLACKFORGE-DMZ-01
│       Private: 10.50.10.116
│       Public:  13.229.212.104
│
├── PRIVATE
│   10.50.20.0/24
│   │
│   └── BLACKFORGE-PRIVATE-01
│       Private: 10.50.20.32
│       Public:  None
│
└── WIREGUARD
    10.50.30.0/24
    │
    ├── DMZ
    │   10.50.30.1
    │
    └── Kali
        10.50.30.2
```

---

# Application Addressing

OWASP Juice Shop runs on the private server:

```text
10.50.20.32:3000
```

The application is not directly exposed to the Internet.

The public traffic path is:

```text
Internet
    ↓
13.229.212.104:80
    ↓
Nginx
    ↓
10.50.20.32:3000
```

This keeps the application inside the private network while allowing Nginx to act as the controlled public entry point.

---

# Management Addressing

Administrative access uses the WireGuard network.

Kali:

```text
10.50.30.2
```

DMZ:

```text
10.50.30.1
```

The management path is:

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

This separates management traffic from normal public application traffic.

---

# Why Different Networks Are Used

Each network has a specific security purpose.

## DMZ

```text
10.50.10.0/24
```

Used for:

* Internet-facing services
* Reverse proxy
* WAF
* VPN endpoint
* Routing
* NAT

---

## Private

```text
10.50.20.0/24
```

Used for:

* Internal workloads
* Applications
* Vulnerable targets
* Services that should not have direct Internet exposure

---

## WireGuard

```text
10.50.30.0/24
```

Used for:

* Administrative access
* Secure management
* SSH access to the DMZ
* SSH ProxyJump access to the private server

---

# Addressing and Security Boundaries

The network design creates clear boundaries:

```text
                    INTERNET
                       │
                       ▼
                10.50.10.0/24
                     DMZ
                       │
                       ▼
                10.50.20.0/24
                    PRIVATE
```

Management uses a separate logical network:

```text
10.50.30.0/24
     │
     ▼
WireGuard
     │
     ▼
DMZ / Private
```

This separation makes it easier to control and monitor different types of traffic.

---

# Routing Verification

The DMZ was tested to determine how it would reach the private server:

```bash
ip route get 10.50.20.32
```

The result showed:

```text
10.50.20.32 via 10.50.10.1 dev ens5 src 10.50.10.116
```

This confirms that the DMZ uses its AWS network interface to reach the private subnet.

---

# Private Server Routing

The private server uses the AWS subnet gateway for traffic leaving its local subnet.

The default route was observed as:

```text
default via 10.50.20.1 dev ens5
```

The local subnet route was:

```text
10.50.20.0/24 dev ens5
```

This confirms that the private server is correctly connected to the AWS private subnet.

---

# WireGuard Routing Verification

Kali has a route for the WireGuard network:

```text
10.50.30.0/24 dev wg0
```

and a route for the private network:

```text
10.50.20.0/24 dev wg0
```

This allows Kali to send private-subnet traffic through the WireGuard interface toward the DMZ.

---

# Important Networking Lesson

An IP address alone does not guarantee connectivity.

Successful communication depends on multiple components:

```text
IP Address
    +
Route
    +
Firewall
    +
Security Group
    +
Service Listener
    =
Successful Connection
```

For example, the private server can have:

```text
10.50.20.32
```

and still reject a connection if:

* The route is incorrect
* The Security Group blocks it
* nftables blocks it
* The service is not listening
* The host firewall blocks it

---

# Public vs Private Addressing

BLACKFORGE intentionally separates public and private addressing.

```text
DMZ

Private IP:
10.50.10.116

Public IP:
13.229.212.104
```

The private server has:

```text
Private IP:
10.50.20.32

Public IP:
None
```

This reduces the direct attack surface of the internal workload.

---

# Verification Summary

```text
VPC CIDR                    10.50.0.0/16
DMZ CIDR                    10.50.10.0/24
DMZ Private IP              10.50.10.116
DMZ Public IP               13.229.212.104

Private CIDR                10.50.20.0/24
Private Server IP           10.50.20.32
Private Public IP           None

WireGuard CIDR              10.50.30.0/24
DMZ WireGuard IP            10.50.30.1
Kali WireGuard IP           10.50.30.2
```

---

# Status

```text
VPC Addressing              ✅
DMZ Addressing              ✅
Private Addressing          ✅
WireGuard Addressing        ✅
Application Addressing      ✅
Management Addressing       ✅
Routing Verification        ✅
Network Separation          ✅
```
