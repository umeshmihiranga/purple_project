````markdown
# BLACKFORGE

> **A hands-on Purple Team cybersecurity laboratory built to simulate attacks, understand how they work, detect them, investigate them, respond to them, and continuously improve the defensive architecture.**

---

<p align="center">

**LEARN • ATTACK • DETECT • INVESTIGATE • RESPOND • IMPROVE**

</p>

---

# 1. Project Overview

**BLACKFORGE** is a hands-on cybersecurity laboratory designed to provide a realistic environment for practicing both offensive and defensive security.

The project combines:

- Cloud infrastructure
- Network segmentation
- Linux administration
- Web application security
- Offensive security
- Vulnerability assessment
- Privilege escalation
- Network pivoting
- Detection engineering
- Incident response
- Purple Team operations

The lab is deployed in **Amazon Web Services (AWS)** and intentionally contains vulnerable workloads that can be attacked from a controlled attacker environment.

The purpose is not simply to run tools or collect flags.

The goal is to understand the complete security lifecycle:

```text
                    BLACKFORGE
                         │
                         ▼
                  BUILD THE LAB
                         │
                         ▼
                 UNDERSTAND THE
                  ATTACK SURFACE
                         │
                         ▼
                    RECON
                         │
                         ▼
                     ATTACK
                         │
                         ▼
                    DETECT
                         │
                         ▼
                 INVESTIGATE
                         │
                         ▼
                    RESPOND
                         │
                         ▼
                    HARDEN
                         │
                         ▼
                    RE-TEST
                         │
                         └───────────────┐
                                         │
                                         ▼
                                  IMPROVE THE LAB
````

BLACKFORGE is therefore designed as an iterative cybersecurity learning environment rather than a collection of isolated tools.

---

# 2. Project Objectives

The primary objective of BLACKFORGE is to build a realistic, isolated cybersecurity environment where offensive and defensive security can be studied together.

## Core Objectives

### 1. Build a realistic cloud environment

Deploy a segmented AWS environment containing:

* Public-facing infrastructure
* A DMZ (Demilitarized Zone)
* An isolated private subnet
* Internal workloads
* Controlled routing
* NAT (Network Address Translation)
* Security Groups
* Dedicated management access

---

### 2. Create a controlled attack surface

Expose only the components that are intentionally designed to be reachable.

The application is placed behind:

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
Private Application
```

This creates a realistic location for testing:

* Web attacks
* WAF (Web Application Firewall) detection
* Request filtering
* Logging
* Rate limiting
* Application-layer attacks

---

### 3. Isolate internal workloads

Internal workloads should not require public Internet exposure.

The primary application server therefore resides inside:

```text
BLACKFORGE-PRIVATE
10.50.20.0/24
```

and does not have a public IP address.

---

### 4. Create a separate management plane

Administrative access is separated from normal application traffic.

WireGuard provides a dedicated management network:

```text
10.50.30.0/24
```

with:

```text
Kali → 10.50.30.2
DMZ  → 10.50.30.1
```

Administrative SSH (Secure Shell) access uses the VPN rather than relying on public SSH exposure.

---

### 5. Practice offensive security

The environment provides controlled targets for studying:

* Reconnaissance
* Enumeration
* Web application vulnerabilities
* Authentication weaknesses
* Injection attacks
* Cross-Site Scripting (XSS)
* SQL Injection (SQLi)
* Command injection
* Privilege escalation
* Lateral movement
* Network pivoting

---

### 6. Practice defensive security

The same attacks are used to study:

* Security logging
* WAF detection
* Network controls
* Alert generation
* Investigation
* Containment
* Eradication
* Recovery
* Hardening

---

### 7. Practice Purple Team methodology

BLACKFORGE combines offensive and defensive activities.

The goal is to answer:

> **Can the attack be performed, can it be detected, can it be investigated, and can the environment be improved afterward?**

---

# 3. High-Level Architecture

