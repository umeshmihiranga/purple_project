# BLACKFORGE — Reconnaissance & Attack Surface Mapping

> **Phase 04:** Reconnaissance & Discovery  
> **Environment:** BLACKFORGE Lab  
> **Focus:** External & Internal Attack Surface Discovery, Service Identification, Web Enumeration, and Detection Correlation

---

## 1. Phase Overview

The **Reconnaissance** phase represents the initial engagement stage of the BLACKFORGE security lifecycle. The objective is to identify reachable services, map network boundaries, enumerate web endpoints, and evaluate what information is leaked to an external or internal adversary.

From a **Purple Team** perspective, reconnaissance is evaluated from both sides:
1. **Offensive Perspective:** What ports, banners, endpoints, technologies, and parameters can be discovered?
2. **Defensive Perspective:** How does the defensive stack (AWS Security Groups, nftables, Nginx, ModSecurity WAF) handle and log scanning traffic?

---

## 2. Reconnaissance Workflow

```text
                     RECONNAISSANCE
                           │
       ┌───────────────────┴───────────────────┐
       ▼                                       ▼
  EXTERNAL RECON                          INTERNAL RECON
  (Internet → DMZ)                        (Kali via VPN → Private)
       │                                       │
  ┌────┴─────────────────────────┐        ┌────┴─────────────────────────┐
  │ 1. Port & Service Discovery  │        │ 1. Internal Host Discovery   │
  │ 2. Banner Grabbing           │        │ 2. Direct Service Checks     │
  │ 3. Web Directory & API Enum  │        │ 3. Docker Exposure Checks    │
  └──────────────────────────────┘        └──────────────────────────────┘
       │                                       │
       └───────────────────┬───────────────────┘
                           │
                           ▼
                 DETECTION CORRELATION
              (Nginx, ModSecurity, VPC)
```

---

## 3. Scope & Target Matrix

| Target ID | Target Identifier | Network Context | Expected Exposed Services | Primary Defensive Controls |
|---|---|---|---|---|
| `DMZ-EXT` | `13.229.212.104` | External Internet | TCP 80 (HTTP), UDP 51820 (WireGuard) | AWS Security Groups, nftables, ModSecurity |
| `DMZ-INT` | `10.50.30.1` | WireGuard VPN | TCP 22 (SSH), TCP 80 (Local) | nftables, SSH Key Auth |
| `PRIV-APP`| `10.50.20.32` | Private Subnet | TCP 22 (SSH via Jump), TCP 3000 (Juice Shop) | AWS Private SG, DMZ Router |

---

## 4. Sub-Modules in this Directory

- [nmap.md](file:///home/umesh/git_projects/BLACKFORGE/04-reconnaissance/nmap.md) — Network port scanning, service version detection, and firewall filtering analysis.
- [web-enumeration.md](file:///home/umesh/git_projects/BLACKFORGE/04-reconnaissance/web-enumeration.md) — Web application surface mapping, directory discovery, API route enumeration, and WAF interaction.
- [findings.md](file:///home/umesh/git_projects/BLACKFORGE/04-reconnaissance/findings.md) — Consolidated reconnaissance findings, identified weaknesses, and attack surface summary.

---

## 5. Purple Team Questions to Answer

- Does full port scanning against the DMZ trigger any defensive alerts?
- Are filtered ports silently dropped by AWS Security Groups or rejected with TCP RST?
- What server banners are leaked by Nginx (e.g., `Server: nginx`, OS versions)?
- Does ModSecurity inspect or block high-frequency URL discovery or directory fuzzing?
