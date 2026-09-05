Absolutely. Replace the contents of:

```text
01-cloud-architecture/security-groups.md
```

with this:

````markdown
# BLACKFORGE Security Groups

## Purpose

Security Groups (SGs) provide stateful, instance-level network filtering for BLACKFORGE EC2 instances.

They define which traffic is allowed to reach an instance and which traffic the instance is allowed to send.

BLACKFORGE uses separate security groups for the DMZ and private workload.

---

# Security Group Architecture

```text
                         INTERNET
                            |
                            | TCP 80/443
                            v
                  +---------------------+
                  | BLACKFORGE-DMZ-SG    |
                  |                     |
                  | DMZ-01               |
                  | 10.50.10.116         |
                  +----------+----------+
                             |
                             | Controlled
                             | internal traffic
                             v
                  +---------------------+
                  | BLACKFORGE-PRIVATE-SG|
                  |                     |
                  | PRIVATE-01           |
                  | 10.50.20.32          |
                  +---------------------+
````

The important security boundary is:

```text
Internet → DMZ → Private
```

There should be no direct Internet → Private access.

---

# 1. BLACKFORGE-DMZ-SG

## Basic Information

```text
Name:
BLACKFORGE-DMZ-SG

Purpose:
Public-facing DMZ security group

VPC:
BLACKFORGE-VPC
```

The DMZ security group is attached to:

```text
BLACKFORGE-DMZ-01
```

Private IP:

```text
10.50.10.116
```

Public IP:

```text
13.229.212.104
```

---

## Inbound Rules

### HTTP

```text
Protocol: TCP
Port: 80
Source: 0.0.0.0/0
Purpose: Public HTTP
```

This allows HTTP traffic from the Internet to reach the DMZ.

### HTTPS

Planned:

```text
Protocol: TCP
Port: 443
Source: 0.0.0.0/0
Purpose: Public HTTPS
```

HTTPS will eventually be used for the public-facing application/WAF path.

### WireGuard

Planned:

```text
Protocol: UDP
Port: 51820
Source: <management client IP>/32
Purpose: WireGuard management
```

The final rule should be restricted to the administrator's known public IP where practical.

---

## Outbound Rules

Current policy:

```text
Protocol: All
Destination: 0.0.0.0/0
```

The DMZ requires outbound connectivity because it performs:

* NAT
* routing
* package updates
* reverse proxy communication
* future security tooling

---

# 2. BLACKFORGE-PRIVATE-SG

## Basic Information

```text
Name:
BLACKFORGE-PRIVATE-SG

Purpose:
Internal workload security group

VPC:
BLACKFORGE-VPC
```

The private security group is attached to:

```text
BLACKFORGE-PRIVATE-01
```

Private IP:

```text
10.50.20.32
```

Public IP:

```text
NONE
```

---

# Inbound Rules

## SSH

```text
Protocol: TCP
Port: 22
Source: BLACKFORGE-DMZ-SG
Purpose: Controlled administrative access
```

Using the security group as the source is intentional.

It means the rule trusts network interfaces belonging to the specified security group rather than opening SSH to an entire CIDR or the Internet.

Management will later be further controlled through the WireGuard management plane.

---

## Juice Shop

Planned:

```text
Protocol: TCP
Port: 3000
Source: BLACKFORGE-DMZ-SG
Purpose: Application traffic
```

This allows the future DMZ reverse proxy to reach OWASP Juice Shop running on the private host.

The private application should not be exposed directly to the Internet.

Traffic should follow:

```text
Internet
   |
   v
DMZ
   |
   v
Nginx / ModSecurity
   |
   v
PRIVATE:3000
```

---

# Outbound Rules

Current policy:

```text
Protocol: All
Destination: 0.0.0.0/0
```

The private host requires outbound connectivity for:

* package updates
* Docker image downloads
* dependency installation
* security tooling
* controlled lab activities

The security group alone does not determine the complete network path.

The private route table forces Internet-bound traffic through the DMZ/NAT path.

---

# Security Boundary

The intended architecture is:

```text
                    INTERNET
                        |
                        v
               +----------------+
               |      DMZ       |
               |                |
               | DMZ-01         |
               | 10.50.10.116   |
               +-------+--------+
                       |
                SG-controlled
                internal traffic
                       |
                       v
               +----------------+
               |    PRIVATE     |
               |                |
               | PRIVATE-01     |
               | 10.50.20.32    |
               +----------------+