```text
                                      INTERNET
                                          │
                         ┌────────────────┴────────────────┐
                         │                                 │
                   Web / Attack Traffic              WireGuard VPN
                         │                                 │
                         ▼                                 ▼
              ┌──────────────────────┐          ┌────────────────┐
              │         DMZ          │          │  Kali Linux    │
              │   10.50.10.0/24     │          │ 10.50.30.2     │
              │                      │          └───────┬────────┘
              │ BLACKFORGE-DMZ-01    │                  │
              │ 10.50.10.116         │◄─────────────────┘
              │ Public IP             │       Management
              │ 13.229.212.104        │
              │                      │
              │ Nginx                │
              │ ModSecurity          │
              │ OWASP CRS            │
              │ WireGuard             │
              │ nftables              │
              │ NAT / Routing         │
              └──────────┬───────────┘
                         │
                         │ Controlled
                         │ Internal Traffic
                         ▼
              ┌──────────────────────┐
              │       PRIVATE       │
              │   10.50.20.0/24     │
              │                      │
              │ BLACKFORGE-PRIVATE-01│
              │ 10.50.20.32         │
              │ No Public IP         │
              │                      │
              │ Docker               │
              │ OWASP Juice Shop     │
              └──────────────────────┘
```

---

# 4. AWS Infrastructure

## AWS Region

```text
Region:
ap-southeast-1

Location:
Singapore
```

---

## VPC

```text
Name:
BLACKFORGE-VPC

CIDR:
10.50.0.0/16
```

The VPC (Virtual Private Cloud) provides the isolated AWS network used by BLACKFORGE.

---

# 5. Network Segmentation

BLACKFORGE uses multiple logical network zones.

## DMZ

```text
Name: BLACKFORGE-DMZ
CIDR: 10.50.10.0/24
```

Purpose:

* Public-facing entry point
* Reverse proxy
* WAF
* VPN endpoint
* Router
* NAT
* Security control point

---

## Private Network

```text
Name: BLACKFORGE-PRIVATE
CIDR: 10.50.20.0/24
```

Purpose:

* Internal workloads
* Vulnerable application
* Backend services
* Future internal targets

The private subnet does not expose its workloads directly to the Internet.

---

## WireGuard Network

```text
CIDR:
10.50.30.0/24
```

This is a logical VPN network rather than an AWS subnet.

Endpoints:

```text
Kali:
10.50.30.2

DMZ:
10.50.30.1
```

---

# 6. EC2 Infrastructure

## BLACKFORGE-DMZ-01

```text
Private IP:
10.50.10.116

Public IP:
13.229.212.104

Subnet:
BLACKFORGE-DMZ
```

Primary roles:

```text
Edge
Reverse Proxy
WAF
WireGuard Server
Router
NAT
Security Control Point
```

---

## BLACKFORGE-PRIVATE-01

```text
Private IP:
10.50.20.32

Public IP:
None

Subnet:
BLACKFORGE-PRIVATE
```

Primary roles:

```text
Internal Workload
Docker Host
OWASP Juice Shop
Vulnerable Application Target
```

---

# 7. Traffic Architecture

BLACKFORGE deliberately separates different traffic types.

## Public Application Traffic

```text
Internet
   │
   ▼
Internet Gateway
   │
   ▼
DMZ
   │
   ▼
Nginx
   │
   ▼
ModSecurity
   │
   ▼
OWASP CRS
   │
   ▼
Private Application
```

---

## Management Traffic

```text
Kali
10.50.30.2
   │
   │ WireGuard
   ▼
DMZ
10.50.30.1
   │
   │ SSH
   ▼
Private Server
10.50.20.32
```

---

## Private Outbound Traffic

```text
Private
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

The private server can access the Internet without requiring a public IP.

---

# 8. Network Security Model

The architecture follows several security principles.

## Segmentation

Public-facing components and internal workloads are separated.

```text
Internet
    ↓
DMZ
    ↓
Private
```

---

## Least Exposure

The private server does not have a public IP.

Administrative services are intended to be accessed through the management network.

---

## Default-Deny Forwarding

The DMZ uses nftables with a default-drop forwarding policy.

```text
forward policy:
drop
```

Only explicitly required forwarding paths are allowed.

---

## Dedicated Management Plane

WireGuard separates administrative traffic from normal application traffic.

```text
Application Plane
    Internet → DMZ → Private

Management Plane
    Kali → WireGuard → DMZ → Private
```

---

# 9. Nginx Reverse Proxy

Nginx acts as the public application entry point.

Traffic arriving at the DMZ is forwarded to the private Juice Shop server.

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
```

