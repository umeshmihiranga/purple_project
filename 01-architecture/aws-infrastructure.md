

````markdown
# BLACKFORGE AWS Infrastructure

## Status

**Phase:** Cloud Architecture  
**Status:** Foundation complete  
**Region:** AWS Asia Pacific (Singapore) — `ap-southeast-1`

## VPC

| Component | Value |
|---|---|
| Name | `BLACKFORGE-VPC` |
| CIDR | `10.50.0.0/16` |
| Region | `ap-southeast-1` |

## Subnets

### DMZ

```text
Name: BLACKFORGE-DMZ
CIDR: 10.50.10.0/24
Role: Public/edge network
````

### Private

```text
Name: BLACKFORGE-PRIVATE
CIDR: 10.50.20.0/24
Role: Internal workload network
```

The private subnet has no direct Internet Gateway route.

## EC2 Instances

### BLACKFORGE-DMZ-01

```text
Instance type: t3.micro
Private IPv4: 10.50.10.116
Public IPv4: 13.229.212.104
Subnet: BLACKFORGE-DMZ
```

Role:

* Public DMZ presence
* Routing
* NAT
* Future reverse proxy
* Future WAF

### BLACKFORGE-PRIVATE-01

```text
Instance type: t3.micro
Private IPv4: 10.50.20.32
Public IPv4: NONE
Subnet: BLACKFORGE-PRIVATE
```

Role:

* Internal applications
* OWASP Juice Shop
* Backend services
* Internal targets

## Traffic Model

```text
Internet ──> DMZ                  Allowed as required
DMZ ───────> Private              Controlled internal traffic
Private ───> Internet             Through DMZ NAT
Internet ──> Private              No direct public access
```

## Architecture

```text
                         INTERNET
                            |
                            v
                +----------------------+
                | BLACKFORGE-DMZ-01    |
                | 10.50.10.116         |
                | Public: 13.229.212.104
                |                      |
                | Routing + NAT        |
                +----------+-----------+
                           |
                    BLACKFORGE-VPC
                     10.50.0.0/16
                           |
                           v
                +----------------------+
                | BLACKFORGE-PRIVATE-01|
                | 10.50.20.32          |
                | NO PUBLIC IP          |
                +----------------------+
```

````

---

### `01-cloud-architecture/security-groups.md`

```markdown
# BLACKFORGE Security Groups

## Purpose

Security groups provide instance-level traffic filtering in addition to subnet and route-table segmentation.

The goal is least-privilege access.

## BLACKFORGE-DMZ-SG

```text
Name: BLACKFORGE-DMZ-SG
VPC: BLACKFORGE-VPC
````

The DMZ security group protects the public-facing edge instance.

Current known inbound rule:

```text
TCP 80
Source: 0.0.0.0/0
Purpose: Public HTTP
```

Administrative access should not be unnecessarily exposed to the Internet.

Future management access will use WireGuard.

## BLACKFORGE-PRIVATE-SG

The private instance must remain inaccessible directly from the public Internet.

Target policy:

```text
Inbound:
- Required application traffic from DMZ
- SSH through the management path
- No unrestricted Internet inbound access
```

Outbound Internet access is provided through the DMZ/NAT path.

## Validation

* [x] Private instance has no public IPv4
* [x] Private subnet does not directly use an Internet Gateway
* [x] Private outbound access works through DMZ NAT
* [ ] Verify Internet -> private host is blocked
* [ ] Finalize private security-group rules
* [ ] Restrict administrative access to WireGuard

````

---

### `03-networking/ip-addressing.md`

```markdown
# BLACKFORGE IP Addressing

## VPC