```

The private host has:

```text
Public IPv4: NONE
```

Therefore an Internet host cannot directly establish a connection to its public address.

---

# Stateful Behavior

AWS Security Groups are stateful.

For example, if the private host initiates:

```text
PRIVATE → DMZ → INTERNET
```

the response traffic is automatically allowed back for that established connection when permitted by the relevant security-group state.

This is different from a stateless network ACL.

---

# Security Group vs Route Table

BLACKFORGE uses both.

## Route Table

Determines:

```text
WHERE traffic goes
```

Example:

```text
Private subnet
0.0.0.0/0
        |
        v
DMZ-01
```

## Security Group

Determines:

```text
WHICH traffic is permitted
```

Example:

```text
PRIVATE-01
TCP 22
Source: BLACKFORGE-DMZ-SG
```

Therefore:

```text
Routing + Security Groups
        =
Network path + Access control
```

---

# Intended Traffic Policy

| Traffic                          | Expected          |
| -------------------------------- | ----------------- |
| Internet → DMZ TCP/80            | ALLOW             |
| Internet → DMZ TCP/443           | ALLOW             |
| Internet → DMZ SSH               | DENY              |
| Internet → Private               | DENY              |
| DMZ → Private TCP/22             | ALLOW             |
| DMZ → Private TCP/3000           | ALLOW             |
| Private → Internet               | ALLOW via DMZ/NAT |
| Private → DMZ                    | Controlled        |
| Kali → infrastructure management | Via WireGuard     |
| Internet → Private TCP/3000      | DENY              |

---

# Security Philosophy

BLACKFORGE follows least privilege.

The goal is not:

```text
ALLOW EVERYTHING
```

Instead:

```text
ALLOW
only what is required
for the intended traffic path.
```

The DMZ is intentionally exposed because it represents the attack surface.

The private network is intentionally restricted because it represents internal infrastructure.

---

# Red Team Perspective

The DMZ is the first target.

Example attack path:

```text
Internet
   |
   v
DMZ
   |
   | compromise / foothold
   v
DMZ host
   |
   | lateral movement attempt
   v
PRIVATE
```

The private security group becomes one of the controls that should restrict this movement.

---

# Blue Team Perspective

A defender should be able to answer:

1. What ports are publicly exposed?
2. Which systems can communicate with the private host?
3. Which security group allowed the connection?
4. Was the connection expected?
5. Did traffic originate from the DMZ?
6. Was there an attempted direct connection from the Internet?
7. Can the traffic be detected and investigated?

---

# Purple Team Validation

The eventual test cycle will be:

```text
ATTACK
   ↓
OBSERVE
   ↓
UNDERSTAND
   ↓
DETECT
   ↓
INVESTIGATE
   ↓
DEFEND
   ↓
RETEST
```

Example:

```text
Attempt Internet → PRIVATE:3000
            ↓
        Should fail
            ↓
       Investigate
            ↓
Verify SG + route table
            ↓
       Document result
```

---

# Current Status

* [x] `BLACKFORGE-DMZ-SG` created
* [x] DMZ HTTP rule configured
* [x] DMZ outbound rule configured
* [x] Private security group planned
* [ ] Final `BLACKFORGE-PRIVATE-SG` rules
* [ ] Validate DMZ → Private access
* [ ] Validate Internet → Private denial
* [ ] Restrict management to WireGuard
* [ ] Configure HTTPS
* [ ] Configure Nginx
* [ ] Configure ModSecurity
* [ ] Deploy OWASP Juice Shop
* [ ] Perform Red Team validation
* [ ] Build Blue Team detections
* [ ] Purple Team retest

---

# Key Principle

> The DMZ is allowed to be attacked. The private network is allowed to be reached only through controlled paths.

BLACKFORGE is designed so that compromising the DMZ does not automatically mean compromising the private network.

```