This provides a centralized location for:

* HTTP logging
* WAF inspection
* Security headers
* Rate limiting
* TLS termination
* Request filtering
* Application traffic analysis

---

# 10. ModSecurity WAF

ModSecurity provides WAF functionality on the Nginx reverse proxy.

Configuration includes:

```text
SecRuleEngine On
```

Relevant audit logging is enabled.

The WAF inspects HTTP requests before forwarding them to the private application.

---

# 11. OWASP Core Rule Set

The OWASP CRS (Core Rule Set) provides detection rules for common web attacks.

BLACKFORGE uses CRS rules covering categories such as:

* SQL Injection
* Cross-Site Scripting
* Remote Code Execution
* Local File Inclusion
* Remote File Inclusion
* Protocol attacks
* Session attacks
* Application-layer attacks

---

# 12. WAF Verification

The WAF was verified using a controlled SQL Injection test.

Example:

```bash
curl -i "http://13.229.212.104/rest/products/search?q=%27%20OR%201%3D1--"
```

The request triggered CRS rule:

```text
942100
SQL Injection Attack Detected via libinjection
```

The request was then blocked:

```text
HTTP/1.1 403 Forbidden
```

This confirms that ModSecurity and OWASP CRS are actively enforcing security policy rather than simply being installed.

---

# 13. WAF False Positive Handling

A normal request using the public IP as the HTTP Host header triggered:

```text
920350
Host header is a numeric IP address
```

A narrow exclusion was created rather than disabling the entire rule category.

```apache
SecRule REQUEST_HEADERS:Host "@streq 13.229.212.104" \
    "id:1001,\
    phase:1,\
    pass,\
    nolog,\
    ctl:ruleRemoveById=920350"
```

Verification demonstrated:

```text
920350 → excluded
942100 → detected
949110 → blocked
```

This demonstrates controlled WAF tuning.

---

# 14. Docker and OWASP Juice Shop

The private server runs Docker.

The primary vulnerable application is:

```text
OWASP Juice Shop
```

Container:

```text
juice-shop
```

Application port:

```text
3000
```

Internal endpoint:

```text
10.50.20.32:3000
```

The application is intentionally vulnerable and provides the main web attack surface for BLACKFORGE.

---

# 15. Linux Routing

The DMZ performs IPv4 routing.

IPv4 forwarding:

```text
net.ipv4.ip_forward=1
```

Persistent configuration:

```text
/etc/sysctl.d/99-blackforge-router.conf
```

This allows the DMZ to forward traffic between:

* WireGuard
* Private subnet
* Internet

---

# 16. NAT

NAT (Network Address Translation) is performed on the DMZ.

## Private → Internet

```text
10.50.20.0/24
        │
        ▼
      DMZ
        │
       NAT
        │
        ▼
    Internet
```

---

## VPN → Private

```text
10.50.30.2
      │
      ▼
DMZ
      │
     NAT
      │
      ▼
10.50.20.32
```

This allows the existing DMZ Security Group trust model to support VPN-originated administrative traffic.

---

# 17. nftables

The DMZ uses nftables as the Linux firewall and forwarding layer.

Persistent configuration:

```text
/etc/nftables.conf
```

Forwarding uses:

```text
policy drop
```

Relevant traffic is explicitly allowed.

NAT rules include:

```text
10.50.20.0/24 → Internet
```

and:

```text
10.50.30.0/24 → 10.50.20.0/24
```

The nftables service is enabled at boot.

---

# 18. WireGuard Management VPN

WireGuard provides the dedicated management plane.

```text
VPN:
10.50.30.0/24
```

Endpoints:

```text
DMZ:
10.50.30.1

Kali:
10.50.30.2
```

WireGuard listens on:

```text
UDP 51820
```

---

# 19. SSH Management

SSH access is structured around the WireGuard network.

```text
Host server1
    ↓
10.50.30.1
    ↓
DMZ
```

The private server is accessed through ProxyJump:

```text
Host server2
    ↓
ProxyJump server1
    ↓
DMZ
    ↓
10.50.20.32
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

---

# 20. VPN Recovery

BLACKFORGE includes a recovery mechanism for the WireGuard management interface.

If `wg0` is accidentally removed:

```text
wg0
 ↓
