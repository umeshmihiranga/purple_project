# BLACKFORGE

> A hands-on Purple Team cybersecurity lab built in AWS to simulate real-world attacks, detect malicious activity, investigate telemetry, and apply defensive controls.

---

## Overview

**BLACKFORGE** is an isolated cybersecurity laboratory designed to combine offensive security, defensive security, network engineering, and cloud infrastructure.

The environment is deployed in **AWS Singapore (`ap-southeast-1`)** and uses a segmented network architecture consisting of:

- A public-facing DMZ (Demilitarized Zone)
- An isolated private application subnet
- Nginx reverse proxy
- ModSecurity Web Application Firewall (WAF)
- OWASP Core Rule Set (CRS)
- OWASP Juice Shop
- Docker
- nftables routing and NAT
- WireGuard management VPN
- Kali Linux attack workstation

The objective is to create a realistic environment where attacks can be performed against an intentionally vulnerable application while defensive controls inspect, detect, log, and block malicious activity.

---

# Architecture

```text
                                      INTERNET
                                          │
                         ┌────────────────┴────────────────┐
                         │                                 │
                    Web Traffic                     WireGuard VPN
                         │                                 │
                         ▼                                 ▼
              ┌──────────────────────┐          ┌────────────────┐
              │         DMZ          │          │  Kali Linux    │
              │   10.50.10.0/24     │          │ 10.50.30.2     │
              │                      │          └───────┬────────┘
              │ BLACKFORGE-DMZ-01    │                  │
              │ 10.50.10.116         │◄─────────────────┘
              │ Public: 13.229.212.104│        WireGuard
              │                      │
              │ ┌──────────────────┐ │
              │ │ Nginx            │ │
              │ │ ModSecurity      │ │
              │ │ OWASP CRS        │ │
              │ │ WireGuard        │ │
              │ │ nftables         │ │
              │ └────────┬─────────┘ │
              └──────────┼───────────┘
                         │
                    NAT / Forwarding
                         │
                         ▼
              ┌──────────────────────┐
              │       PRIVATE        │
              │   10.50.20.0/24      │
              │                      │
              │ BLACKFORGE-PRIVATE-01│
              │ 10.50.20.32          │
              │ No Public IP         │
              │                      │
              │ ┌──────────────────┐ │
              │ │ Docker           │ │
              │ │ OWASP Juice Shop │ │
              │ │ :3000            │ │
              │ └──────────────────┘ │
              └──────────────────────┘
```

---

# AWS Infrastructure

## Region

```text
AWS Region: ap-southeast-1
Location: Singapore
```

## VPC

| Component | Configuration |
|---|---|
| VPC | `BLACKFORGE-VPC` |
| CIDR | `10.50.0.0/16` |
| Region | `ap-southeast-1` |

---

## Subnets

### DMZ

```text
Name: BLACKFORGE-DMZ
CIDR: 10.50.10.0/24
```

The DMZ contains the public-facing security infrastructure.

### Private

```text
Name: BLACKFORGE-PRIVATE
CIDR: 10.50.20.0/24
```

The private subnet contains the intentionally vulnerable application.

The private server has **no public IP address**.

---

# EC2 Instances

## BLACKFORGE-DMZ-01

```text
Private IP: 10.50.10.116
Public IP:  13.229.212.104
Subnet:     BLACKFORGE-DMZ
```

Responsibilities:

- Nginx reverse proxy
- ModSecurity WAF
- OWASP CRS
- WireGuard VPN server
- IPv4 routing
- nftables firewall
- NAT gateway functionality

---

## BLACKFORGE-PRIVATE-01

```text
Private IP: 10.50.20.32
Public IP:  None
Subnet:     BLACKFORGE-PRIVATE
```

Responsibilities:

- Docker host
- OWASP Juice Shop
- Vulnerable application target

---

# Network Routing

The DMZ acts as a Linux router between the private subnet, Internet, and WireGuard management network.

