````markdown
# BLACKFORGE Packet Analysis

This document records packet-level troubleshooting and traffic analysis performed during the BLACKFORGE networking implementation.

Packet analysis was used to understand how traffic moves through the environment and to identify where communication succeeds or fails.

The primary tool used was `tcpdump`.

---

# Why Packet Analysis?

When a connection fails, configuration files alone do not always explain what happened.

A packet may be:

- Never sent
- Sent through the wrong interface
- Dropped by a firewall
- Dropped by an AWS Security Group
- Routed incorrectly
- Received by the destination but not answered
- Answered but unable to return
- Rejected because the service is not listening

Packet capture allows the actual traffic to be observed.

---

# Tool Used

```bash
sudo tcpdump
````

`tcpdump` is a command-line packet capture and analysis tool.

It allows network traffic to be observed directly from a network interface.

---

# BLACKFORGE Network Interfaces

The DMZ server has two important interfaces for this analysis.

## `ens5`

```text
AWS network interface
```

This interface connects the DMZ server to the AWS VPC.

It is used for traffic involving:

* The DMZ subnet
* The private subnet
* Internet traffic

---

## `wg0`

```text
WireGuard VPN interface
```

This interface carries management traffic from the Kali system.

The WireGuard network is:

```text
10.50.30.0/24
```

Kali:

```text
10.50.30.2
```

DMZ:

```text
10.50.30.1
```

---

# Intended VPN → Private Traffic Path

The intended path is:

```text
Kali
10.50.30.2
    │
    │ WireGuard
    ▼
DMZ
10.50.30.1
    │
    │ IPv4 forwarding
    ▼
Private Network
10.50.20.0/24
    │
    ▼
Private Server
10.50.20.32
```

The packet should therefore enter the DMZ through `wg0` and leave the DMZ through `ens5`.

---

# Capturing WireGuard Traffic

To observe traffic entering the DMZ through WireGuard:

```bash
sudo tcpdump -ni wg0 host 10.50.20.32
```

### Command breakdown

```text
sudo
```

Runs the command with elevated privileges.

```text
tcpdump
```

Starts packet capture.

```text
-n
```

Disables hostname resolution.

This keeps the output easier to read.

```text
-i wg0
```

Captures traffic on the WireGuard interface.

```text
host 10.50.20.32
```

Filters traffic involving the private server.

---

# Observed Packet

During testing, the capture showed:

```text
10.50.30.2 > 10.50.20.32: ICMP echo request
```

This means:

```text
Source:
10.50.30.2
```

Kali.

```text
Destination:
10.50.20.32
```

Private server.

```text
Protocol:
ICMP
```

Internet Control Message Protocol.

```text
Type:
echo request
```

A ping request.

---

# What This Proved

The packet appearing on `wg0` demonstrated that the request successfully reached the DMZ through WireGuard.

The path:

```text
Kali
   ↓
WireGuard
   ↓
DMZ
```

was therefore functioning.

This was important because it eliminated the WireGuard tunnel itself as the primary cause of the connectivity problem.

---

# Capturing Traffic on `ens5`

The next step was to observe whether the DMZ forwarded the packet toward the private subnet.

Command:

```bash
sudo tcpdump -ni ens5 host 10.50.20.32
```

The request was observed leaving through the AWS network interface toward:

```text
10.50.20.32
```

This demonstrated that the packet progressed through:

```text
wg0
   ↓
DMZ
   ↓
ens5
   ↓
Private Network
```

---

# Packet Flow

The observed traffic can be represented as:

```text
Kali
10.50.30.2
     │
     │ ICMP Echo Request
     ▼
wg0
     │
     ▼
DMZ
10.50.30.1
     │
     │ forwarding
     ▼
ens5
     │
     ▼
10.50.20.32
```

This allowed the troubleshooting process to move beyond assumptions and observe the actual packet path.

---

# No ICMP Reply

The request was observed leaving the DMZ toward the private server, but an ICMP reply was not observed.

This was an important result.

It meant that the problem was not simply:

```text
Kali → WireGuard
```

because that part had already been verified.

The investigation therefore moved toward the private-side path.

Possible causes included:

* AWS Security Group behavior
* Host firewall rules
* ICMP filtering
* Return routing
* NAT behavior
* Service availability

---

# Testing the Actual Service

ICMP is not necessarily required for the service being tested.

The private server's SSH (Secure Shell) service runs on TCP (Transmission Control Protocol) port 22.

Therefore, TCP/22 was tested directly:

```bash
nc -vz -w 5 10.50.20.32 22
```

Result:

```text
10.50.20.32:22 open
```

This confirmed that the SSH service was reachable.

---

# Important Lesson: Ping Is Not Everything

A failed ping does not automatically mean:

```text
Host unreachable
```

It can mean:

```text
ICMP blocked
```

while another service remains reachable.

For example:

```text
ICMP
    ↓
Blocked

TCP/22
    ↓