missing
 ↓
watchdog detects failure
 ↓
wg-quick@wg0 restarted
 ↓
wg0 recreated
 ↓
WireGuard handshake restored
 ↓
management access restored
```

The failure scenario was deliberately tested.

Recovery succeeded.

This is important because the management plane itself should not become a single point of administrative failure.

---

# 21. Why ICMP Ping Is Not Used as the Only Connectivity Test

The private server does not necessarily respond to ICMP (Internet Control Message Protocol) echo requests.

Therefore:

```text
ping 10.50.20.32
```

may fail even when the required services are reachable.

TCP (Transmission Control Protocol) port 22 was tested directly:

```bash
nc -vz -w 5 10.50.20.32 22
```

Result:

```text
10.50.20.32:22 open
```

SSH connectivity was also successfully verified.

This demonstrates an important networking principle:

> **A failed ping does not necessarily mean that a host or service is unreachable. Test the actual service and protocol you require.**

---

# 22. Reconnaissance

The reconnaissance phase focuses on understanding the attack surface before exploitation.

Areas of study include:

```text
Network discovery
Port scanning
Service enumeration
HTTP enumeration
Technology identification
Directory discovery
Application mapping
Attack surface identification
```

The goal is to answer:

```text
What exists?
What is exposed?
What services are running?
What versions are present?
What can be reached?
What should not be reachable?
```

---

# 23. Vulnerability Assessment

The vulnerability phase focuses on identifying weaknesses within the BLACKFORGE environment.

Potential areas include:

```text
Web application vulnerabilities
Authentication weaknesses
Injection vulnerabilities
Access control issues
Configuration weaknesses
Exposed services
Misconfigured network controls
Container weaknesses
Linux weaknesses
```

Each vulnerability should ideally document:

```text
Vulnerability
    ↓
Root Cause
    ↓
Exploitation
    ↓
Evidence
    ↓
Detection
    ↓
Impact
    ↓
Mitigation
    ↓
Re-test
```

---

# 24. Privilege Escalation

The privilege escalation phase focuses on understanding how a compromised system can potentially be taken from a low-privileged context to a higher-privileged context.

Areas include:

```text
Linux enumeration
Sudo configuration
File permissions
SUID / SGID
Scheduled tasks
Services
Environment variables
Credentials
Weak configurations
Kernel exposure
```

The objective is not merely to obtain root access.

The objective is to understand:

```text
Initial Access
      ↓
Enumeration
      ↓
Weakness
      ↓
Privilege Escalation
      ↓
Detection Opportunity
      ↓
Defensive Improvement
```

---

# 25. Network Pivoting

The pivoting phase studies movement between network segments after initial compromise.

BLACKFORGE intentionally provides separate network zones:

```text
DMZ
10.50.10.0/24

Private
10.50.20.0/24

Management
10.50.30.0/24
```

This provides a realistic environment for studying:

* Internal discovery
* Route analysis
* Segmentation
* Lateral movement
* Pivoting
* Access control
* Trust relationships

The objective is to understand how an attacker could move from an exposed system toward protected internal resources.

---

# 26. Detection Engineering

The detection phase focuses on answering:

> **What evidence does an attack leave behind?**

Potential telemetry sources include:

```text
Nginx logs
ModSecurity audit logs
Linux authentication logs
System logs
Network traffic
nftables events
Application logs
Container logs
WireGuard logs
```

Detection work should map:

```text
Attack
  ↓
Observable Activity
  ↓
Log Source
  ↓
Detection Logic
  ↓
Alert
  ↓
Investigation
```

---

# 27. Incident Response

The incident-response phase models the defensive lifecycle following an attack.

The process includes:

```text
Preparation
    ↓
Identification
    ↓
Containment
    ↓
Eradication
    ↓
Recovery
    ↓
Lessons Learned
```

The goal is to move beyond simply detecting an attack.

BLACKFORGE should demonstrate the ability to:

* Identify malicious activity
* Determine scope
* Contain the attacker
* Remove persistence
* Restore services
* Harden affected systems
* Validate the fix

---

# 28. Purple Team Operations

Purple Team exercises connect the offensive and defensive sides of the project.

The workflow is:

```text
RED TEAM
   │
   │ Attack
   ▼