## Private → Internet

```text
Private Server
10.50.20.32
      │
      ▼
DMZ
10.50.10.116
      │
      │ NAT
      ▼
Internet
```

The private server can access the Internet without having a public IP.

This was verified using:

```bash
curl -4 https://ifconfig.me
```

The observed public address was the DMZ public IP.

---

# IPv4 Forwarding

Linux IPv4 forwarding is enabled on the DMZ:

```text
net.ipv4.ip_forward=1
```

Persistent configuration:

```text
/etc/sysctl.d/99-blackforge-router.conf
```

Verification:

```bash
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

---

# nftables

The DMZ uses **nftables** for packet forwarding and NAT.

The forwarding chain uses a default-drop policy:

```text
policy drop
```

This provides a default-deny forwarding model.

## Private subnet NAT

```text
10.50.20.0/24 → Internet
```

## WireGuard VPN NAT

```text
10.50.30.0/24 → 10.50.20.0/24
```

Relevant rules:

```text
ip saddr 10.50.20.0/24 oifname "ens5" masquerade

ip saddr 10.50.30.0/24 \
ip daddr 10.50.20.0/24 \
oifname "ens5" masquerade
```

Persistent configuration:

```text
/etc/nftables.conf
```

The nftables service is enabled at boot.

---

# Application Layer

## OWASP Juice Shop

The private server runs OWASP Juice Shop using Docker.

Container:

```text
juice-shop
```

Port:

```text
3000
```

Internal application endpoint:

```text
10.50.20.32:3000
```

The application is intentionally vulnerable and is used as the primary attack target.

---

# Docker

Juice Shop runs inside a Docker container:

```bash
docker run -d \
  --name juice-shop \
  -p 3000:3000 \
  bkimminich/juice-shop
```

The application is not directly exposed to the Internet.

---

# Nginx Reverse Proxy

Nginx runs on the DMZ server.

Its role is to provide a controlled public entry point to the private application.

Traffic flow:

```text
Internet
    │
    ▼
13.229.212.104:80
    │
    ▼
Nginx
    │
    ▼
10.50.20.32:3000
    │
    ▼