```text
BLACKFORGE-VPC
10.50.0.0/16
````

## Subnet Allocation

```text
10.50.10.0/24    BLACKFORGE-DMZ
10.50.20.0/24    BLACKFORGE-PRIVATE
```

## Hosts

### DMZ

```text
BLACKFORGE-DMZ-01
10.50.10.116/24
Public IPv4: 13.229.212.104
```

### Private

```text
BLACKFORGE-PRIVATE-01
10.50.20.32/24
Public IPv4: NONE
```

## Default Gateways

DMZ:

```text
default via 10.50.10.1
```

Private:

```text
default via 10.50.20.1
```

## Why Separate Subnets?

The DMZ and private networks use different CIDRs:

```text
DMZ     10.50.10.0/24
PRIVATE 10.50.20.0/24
```

Traffic between them is routed at Layer 3.

This allows route tables and security controls to define permitted traffic paths.

## Validation

Run:

```bash
ip -br addr
ip route
```

on both hosts.

Expected private address:

```text
10.50.20.32/24
```

The private instance has no public IPv4 address.

````

---

### `03-networking/routing.md`

```markdown
# BLACKFORGE Routing

## BLACKFORGE-DMZ-RT

Associated subnet:

```text
BLACKFORGE-DMZ
10.50.10.0/24
````

Routes:

```text
10.50.0.0/16  -> local
0.0.0.0/0     -> Internet Gateway
```

The DMZ therefore has Internet connectivity.

## BLACKFORGE-PRIVATE-RT

Associated subnet:

```text
BLACKFORGE-PRIVATE
10.50.20.0/24
```

Routes:

```text
10.50.0.0/16  -> local
0.0.0.0/0     -> BLACKFORGE-DMZ-01
```

The private subnet does not send its default route directly to the Internet Gateway.

## Private Outbound Traffic

```text
PRIVATE EC2
10.50.20.32
      |
      v
PRIVATE-RT
      |
      v
DMZ EC2
10.50.10.116
      |
      v
nftables NAT
      |
      v
Internet Gateway
      |
      v
Internet
```

## Linux IP Forwarding

On the DMZ host:

```bash
sysctl net.ipv4.ip_forward
```

Result:

```text
net.ipv4.ip_forward = 1
```

Persistent configuration:

```text
/etc/sysctl.d/99-blackforge-router.conf
```

Contents:

```text
net.ipv4.ip_forward=1
```

Applied with:

```bash
sudo sysctl --system
```

## EC2 Source/Destination Check

Source/destination checking was disabled on:

```text
BLACKFORGE-DMZ-01
```

This allows the instance to forward traffic for the private host.

## NAT

The DMZ uses nftables masquerading for:

```text
10.50.20.0/24
```

The private host's outbound traffic is translated through the DMZ public IP.

## Verification

From the private host:

```bash
curl -4 https://ifconfig.me
```

Observed:

```text
13.229.212.104
```

Therefore:

```text
PRIVATE -> DMZ -> NAT -> INTERNET
```

is working.

````

---

### `notes/lab-journal/2026-09-05-segmentation-baseline.md`

```markdown
# BLACKFORGE — Segmentation Baseline

## Date

2026-09-05

## Phase

BLACKFORGE Cloud Architecture — Network Segmentation

## Objective

Build a VPC architecture with a public DMZ subnet and an isolated private subnet.

The private host must have no public IPv4 address while retaining controlled outbound Internet access.

## Environment

### AWS

```text
Region: ap-southeast-1
VPC: BLACKFORGE-VPC
CIDR: 10.50.0.0/16
````

### DMZ

```text
BLACKFORGE-DMZ
10.50.10.0/24

BLACKFORGE-DMZ-01
Private: 10.50.10.116
Public: 13.229.212.104
```

### Private

```text
BLACKFORGE-PRIVATE
10.50.20.0/24

BLACKFORGE-PRIVATE-01
Private: 10.50.20.32
Public: NONE
```

## Initial State

The previous Lightsail environment had both servers in the same private address space.

Testing showed direct private connectivity.

That environment was useful as a baseline but did not provide the desired DMZ/private subnet architecture.

BLACKFORGE was therefore rebuilt using an Amazon VPC with separate subnets.

## Architecture

```text
                         INTERNET
                            |
                            v
                  +-------------------+
                  |       DMZ         |
                  | 10.50.10.0/24     |
                  |                   |
                  | BLACKFORGE-DMZ-01 |
                  | 10.50.10.116      |
                  +---------+---------+
                            |
                       VPC routing
                            |
                  +---------v---------+
                  |      PRIVATE      |
                  | 10.50.20.0/24     |
                  |                   |
                  | BLACKFORGE-       |
                  | PRIVATE-01        |
                  | 10.50.20.32       |
                  | NO PUBLIC IP       |
                  +-------------------+