Target
   │
   ▼
BLUE TEAM
   │
   │ Detect
   ▼
Investigation
   │
   ▼
Response
   │
   ▼
Hardening
   │
   ▼
RED TEAM
   │
   │ Re-test
   ▼
Improved Control
```

Each exercise should answer four questions:

### 1. Can the attack be performed?

```text
ATTACK
```

### 2. Was it detected?

```text
DETECTION
```

### 3. Can the activity be investigated?

```text
INVESTIGATION
```

### 4. Did the defensive change actually work?

```text
VALIDATION
```

---

# 29. Repository Structure

The repository is organized according to the BLACKFORGE workflow.

```text
BLACKFORGE/
│
├── README.md
│
├── 01-architecture/
│   ├── aws-infrastructure.md
│   ├── diagram.png
│   └── security-groups.md
│
├── 03-networking/
│   ├── ip-addressing.md
│   ├── packet-analysis.md
│   ├── README.md
│   └── routing.md
│
├── 04-reconnaissance/
│
├── 05-vulnerabilities/
│
├── 06-privilege-escalation/
│
├── 07-pivoting/
│
├── 08-detection/
│
├── 09-incident-response/
│
├── 10-purple-team/
│
└── notes/
```

The repository intentionally separates the different stages of the security lifecycle.

---

# 30. Documentation Philosophy

BLACKFORGE documentation focuses on **understanding**, not just recording commands.

Each technical exercise should attempt to document:

```text
WHAT
What are we doing?

WHY
Why are we doing it?

HOW
How does it work?

COMMAND
What command/configuration was used?

RESULT
What happened?

MEANING
What does the result tell us?

SECURITY IMPACT
Why does it matter?

VERIFICATION
How do we know it works?

LESSON
What was learned?
```

This makes the repository useful as both:

* A personal learning environment
* A technical portfolio
* A security reference
* A record of experimentation

---

# 31. Technologies

BLACKFORGE currently uses:

### Cloud

```text
Amazon Web Services (AWS)
Amazon VPC
Amazon EC2
Security Groups
Internet Gateway
Route Tables
```

### Operating Systems

```text
Kali Linux
Ubuntu Linux
```

### Networking

```text
IPv4
Routing
NAT
nftables
TCP/IP
WireGuard
```

### Web Infrastructure

```text
Nginx
HTTP
Reverse Proxy
```

### Security

```text
ModSecurity
OWASP Core Rule Set
Web Application Firewall
SQL Injection detection
Security logging
```

### Application

```text
Docker
OWASP Juice Shop
```

### Offensive Security

```text
Reconnaissance
Enumeration
Web Application Testing
Vulnerability Exploitation
Privilege Escalation
Pivoting
```

### Defensive Security

```text
WAF
Network Segmentation
Firewalling
Logging
Detection
Incident Response
Hardening
```

---

# 32. Security Principles Demonstrated

BLACKFORGE is designed around practical security principles.

## Defense in Depth

Multiple security controls protect the environment:

```text
AWS Security Groups
        +
Network Segmentation
        +
nftables
        +
Nginx
        +
ModSecurity
        +
OWASP CRS
        +
Application Security
        +
Monitoring
```

---

## Least Privilege

Systems should only expose the services required for their role.

---

## Network Segmentation

Public-facing and internal systems are separated.

---

## Reduced Attack Surface

The private workload does not require direct Internet exposure.

---

## Separate Management Plane

Administrative traffic uses WireGuard rather than the public application path.

---

## Continuous Validation

Security controls are tested rather than assumed to work.

---

# 33. Verification Philosophy

BLACKFORGE does not consider a configuration complete merely because a command succeeds.

The project uses four levels of validation:

```text
CONFIGURED
    ↓
VERIFIED
    ↓
UNDERSTOOD
    ↓
DOCUMENTED
```

For security controls, an additional level is used:

```text
CONFIGURED
    ↓
VERIFIED
    ↓
ATTACK TEST
    ↓
DETECTION TEST
    ↓
RESPONSE TEST
    ↓