OWASP Juice Shop
```

Nginx forwards requests using:

```nginx
proxy_pass http://10.50.20.32:3000;
```

Important proxy headers include:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

This allows the reverse proxy to preserve useful client/request information for downstream logging and application behavior.

---

# Web Application Firewall

## ModSecurity

ModSecurity provides the WAF (Web Application Firewall) functionality.

It is integrated with Nginx and inspects incoming HTTP requests before they reach Juice Shop.

Current configuration:

```text
/etc/nginx/modsecurity.conf
```

ModSecurity is enabled with:

```text
SecRuleEngine On
```

Audit logging is enabled:

```text
SecAuditEngine RelevantOnly
SecAuditLog /var/log/nginx/modsec_audit.log
```

---

# OWASP Core Rule Set

OWASP CRS (Core Rule Set) provides the detection rules used by ModSecurity.

The environment includes rules for attack categories including:

- SQL Injection (SQLi)
- Cross-Site Scripting (XSS)
- Remote Code Execution (RCE)
- Local File Inclusion (LFI)
- Remote File Inclusion (RFI)
- Protocol attacks
- Session fixation
- Application-layer attacks

The Nginx-compatible CRS loader is:

```text
/etc/nginx/crs-blackforge.conf
```

It loads:

```text
/etc/modsecurity/crs/crs-setup.conf
/etc/modsecurity/crs/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
/usr/share/modsecurity-crs/rules/*.conf
/etc/modsecurity/crs/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf
```

---

# WAF Verification

A controlled SQL Injection test was performed against the Juice Shop API:

```bash
curl -i "http://13.229.212.104/rest/products/search?q=%27%20OR%201%3D1--"
```

ModSecurity detected the request using:

```text
Rule ID: 942100
SQL Injection Attack Detected via libinjection
```

The request was subsequently blocked:

```text
HTTP/1.1 403 Forbidden
```

The audit log also recorded:

```text
942100 SQL Injection Attack Detected via libinjection
949110 Inbound Anomaly Score Exceeded
```

This confirmed that the WAF is actively inspecting and blocking malicious requests.

---

# CRS False Positive Handling

A normal request using the public IP as the HTTP Host header triggered:

```text
920350
Host header is a numeric IP address
```

Because this was expected behavior for the BLACKFORGE lab, a targeted exclusion was created.

```apache
SecRule REQUEST_HEADERS:Host "@streq 13.229.212.104" \
    "id:1001,\
    phase:1,\
    pass,\
    nolog,\
    ctl:ruleRemoveById=920350"
```

The exclusion is intentionally narrow and only removes rule `920350` when the Host header matches the BLACKFORGE public IP.

SQL Injection protection remained active after the exclusion.

Verification demonstrated:

```text
920350 → excluded
942100 → detected
949110 → blocked
```

This demonstrates controlled WAF tuning rather than disabling broad categories of protection.

---

# WireGuard Management VPN

BLACKFORGE uses WireGuard as a dedicated management network.

```text
VPN Network: 10.50.30.0/24
```

Endpoints:

```text
DMZ:  10.50.30.1
Kali: 10.50.30.2
```

WireGuard listens on:

```text
UDP 51820
```

Architecture:

```text
Kali
10.50.30.2
    │
    │ WireGuard
    ▼
DMZ
10.50.30.1
    │
    │ NAT
    ▼
Private
10.50.20.32
```

---

# Administrative Access

Administrative SSH (Secure Shell) access is performed through the WireGuard management network.

Kali SSH configuration:

```sshconfig
Host server1
    HostName 10.50.30.1
    User ubuntu
    IdentityFile ~/.ssh/BLACKFORGE-KEY.pem
    IdentitiesOnly yes

Host server2
    HostName 10.50.20.32
    User ubuntu
    IdentityFile ~/.ssh/BLACKFORGE-KEY.pem
    IdentitiesOnly yes
    ProxyJump server1
```

This creates:

```text
Kali
 │
 │ WireGuard
 ▼
DMZ
 │
 │ SSH ProxyJump
 ▼
Private Server
```

The private server does not require a public IP.

---

# SSH Connectivity Verification

WireGuard connectivity was verified using:

```bash
sudo wg show
```

Kali successfully established a WireGuard handshake with the DMZ.

The private SSH service was independently verified:

```bash
nc -vz -w 5 10.50.20.32 22
```

Result:

```text
10.50.20.32:22 open
```

End-to-end SSH access was verified using:

```bash
ssh server1
```

and:

```bash
ssh server2
```

---

# Why Ping to the Private Server Fails

ICMP (Internet Control Message Protocol) ping to the private server does not receive a response:

```bash
ping 10.50.20.32
```

This does not indicate that the VPN or routing path is broken.

The Private Security Group allows the required TCP (Transmission Control Protocol) SSH service on port 22, while ICMP echo traffic is not permitted.

TCP connectivity was successfully verified:

```text
TCP/22 → OPEN
SSH     → WORKING
ICMP    → BLOCKED
```

This demonstrates an important defensive networking principle:

> A failed ping does not necessarily mean a host or service is unreachable. Test the specific protocol and port required by the service.

---

# WireGuard Persistence

WireGuard is managed using:

```text
wg-quick@wg0.service
```

The service is enabled at boot:

```bash
sudo systemctl enable wg-quick@wg0
```

Expected state:

```text
Loaded:  loaded; enabled
Active:  active (exited)
```

This ensures that the DMZ recreates the WireGuard interface after reboot.

---

# WireGuard Failure Recovery

A watchdog was implemented on the DMZ to detect when the WireGuard interface is missing.

Recovery flow:

```text
wg0 accidentally DOWN
        │
        ▼
Watchdog detects missing interface
        │
        ▼
Restart wg-quick@wg0
        │
        ▼
wg0 recreated
        │
        ▼
WireGuard handshake restored
        │
        ▼
Kali management access restored
```

The failure scenario was deliberately tested by taking `wg0` down.

The watchdog successfully restored the interface and management connectivity.

This provides protection against accidental manual removal of the WireGuard interface.

---

# Security Groups

The AWS Security Group architecture follows the principle of minimizing unnecessary public exposure.

The DMZ provides the public-facing application entry point.

The private server does not have a public IP.

Administrative SSH access is intended to occur through the WireGuard management network rather than direct public exposure.

WireGuard uses:

```text
UDP 51820
```

as its public VPN entry point.

---

# Management Model

BLACKFORGE separates:

### Application traffic

```text
Internet
   ↓
Nginx
   ↓
ModSecurity
   ↓
OWASP CRS
   ↓
Juice Shop
```

### Administrative traffic

```text
Kali
   ↓
WireGuard
   ↓
DMZ
   ↓
SSH
   ↓
Private Server
```

This separation prevents normal administrative access from being dependent on the public application interface.

---

# Verification Matrix

| Component | Verification | Status |
|---|---|---:|
| AWS VPC | AWS Console | ✅ |
| DMZ subnet | AWS Console | ✅ |
| Private subnet | AWS Console | ✅ |
| Private server public IP | None | ✅ |
| DMZ routing | `ip route` | ✅ |
| IPv4 forwarding | `sysctl` | ✅ |
| Private → Internet NAT | `curl ifconfig.me` | ✅ |
| nftables | `nft list ruleset` | ✅ |
| nftables persistence | systemd | ✅ |
| Nginx | HTTP request | ✅ |
| Reverse proxy | Public → Juice Shop | ✅ |
| ModSecurity | WAF test | ✅ |
| OWASP CRS | Rule loading | ✅ |
| SQL Injection detection | CRS 942100 | ✅ |
| SQL Injection blocking | HTTP 403 | ✅ |
| CRS exclusion | Rule 920350 | ✅ |
| WireGuard | `wg show` | ✅ |
| WireGuard handshake | Peer status | ✅ |
| Kali → DMZ | VPN | ✅ |
| Kali → Private | TCP/22 | ✅ |
| SSH server1 | SSH | ✅ |
| SSH server2 | ProxyJump | ✅ |
| WireGuard boot persistence | systemd | ✅ |
| WireGuard recovery | Failure test | ✅ |

---

# Current Infrastructure Status

| Component | Status |
|---|---:|
| AWS Infrastructure | ✅ COMPLETE |
| Network Segmentation | ✅ COMPLETE |
| DMZ | ✅ COMPLETE |
| Private Application Network | ✅ COMPLETE |
| Nginx Reverse Proxy | ✅ COMPLETE |
| ModSecurity WAF | ✅ COMPLETE |
| OWASP CRS | ✅ COMPLETE |
| Docker / Juice Shop | ✅ COMPLETE |
| nftables Firewall | ✅ COMPLETE |
| NAT | ✅ COMPLETE |
| WireGuard VPN | ✅ COMPLETE |
| SSH Management Path | ✅ COMPLETE |
| Boot Persistence | ✅ COMPLETE |
| WireGuard Recovery | ✅ COMPLETE |
| Infrastructure Verification | ✅ COMPLETE |


# Infrastructure Milestone

The BLACKFORGE infrastructure phase is complete.

The environment is now ready for controlled offensive security exercises.

The next phase is:

```text
┌──────────────────────┐
│   INFRASTRUCTURE     │
│      COMPLETE        │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  RECONNAISSANCE      │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   ATTACK SIMULATION  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│      DETECTION       │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│    INVESTIGATION     │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│      RESPONSE        │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│      HARDENING       │
└──────────────────────┘
```