Allowed
```

Therefore, connectivity testing should use the protocol and service that actually matters.

---

# Testing TCP Instead of ICMP

For SSH:

```bash
nc -vz -w 5 10.50.20.32 22
```

For HTTP:

```bash
nc -vz -w 5 10.50.20.32 3000
```

For HTTPS:

```bash
nc -vz -w 5 <target> 443
```

The exact test should match the service being investigated.

---

# Useful tcpdump Commands

## Capture all traffic on WireGuard

```bash
sudo tcpdump -ni wg0
```

---

## Capture traffic involving the private server

```bash
sudo tcpdump -ni wg0 host 10.50.20.32
```

---

## Capture private-server traffic on the AWS interface

```bash
sudo tcpdump -ni ens5 host 10.50.20.32
```

---

## Capture TCP/22 traffic

```bash
sudo tcpdump -ni ens5 tcp port 22
```

This is useful when investigating SSH connectivity.

---

## Capture ICMP traffic

```bash
sudo tcpdump -ni ens5 icmp
```

This is useful when investigating ping behavior.

---

# Understanding the `tcpdump` Filter

Example:

```bash
sudo tcpdump -ni ens5 host 10.50.20.32
```

The important parts are:

```text
-i ens5
```

Capture from the `ens5` interface.

```text
host 10.50.20.32
```

Only display packets involving the private server.

This prevents unrelated network traffic from making the output difficult to understand.

---

# Packet Direction

A useful mental model for BLACKFORGE is:

```text
                KALI
             10.50.30.2
                  │
                  │
             WireGuard
                  │
                  ▼
                 wg0
                  │
                  ▼
             ┌─────────┐
             │   DMZ   │
             │10.50.30.1
             └────┬────┘
                  │
                 ens5
                  │
                  ▼
          PRIVATE NETWORK
          10.50.20.0/24
                  │
                  ▼
          10.50.20.32
```

Packet capture can be performed at different points in this path.

---

# Troubleshooting Method

BLACKFORGE uses packet analysis as part of a structured troubleshooting process:

```text
Connection Failure
        ↓
Check IP Address
        ↓
Check Route
        ↓
Check Security Group
        ↓
Check Firewall
        ↓
Capture Packet
        ↓
Identify Interface
        ↓
Check Destination
        ↓
Check Return Traffic
        ↓
Test Actual Service
```

This is more reliable than repeatedly changing configuration without knowing where the packet is failing.

---

# Packet Analysis and Firewalls

Packet capture can help distinguish different situations.

## No packet observed

If a packet is not visible on the expected interface:

```text
Possible causes:
- Incorrect route
- Interface problem
- Local firewall
- Application never sent the traffic
```

---

## Packet leaves but no reply

If the request leaves the DMZ but no response returns:

```text
Possible causes:
- Destination firewall
- Security Group
- Service not responding
- Return route
- NAT problem
```

---

## Request and reply observed

If both are visible:

```text
Request
   ↓
Destination
   ↓
Reply
```

then the network path is functioning and the investigation should move higher in the stack.

---

# Packet Analysis as Security Evidence

Packet captures are useful not only for troubleshooting.

They can also provide evidence during security investigations.

Examples include:

```text
Port scanning
Connection attempts
Unexpected protocols
Lateral movement
Command-and-control traffic
Internal reconnaissance
Suspicious connections
```

This will become especially important during the BLACKFORGE detection and Purple Team phases.

---

# Future Detection Use

The same packet-analysis concepts can later be used to investigate attacks.

Example:

```text
Attacker
   ↓
Reconnaissance
   ↓
Network traffic
   ↓
Packet capture
   ↓
Observable behavior
   ↓
Detection
```

This connects the networking phase directly with the future detection phase.

---

# Lessons Learned

## 1. Observe Before Changing Configuration

Instead of immediately changing routes or firewall rules:

```text
Capture traffic
      ↓
Understand behavior
      ↓
Identify failure point
      ↓
Make targeted change
```

---

## 2. Interfaces Reveal Packet Movement

If traffic appears on:

```text
wg0
```

but does not appear on:

```text
ens5
```

the forwarding path inside the DMZ becomes the primary investigation area.

If traffic appears on:

```text
ens5
```

but no response returns, investigate the private-side path.

---

## 3. Packet Captures Provide Evidence

Configuration tells us what should happen.

Packet capture tells us what actually happened.

That distinction is extremely useful in network troubleshooting and security investigations.

---

# Verification

The BLACKFORGE networking phase successfully used packet capture to verify:

```text
WireGuard ingress             ✅
DMZ packet reception          ✅
DMZ forwarding               ✅
Private destination traffic   ✅
TCP/22 connectivity           ✅
ICMP behavior                 ✅
Network troubleshooting       ✅
```

---

# Final Lesson

Packet analysis is a core networking and security skill.

The BLACKFORGE lab uses `tcpdump` to understand:

```text
Where did the packet start?
        ↓
Which interface received it?
        ↓
Where was it forwarded?
        ↓
Did it reach the destination?
        ↓
Did a response return?
        ↓
If not, where did communication stop?
```

This approach provides a practical foundation for future:

* Network reconnaissance
* Attack detection
* Incident investigation
* Lateral movement analysis
* Network monitoring
* Purple Team exercises

```
```