RE-TEST
```

This prevents the project from becoming a collection of unverified configurations.

---

# 34. Current Infrastructure Verification

The current environment has successfully verified:

```text
AWS VPC                         ✅
DMZ subnet                      ✅
Private subnet                  ✅
Private server isolation        ✅
DMZ routing                     ✅
IPv4 forwarding                 ✅
Private → Internet NAT          ✅
nftables firewall               ✅
nftables persistence            ✅
Nginx reverse proxy             ✅
Private Juice Shop              ✅
ModSecurity                     ✅
OWASP CRS                       ✅
SQL Injection detection         ✅
SQL Injection blocking          ✅
CRS false-positive tuning       ✅
WireGuard VPN                   ✅
WireGuard handshake             ✅
Kali → DMZ                      ✅
Kali → Private                  ✅
SSH server1                     ✅
SSH server2 via ProxyJump       ✅
WireGuard boot persistence      ✅
WireGuard failure recovery      ✅
```

---

# 35. Infrastructure Status

```text
┌────────────────────────────────────┐
│      BLACKFORGE INFRASTRUCTURE     │
│                                    │
│             COMPLETE               │
│                                    │
│  AWS Network               ✅      │
│  DMZ                       ✅      │
│  Private Network           ✅      │
│  Routing                   ✅      │
│  NAT                       ✅      │
│  Firewall                  ✅      │
│  Reverse Proxy             ✅      │
│  WAF                       ✅      │
│  Vulnerable Application    ✅      │
│  Management VPN            ✅      │
│  Persistence               ✅      │
│  Recovery                  ✅      │
│  Verification              ✅      │
└────────────────────────────────────┘
```

The infrastructure phase is complete.

The project can now focus primarily on security operations and Purple Team exercises.

---

# 36. Project Roadmap

## Phase 1 — Architecture

```text
AWS VPC
DMZ
Private subnet
Security Groups
Routing
Network isolation
```

**Status: COMPLETE**

---

## Phase 2 — Networking

```text
IP addressing
Routing
NAT
Packet analysis
WireGuard
Network troubleshooting
```

**Status: ACTIVE / CONTINUOUS**

---

## Phase 3 — Reconnaissance

```text
Network discovery
Port scanning
Service enumeration
Web enumeration
Attack surface mapping
```

**Status: NEXT**

---

## Phase 4 — Vulnerabilities

```text
Web vulnerabilities
Authentication
Injection
Access control
Application weaknesses
```

**Status: PLANNED**

---

## Phase 5 — Privilege Escalation

```text
Linux enumeration
Misconfiguration discovery
Privilege escalation
Post-exploitation analysis
```

**Status: PLANNED**

---

## Phase 6 — Pivoting

```text
Internal discovery
Network segmentation testing
Lateral movement
Pivoting
```

**Status: PLANNED**

---

## Phase 7 — Detection

```text
Log analysis
WAF detection
Network detection
Host detection
Attack telemetry
Detection engineering
```

**Status: PLANNED**

---

## Phase 8 — Incident Response

```text
Identification
Containment
Eradication
Recovery
Lessons learned
```

**Status: PLANNED**

---

## Phase 9 — Purple Team

```text
Attack
Detect
Investigate
Respond
Harden
Re-test
```

**Status: PLANNED**

---

# 37. Example Purple Team Exercise

A typical BLACKFORGE exercise should follow this structure:

```text
1. Establish target
        ↓
2. Perform reconnaissance
        ↓
3. Identify vulnerability
        ↓
4. Exploit vulnerability
        ↓
5. Record evidence
        ↓
6. Identify telemetry
        ↓
7. Build detection
        ↓
8. Investigate activity
        ↓
9. Contain attack
        ↓
10. Harden system
        ↓
11. Re-run attack
        ↓