```

## Routing

DMZ:

```text
10.50.0.0/16 -> local
0.0.0.0/0    -> Internet Gateway
```

Private:

```text
10.50.0.0/16 -> local
0.0.0.0/0    -> BLACKFORGE-DMZ-01
```

## IP Forwarding

Enabled on DMZ:

```bash
sysctl net.ipv4.ip_forward
```

Result:

```text
net.ipv4.ip_forward = 1
```

Persistent configuration:

```text
/etc/sysctl.d/99-blackforge-router.conf
```

## nftables NAT

The DMZ host was configured with a dedicated nftables table.

Relevant configuration:

```text
table ip blackforge {
        chain forward {
                type filter hook forward priority filter; policy drop;
                iifname "ens5" oifname "ens5" ip saddr 10.50.20.0/24 ct state established,related,new accept
                iifname "ens5" oifname "ens5" ip daddr 10.50.20.0/24 ct state established,related accept
        }

        chain postrouting {
                type nat hook postrouting priority srcnat; policy accept;
                ip saddr 10.50.20.0/24 oifname "ens5" masquerade
        }
}
```

## Source/Destination Check

Disabled on:

```text
BLACKFORGE-DMZ-01
```

This permits the DMZ instance to forward packets for the private host.

## Testing

### Before NAT

From the private host:

```bash
curl -4 --connect-timeout 5 https://example.com
```

Result:

```text
Connection timed out
```

This demonstrated that the private host initially had no working Internet egress.

### After NAT

From the private host:

```bash
curl -4 https://ifconfig.me
```

Result:

```text
13.229.212.104
```

This proves private outbound traffic is being NATed through the DMZ.

## SSH Management

Kali uses:

```bash
ssh server1
```

for the DMZ.

```bash
ssh server2
```

for the private host through `ProxyJump`.

Conceptually:

```text
Kali
 |
 +--> server1 -> DMZ
 |
 +--> server2 -> server1 -> PRIVATE
```

## Technical Understanding

### Subnet isolation

```text
DMZ     10.50.10.0/24
PRIVATE 10.50.20.0/24
```

are separate routed networks.

### Routing

The private subnet sends its default traffic toward the DMZ instead of directly to the Internet Gateway.

### IP forwarding

The DMZ Linux kernel forwards packets between the private workload and external network.

### NAT

Masquerading changes the source address of private outbound traffic so external services see the DMZ public IP.

## Security Perspective

### Red Team

The DMZ represents the intended exposed attack surface.

A later compromise of an exposed service will be used to study movement toward the private application.

### Blue Team

The private subnet creates a separate location for internal workloads and monitoring.

### Purple Team

Later testing will validate whether movement from the DMZ toward private assets can be detected and controlled.

## Evidence

* VPC/subnet screenshots
* EC2 instance details
* Route-table screenshots
* `ip -br addr`
* `ip route`
* nftables ruleset
* NAT verification
* SSH ProxyJump verification

## Status

* [x] VPC created
* [x] DMZ subnet created
* [x] Private subnet created
* [x] DMZ route table configured
* [x] Private route table configured
* [x] DMZ EC2 deployed
* [x] Private EC2 deployed
* [x] Private EC2 has no public IP
* [x] SSH aliases configured
* [x] SSH ProxyJump working
* [x] IPv4 forwarding enabled
* [x] nftables NAT configured
* [x] Source/destination check disabled
* [x] Private outbound Internet verified
* [ ] Final private security-group policy
* [ ] Verify Internet -> private is blocked
* [ ] WireGuard management plane
* [ ] Docker/Juice Shop
* [ ] Nginx/ModSecurity

```

**These are the files/locations to maintain. No new folder structure.**
```
