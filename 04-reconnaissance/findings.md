# BLACKFORGE — Consolidated Reconnaissance Findings & Attack Surface Map

> **Phase:** Phase 04 — Reconnaissance & Discovery  
> **Environment:** BLACKFORGE Lab (`13.229.212.104` / `10.50.0.0/16`)  
> **Source Documents:** [`nmap.md`](file:///home/umesh/git_projects/BLACKFORGE/04-reconnaissance/nmap.md) & [`web-enumeration.md`](file:///home/umesh/git_projects/BLACKFORGE/04-reconnaissance/web-enumeration.md)  
> **Date:** 2026-09-29  
> **Status:** Complete & Verified Live

---

## 1. Executive Summary

During Phase 04 (Reconnaissance & Discovery), the perimeter and application layer of the BLACKFORGE environment were thoroughly assessed through both offensive probing and defensive log correlation.

The assessment mapped the complete external attack surface, confirmed network-level filtering, evaluated WAF inspection behavior, and discovered critical unauthenticated sensitive resources and configuration endpoints.

```text
                                  ATTACK SURFACE MAP
                                          │
                    ┌─────────────────────┴─────────────────────┐
                    ▼                                           ▼
          NETWORK PERIMETER (L3/L4)                   WEB APPLICATION (L7)
          Target: 13.229.212.104                      Target: Nginx / Juice Shop
          - ICMP: Dropped (AWS SG)                    - Nginx 1.24.0 (Ubuntu)
          - TCP 80: Open (Reverse Proxy)              - Unauthenticated /ftp directory listing
          - TCP 443: Closed (TCP RST)                 - Downloadable KeePass vault (.kdbx)
          - TCP 22: Filtered (VPN only)               - 41 Client-side routes in main.js
          - UDP 51820: Filtered / Open                - 35+ Backend REST API endpoints
                                                      - Unauthenticated config dump (/rest/admin)
```

---

## 2. Attack Surface Inventory

### 2.1 Perimeter & Network Services

| Endpoint / Port | Protocol | State | Exposed Service | Identification & Boundary Control |
|---|---|:---:|---|---|
| `13.229.212.104` | ICMP | `filtered` | None | 100% dropped by AWS Security Group. |
| `13.229.212.104:80` | TCP | `open` | HTTP | Nginx 1.24.0 reverse proxy + ModSecurity WAF. |
| `13.229.212.104:443` | TCP | `closed` | HTTPS | Allowed by AWS SG, but kernel returns TCP RST (no listener). |
| `13.229.212.104:22` | TCP | `filtered` | SSH | Silently dropped by AWS SG (SSH bound to port 2244 on VPN). |
| `13.229.212.104:51820`| UDP | `open\|filtered` | WireGuard | VPN management gateway (cryptokey routing required). |

### 2.2 Web Application & API Surface

| Component | Architecture / Technology | Key Observations & Endpoints |
|---|---|---|
| **Edge Proxy** | Nginx 1.24.0 (Ubuntu) | Reverse proxy terminating port 80, proxying to `10.50.20.32:3000`. |
| **WAF Engine** | ModSecurity v3 + OWASP CRS v3.3.5 | Inbound anomaly scoring threshold = 5. Blocks `.bak` and scanner user-agents. |
| **Frontend UI** | Angular Single-Page Application (SPA) | Routes defined with ES6 backticks in `main.js`: `/score-board`, `/administration`, `/accounting`, `/data-export`. |
| **Backend API** | Node.js / Express / Sequelize ORM | REST APIs under `/rest/` and `/api/`: `/rest/admin/*`, `/api/Users`, `/rest/products/search`. |
| **Static Storage** | Public `/ftp` Directory | Directory listing enabled. Exposes `.kdbx`, `.bak`, `.pyc`, and `.md` files. |

---

## 3. Discovered Vulnerabilities & Weaknesses Matrix

| ID | Severity | Finding Name | Affected Asset | Impact |
|---|:---:|---|---|---|
| **VULN-01** | **HIGH** | Unauthenticated KeePass Vault Download | `GET /ftp/incident-support.kdbx` | Allows external attackers to download encrypted internal password database without authentication. |
| **VULN-02** | **MEDIUM** | Unauthenticated Runtime Configuration Disclosure | `GET /rest/admin/application-configuration` | Discloses internal ports, domain configuration, and feature flags in raw JSON. |
| **VULN-03** | **MEDIUM** | Unrestricted Directory Listing on `/ftp` | `GET /ftp` | Exposes internal file naming, backup files, and compiled bytecode (`encrypt.pyc`). |
| **VULN-04** | **LOW** | Detailed Web Server & OS Banner Leakage | `Server` Header | Exposes exact software version (`nginx/1.24.0 (Ubuntu)`), facilitating vulnerability targeting. |
| **VULN-05** | **LOW** | Internal Route Leakage in HTTP Headers | `X-Recruiting` Header | Exposes application route `/#/jobs` in response headers. |
| **VULN-06** | **LOW** | Overly Permissive CORS Policy | `Access-Control-Allow-Origin: *` | Permits cross-origin requests from any arbitrary domain. |

---

## 4. Defensive & Detection Telemetry Evaluation

| Offensive Recon Action | Defensive Telemetry Produced | Effectiveness Evaluation |
|---|---|---|
| **Port Scanning (SYN scan)** | AWS Security Group drops packets; no OS or WAF logs generated. | **Effective**: Prevents perimeter noise from reaching application logs. |
| **TCP Port 443 Probe** | Host kernel sends TCP RST/ACK. | **Improvement Area**: Port 443 is open in SG but unused. Closing it prevents probe fingerprinting. |
| **Accessing `/ftp/package.json.bak`** | ModSecurity logged Rule `949110` (Anomaly score: 10) in `/var/log/nginx/error.log`; HTTP 403 returned. | **Effective**: CRS restricted extension rule successfully prevented backup file download. |
| **Accessing `/ftp/incident-support.kdbx`** | Logged as standard `200 OK` in `/var/log/nginx/access.log`; 0 anomaly score. | **Defensive Gap**: `.kdbx` bypassed WAF inspection due to missing extension signature in CRS. |
| **Accessing `/api/Users` without JWT** | Application returned `401 Unauthorized` (`UnauthorizedError`). | **Effective**: Application middleware properly validates bearer tokens. |

---

## 5. Transition to Phase 05: Vulnerability Assessment

The reconnaissance findings directly inform the target list for **Phase 05 (Vulnerability Assessment)**:

```text
┌────────────────────────────────────────────────────────┐
│           PHASE 05 TARGETED ATTACK VECTORS             │
├────────────────────────────────────────────────────────┤
│ 1. Password Vault Analysis: Offline cracking of        │
│    the downloaded incident-support.kdbx file.          │
│                                                        │
│ 2. SQL Injection Testing: Testing /rest/products/search │
│    with WAF evasion and CRS threshold bypass.          │
│                                                        │
│ 3. Broken Authentication / IDOR: Testing the unauth    │
│    configuration endpoints and /api/Users endpoints.   │
│                                                        │
│ 4. Client Route Exploitation: Testing access to the    │
│    /#/administration and /#/score-board interfaces.    │
└────────────────────────────────────────────────────────┘
```