12. Validate detection
```

The final result should demonstrate improvement rather than simply successful exploitation.

---

# 38. Learning Objectives

BLACKFORGE is intended to develop practical understanding of:

### Cloud Security

```text
VPC design
Subnetting
Routing
Security Groups
Instance isolation
Cloud attack surfaces
```

### Network Security

```text
TCP/IP
Routing
NAT
Firewalls
VPNs
Packet analysis
Segmentation
```

### Web Security

```text
HTTP
Reverse proxies
WAF
SQL Injection
Cross-Site Scripting
Authentication
Application vulnerabilities
```

### Linux Security

```text
System administration
Permissions
Services
Networking
Logging
Privilege escalation
```

### Offensive Security

```text
Reconnaissance
Enumeration
Exploitation
Post-exploitation
Pivoting
Lateral movement
```

### Defensive Security

```text
Detection
Logging
Investigation
Incident response
Containment
Hardening
Validation
```

### Purple Team

```text
Attack → Detect → Investigate → Respond → Improve
```

---

# 39. Operational Philosophy

BLACKFORGE is intentionally built as a learning environment.

The goal is not:

```text
Run command
   ↓
Get result
   ↓
Move on
```

The goal is:

```text
Run command
   ↓
Understand the command
   ↓
Understand the network/system behavior
   ↓
Interpret the result
   ↓
Understand the security implication
   ↓
Document the lesson
```

This philosophy is applied throughout the project.

---

# 40. Lab Safety

BLACKFORGE is designed as a controlled cybersecurity laboratory.

The intentionally vulnerable workloads are deployed for authorized testing within the project environment.

Testing should remain limited to:

```text
BLACKFORGE infrastructure
Authorized lab systems
Controlled attack scenarios
```

The project should not be used to target systems without authorization.

---

# 41. Project Success Criteria

BLACKFORGE will be considered a mature Purple Team environment when the following lifecycle can be demonstrated:

```text
┌──────────────────────────────┐
│       ATTACKER               │
│                              │
│ Recon → Exploit → Pivot      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       DEFENDER               │
│                              │
│ Detect → Investigate         │
│ Contain → Respond            │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       ENGINEER               │
│                              │
│ Harden → Validate → Improve  │
└──────────────┬───────────────┘
               │
               └──────────────┐
                              │
                              ▼
                         Re-test
```

Success is therefore measured by the ability to demonstrate the **entire security lifecycle**, not simply by obtaining access to a vulnerable system.

---

# 42. BLACKFORGE Motto

```text
LEARN
ATTACK
DETECT
IMPROVE
```

> **A safer tomorrow through a deeper understanding today.**

---

# 43. Final Project Vision

BLACKFORGE is intended to evolve from a small AWS security lab into a complete Purple Team environment.

The long-term vision is:

```text
                     BLACKFORGE
                          │
          ┌───────────────┼────────────────┐
          │               │                │
          ▼               ▼                ▼
       RED TEAM       BLUE TEAM        CLOUD SECURITY
          │               │                │
          │               │                │
          ▼               ▼                ▼
      Attacks          Detection        AWS Security
      Exploits         Logging          Segmentation
      Pivoting         IR              IAM
          │               │                │
          └───────────────┼────────────────┘
                          │
                          ▼
                    PURPLE TEAM
                          │
                          ▼
                 Continuous Validation
                          │
                          ▼
                       HARDEN
                          │
                          ▼
                       RE-TEST
```

BLACKFORGE is therefore more than an intentionally vulnerable application.

It is a practical environment for studying how **infrastructure, attackers, defenders, detection systems, and incident response interact in a real security workflow.**

---

# 44. Repository Status

```text
Architecture                 COMPLETE
AWS Infrastructure           COMPLETE
Network Segmentation         COMPLETE
DMZ                          COMPLETE
Private Workload             COMPLETE
Reverse Proxy                COMPLETE
WAF                          COMPLETE
WireGuard Management         COMPLETE
Firewall / NAT               COMPLETE
Persistence                  COMPLETE
Recovery                     COMPLETE

Reconnaissance               NEXT
Vulnerability Research       NEXT
Privilege Escalation         PLANNED
Pivoting                     PLANNED
Detection Engineering       PLANNED
Incident Response            PLANNED
Purple Team Exercises        PLANNED
```

---

# BLACKFORGE

```text
LEARN
   │
   ▼
BUILD
   │
   ▼
ATTACK
   │
   ▼
DETECT
   │
   ▼
INVESTIGATE
   │
   ▼
RESPOND
   │
   ▼
HARDEN
   │
   ▼
RE-TEST
   │
   └──────────────► IMPROVE
```

**BLACKFORGE is a continuously evolving cybersecurity laboratory focused on practical security engineering and Purple Team learning.**

```
```
