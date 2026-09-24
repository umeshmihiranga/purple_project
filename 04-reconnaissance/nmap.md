# BLACKFORGE — Network Reconnaissance & Port Scanning

> **Document:** `04-reconnaissance/nmap.md`  
> **Target:** `13.229.212.104` (`ec2-13-229-212-104.ap-southeast-1.compute.amazonaws.com`)  
> **Source Host:** Kali Linux (`10.50.30.2` / Public ISP)  
> **Date:** 2026-09-24  
> **Objective:** Map the external perimeter, identify active network services, and evaluate firewall filtering characteristics.

---

## 1. Test Objectives

1. Determine whether ICMP Echo (ping) is accepted or dropped by the edge security groups.
2. Enumerate accessible TCP listening services on the DMZ public IP.
3. Detect service versions and server banners exposed to external clients.
4. Evaluate differences between `filtered`, `closed`, and `open` states across the perimeter.
5. Correlate scanning traffic with defensive telemetry (AWS Security Groups, Nginx, ModSecurity).

---

## 2. Test Plan & Execution

| Test ID | Test Category | Target Endpoint | Description |
|---|---|---|---|
| `REC-NET-01` | Host Reachability / ICMP | `13.229.212.104` | Test standard ICMP ping response. |
| `REC-NET-02` | TCP Port Scan | `13.229.212.104` | SYN scan across standard 1,000 TCP ports. |
| `REC-NET-03` | Service & Version Detection | `13.229.212.104:80` | Grab HTTP server header and protocol version. |

---

## 3. Command Execution & Observations

### Test REC-NET-01: Host Reachability (ICMP)

**Command:**
```bash
ping -c 4 13.229.212.104
```

**Observed Output:**
```text
PING 13.229.212.104 (13.229.212.104) 56(84) bytes of data.

--- 13.229.212.104 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3058ms
```

**Analysis:**
- 100% packet loss confirms that unsolicited ICMP Echo requests are dropped by the AWS Security Group at the hypervisor boundary.
- The host does not respond to ICMP sweeps, requiring TCP/UDP probes for discovery.

---

### Test REC-NET-02: TCP Port Scan

**Command:**
```bash
nmap -sS -Pn -T4 13.229.212.104
```

**Observed Output:**
```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-24 13:20 +0530
Nmap scan report for ec2-13-229-212-104.ap-southeast-1.compute.amazonaws.com (13.229.212.104)
Host is up (0.089s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT    STATE  SERVICE
80/tcp  open   http
443/tcp closed https

Nmap done: 1 IP address (1 host up) scanned in 9.60 seconds
```

**Analysis:**
- **998 Filtered Ports:** Packets received no response. AWS Security Group silently dropped the incoming TCP SYN packets (Default Deny).
- **Port 80/tcp (HTTP) — `open`:** Completes the TCP three-way handshake with Nginx reverse proxy.
- **Port 443/tcp (HTTPS) — `closed`:**
  - Notice that port 443 is `closed`, not `filtered`.
  - This indicates the AWS Security Group allows inbound port 443, so the packet reaches the DMZ instance OS.
  - Because no service is listening on port 443, the DMZ kernel returns a TCP RST/ACK.

---

### Test REC-NET-03: Service & Version Identification

**Command:**
```bash
nmap -sV -p 80 -Pn 13.229.212.104
```

**Observed Output:**
```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-24 13:20 +0530
Nmap scan report for ec2-13-229-212-104.ap-southeast-1.compute.amazonaws.com (13.229.212.104)
Host is up (0.096s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.19 seconds
```

**Analysis:**
- The banner discloses both web server software and version: `nginx 1.24.0 (Ubuntu)`.
- Operating system disclosure: Ubuntu Linux.
- **Defensive Note:** Server version tokens can be disabled in Nginx using `server_tokens off;` to minimize banner leakage.

---

## 4. Results Matrix

| Port | Protocol | State | Identified Service | Version / Banner Leaked | Edge Control |
|---|---|---|---|---|---|
| **80** | TCP | `open` | HTTP | `nginx 1.24.0 (Ubuntu)` | Allowed by SG, routed to Nginx |
| **443** | TCP | `closed` | HTTPS | None (TCP RST) | Allowed by SG, no listening daemon |
| **22** | TCP | `filtered` | SSH | None | Dropped by AWS SG (VPN-only access) |
| **51820** | UDP | `open\|filtered` | WireGuard | None | Handshake required to respond |
| **All Others** | TCP/UDP | `filtered` | - | None | Dropped by AWS SG (Default Deny) |

---

## 5. Defensive & Detection Telemetry Analysis

Inspection of DMZ logs during the network reconnaissance tests:

1. **Nginx Access Log (`/var/log/nginx/access.log`):**
   - Raw TCP SYN scans against port 80 do not produce access log entries because no valid HTTP request was negotiated.
2. **ModSecurity WAF Log (`/var/log/nginx/error.log`):**
   - ModSecurity did not alert on the port scan, as it operates strictly at Layer 7.
3. **WAF Baseline Anomaly Observation:**
   - Reviewing `error.log` during this phase highlighted an existing false positive: Juice Shop's real-time Socket.IO polling (`POST /socket.io/`) triggers ModSecurity rule `949110` (Inbound Anomaly Score Exceeded), returning `HTTP 403`. This is documented for defensive tuning.

---

## 6. Summary & Recommendations

- **Perimeter Posture:** Strong isolation. Only port 80 is reachable and responsive from the external Internet.
- **Remediation / Hardening Opportunities:**
  - Close or remove port 443 from `BLACKFORGE-DMZ-SG` until TLS certificates and HTTPS listeners are configured (avoids `closed` state probe leakage).
  - Add `server_tokens off;` to `/etc/nginx/nginx.conf` on DMZ to suppress `nginx 1.24.0 (Ubuntu)` version exposure.
