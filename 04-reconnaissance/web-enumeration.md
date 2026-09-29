# BLACKFORGE — Web Application Reconnaissance & Surface Mapping

> **Document:** `04-reconnaissance/web-enumeration.md`  
> **Target:** `http://13.229.212.104/` (`BLACKFORGE-DMZ-01` → `BLACKFORGE-PRIVATE-01:3000`)  
> **Source Host:** Kali Linux (`10.50.30.2` / Public Client)  
> **DMZ Host:** `BLACKFORGE-DMZ-01` (`10.50.10.116`)  
> **Date:** 2026-09-29  
> **Phase:** Phase 04 — Reconnaissance & Discovery  
> **Focus:** Web Server Profiling, Directory Discovery, Client Bundle Mining, API Surface Mapping, and WAF Telemetry Correlation

---

## 1. Phase Overview & Objectives

In this phase, reconnaissance transitions from Layer 3/4 network scanning to **Layer 7 web application enumeration**.

The target web endpoint (`http://13.229.212.104:80`) is fronted by an Nginx reverse proxy running on the DMZ host with **ModSecurity v3 WAF** and the **OWASP Core Rule Set (CRS) v3.3.5**, proxying requests across the VPC to an isolated Docker container running **OWASP Juice Shop** on the private subnet host (`10.50.20.32:3000`).

### Purple Team Objectives:
1. **Offensive Perspective:**
   - Fingerprint web server banners and exposed technologies.
   - Enumerate directories and identify leaked sensitive resources.
   - Analyze client-side Single-Page Application (SPA) bundles to uncover hidden navigation routes and backend REST API endpoints.
   - Test unauthenticated API boundaries for information disclosure.
2. **Defensive Perspective:**
   - Correlate offensive scanner probes with DMZ telemetry (`/var/log/nginx/access.log` and `/var/log/nginx/error.log`).
   - Validate ModSecurity anomaly scoring and identify what malicious or non-standard probes trigger blocks versus what bypasses inspection.

---

## 2. Test Execution & Verified Observations

### Test REC-WEB-01: HTTP Header & Server Banner Inspection

**Objective:** Observe web server software disclosure, proxy headers, and security controls in HTTP responses.

**Command:**
```bash
curl -I -s http://13.229.212.104/
```

**Observed Output:**
```http
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Tue, 29 Sep 2026 03:41:46 GMT
Content-Type: text/html; charset=UTF-8
Content-Length: 9393
Connection: keep-alive
Access-Control-Allow-Origin: *
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Feature-Policy: payment 'self'
X-Recruiting: /#/jobs
Accept-Ranges: bytes
Cache-Control: public, max-age=0
Last-Modified: Sat, 05 Sep 2026 09:44:40 GMT
ETag: W/"24b1-1a070f4b3fc"
Vary: Accept-Encoding
```

**Purple Team Analysis:**
- **Banner Leakage:** `Server: nginx/1.24.0 (Ubuntu)` directly discloses web server vendor, minor version, and host OS distribution. Enables targeted CVE lookups.
- **Route Disclosure:** `X-Recruiting: /#/jobs` leaks an application route directly within the HTTP headers.
- **Permissive CORS:** `Access-Control-Allow-Origin: *` allows any domain to issue cross-origin requests.
- **Defensive Headers Present:** `X-Content-Type-Options: nosniff` (MIME sniffing mitigation) and `X-Frame-Options: SAMEORIGIN` (clickjacking mitigation).
- **Missing Controls:** Unencrypted HTTP traffic (no TLS / HSTS), and no `Content-Security-Policy (CSP)`.

---

### Test REC-WEB-02: Crawler Directives (`robots.txt`)

**Objective:** Inspect search crawler instructions for intentionally excluded directories.

**Command:**
```bash
curl -s http://13.229.212.104/robots.txt
```

**Observed Output:**
```text
User-agent: *
Disallow: /ftp
```

**Purple Team Analysis:**
- Discloses the presence of the `/ftp` path, highlighting it as an immediate reconnaissance target.

---

### Test REC-WEB-03: Directory Listing on `/ftp`

**Objective:** Verify access permissions and directory browsing on the `/ftp` path.

**Command:**
```bash
curl -s http://13.229.212.104/ftp
```

**Observed Output (Snippet):**
```html
<title>listing directory /ftp</title>
...
<ul id="files" class="view-tiles">
  <li><a href="ftp/quarantine" class="icon icon-directory">quarantine</a></li>
  <li><a href="ftp/acquisitions.md" class="icon icon-text">acquisitions.md</a></li>
  <li><a href="ftp/announcement_encrypted.md" class="icon icon-text">announcement_encrypted.md</a></li>
  <li><a href="ftp/coupons_2013.md.bak" class="icon icon-default">coupons_2013.md.bak</a></li>
  <li><a href="ftp/eastere.gg" class="icon icon-default">eastere.gg</a></li>
  <li><a href="ftp/encrypt.pyc" class="icon icon-default">encrypt.pyc</a></li>
  <li><a href="ftp/incident-support.kdbx" class="icon icon-default">incident-support.kdbx</a></li>
  <li><a href="ftp/legal.md" class="icon icon-text">legal.md</a></li>
  <li><a href="ftp/package-lock.json.bak" class="icon icon-default">package-lock.json.bak</a></li>
  <li><a href="ftp/package.json.bak" class="icon icon-default">package.json.bak</a></li>
  <li><a href="ftp/suspicious_errors.yml" class="icon icon-text">suspicious_errors.yml</a></li>
</ul>
```

**Purple Team Analysis:**
- Unauthenticated directory listing is enabled on the backend.
- Exposes critical sensitive artifacts: KeePass database (`incident-support.kdbx`), Python bytecode (`encrypt.pyc`), and configuration backups (`package.json.bak`).

---

### Test REC-WEB-04: File Retrieval & Access Controls (`legal.md` vs `package.json.bak`)

**Objective:** Test whether files in `/ftp` can be downloaded directly.

**Command 1 (Allowed File):**
```bash
curl -s -i http://13.229.212.104/ftp/legal.md | head -n 10
```
- **Observed Result:** `HTTP/1.1 200 OK` (full document contents returned).

**Command 2 (Restricted Extension Probe):**
```bash
curl -s -i http://13.229.212.104/ftp/package.json.bak
```

**Observed Output:**
```http
HTTP/1.1 403 Forbidden
Server: nginx
Date: Tue, 29 Sep 2026 04:01:10 GMT
Content-Type: text/html
Content-Length: 146
Connection: keep-alive

<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

---

### Test REC-WEB-05: Blue Team WAF Telemetry Verification

**Objective:** Determine the source of the `403 Forbidden` response for `.bak`.

**Command (Executed on DMZ Host `BLACKFORGE-DMZ-01`):**
```bash
sudo tail -n 25 /var/log/nginx/error.log
```

**Observed Log Output:**
```text
2026/09/29 04:01:10 [error] 186581#186581: *2138 [client 192.248.9.142] ModSecurity: Access denied with code 403 (phase 2). Matched "Operator `Ge' with parameter `5' against variable `TX:ANOMALY_SCORE' (Value: `10' ) [file "/usr/share/modsecurity-crs/rules/REQUEST-949-BLOCKING-EVALUATION.conf"] [line "81"] [id "949110"] [rev ""] [msg "Inbound Anomaly Score Exceeded (Total Score: 10)"] [data ""] [severity "2"] [ver "OWASP_CRS/3.3.5"] [hostname "10.50.10.116"] [uri "/ftp/package.json.bak"] [unique_id "179065447041.533977"], client: 192.248.9.142, server: _, request: "GET /ftp/package.json.bak HTTP/1.1", host: "13.229.212.104", referrer: "http://13.229.212.104/ftp"
```

**Purple Team Analysis:**
- **WAF Rule Triggered:** The request triggered OWASP CRS rule `920440` (restricted extension `.bak`), accumulating an anomaly score of **10** (threshold is 5).
- **Enforcement Point:** ModSecurity blocked the request at Phase 2 and returned an Nginx 403 error page before reaching the backend Juice Shop container.

---

### Test REC-WEB-06: Password Vault Download Bypass (`incident-support.kdbx`)

**Objective:** Evaluate if sensitive extensions not present in the default CRS list can be retrieved.

**Command:**
```bash
curl -s -i http://13.229.212.104/ftp/incident-support.kdbx
```

**Observed Output:**
```http
HTTP/1.1 200 OK
Server: nginx
Date: Tue, 29 Sep 2026 04:03:18 GMT
Content-Type: application/octet-stream
Content-Length: 3246
Connection: keep-alive
Access-Control-Allow-Origin: *
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Feature-Policy: payment 'self'
X-Recruiting: /#/jobs
Accept-Ranges: bytes
Cache-Control: public, max-age=0
Last-Modified: Tue, 11 Aug 2026 05:21:02 GMT
ETag: W/"cae-19fef445a30"
```

**Critical Purple Team Finding:**
- **Defense Gap:** While `.bak` is blocked by default CRS rules, `.kdbx` (KeePass Password Database) is **not included in CRS rule 920440**.
- **Impact:** An external attacker can download the internal password vault file (`3,246 bytes`) directly over HTTP without credentials.
- **Remediation:** Add `.kdbx` to the custom restricted extensions policy or disable direct access to `/ftp` at the Nginx reverse proxy level.

---

### Test REC-WEB-07: Client-Side Script Discovery

**Objective:** Identify JavaScript bundles responsible for frontend SPA routing.

**Command:**
```bash
curl -s http://13.229.212.104/ | grep -oE '<script src="[^"]+"'
```

**Observed Output:**
```html
<script src="polyfills.js"
<script src="scripts.js"
<script src="main.js"
```

---

### Test REC-WEB-08: Client-Side Route Mining (`main.js`)

**Objective:** Extract application navigation routes from the compiled Angular bundle.

**Command:**
```bash
curl -s http://13.229.212.104/main.js | grep -oE 'path:`[^`]+`' | sort -u
```

**Extracted Routes:**
```text
path:`**`
path:`2fa/enter`
path:`403`
path:`about`
path:`accounting`
path:`address/create`
path:`address/edit/:addressId`
path:`address/saved`
path:`address/select`
path:`administration`
path:`basket`
path:`bee-haven`
path:`change-password`
path:`chatbot`
path:`coding-challenge/:challengeKey`
path:`complain`
path:`contact`
path:`conversation/:id`
path:`data-export`
path:`delivery-method`
path:`deluxe-membership`
path:`/engine.io`
path:`forgot-password`
path:`hacking-instructor`
path:`juicy-nft`
path:`last-login-ip`
path:`login`
path:`order-completion/:id`
path:`order-history`
path:`order-summary`
path:`payment/:entity`
path:`photo-wall`
path:`privacy-policy`
path:`privacy-security`
path:`recycle`
path:`register`
path:`saved-payment-methods`
path:`score-board`
path:`search`
path:`track-result`
path:`track-result/new`
path:`two-factor-authentication`
path:`wallet`
path:`wallet-web3`
path:`web3-sandbox`
```

**Key Discovered Interfaces:**
- `http://13.229.212.104/#/administration` (Admin user management)
- `http://13.229.212.104/#/score-board` (Challenge progress tracker)
- `http://13.229.212.104/#/accounting` (Order and transaction auditing)
- `http://13.229.212.104/#/data-export` (Data export interface)

---

### Test REC-WEB-09: Backend REST API Mining (`main.js`)

**Objective:** Map out all backend API endpoints called by the SPA client.

**Command:**
```bash
curl -s http://13.229.212.104/main.js | grep -oE "[\`'\"](/rest/|/api/)[^\`'\"]+[\`'\"]" | sort -u | head -n 35
```

**Extracted Endpoints:**
```text
`/api/Addresss`
`/api/BasketItems`
`/api/Cards`
`/api/Challenges`
`/api/Challenges/?key=nftMintChallenge`
`/api/Complaints`
`/api/Deliverys`
`/api/Feedbacks`
`/api/Hints`
`/api/Products`
`/api/Quantitys`
`/api/Recycles`
`/api/SecurityAnswers`
`/api/SecurityQuestions`
`/api/Users`
`/rest/admin`
`/rest/captcha`
`/rest/chat`
`/rest/continue-code`
`/rest/continue-code/apply/`
`/rest/continue-code-findIt`
`/rest/continue-code-findIt/apply/`
`/rest/continue-code-fixIt`
`/rest/continue-code-fixIt/apply/`
`/rest/country-mapping`
`/rest/deluxe-membership`
`/rest/image-captcha/`
`/rest/memories`
`/rest/order-history`
`/rest/products`
`/rest/repeat-notification`
`/rest/saveLoginIp`
`/rest/track-order`
`/rest/user`
`/rest/user/authentication-details/`
```

---

### Test REC-WEB-10: Information Disclosure Verification (`/rest/admin/application-configuration`)

**Objective:** Test whether sensitive administrative API routes can be accessed without authentication.

**Command:**
```bash
curl -s http://13.229.212.104/rest/admin/application-configuration | head -c 300
```

**Observed Output:**
```json
{"config":{"server":{"port":3000,"basePath":"","baseUrl":"http://localhost:3000"},"application":{"domain":"juice-sh.op","name":"OWASP Juice Shop","logo":"JuiceShop_Logo.png","favicon":"favicon_js.ico","theme":"bluegrey-lightgreen","showVersionNumber":true,"showGitHubLinks":true,"localBackupEnabled":
```

**Purple Team Analysis:**
- **Finding:** The endpoint `/rest/admin/application-configuration` allows unauthenticated access.
- **Disclosure:** Dumps backend server ports, internal application domains, and feature flag settings directly to any unauthenticated client.

---

### Test REC-WEB-11: Authentication Boundary Verification (`/api/Users`)

**Objective:** Test whether user objects can be read without credentials.

**Command:**
```bash
curl -s -i http://13.229.212.104/api/Users | head -n 25
```

**Observed Output:**
```http
HTTP/1.1 401 Unauthorized
Server: nginx
Date: Tue, 29 Sep 2026 04:45:07 GMT
Content-Type: text/html; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Access-Control-Allow-Origin: *
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Feature-Policy: payment 'self'
X-Recruiting: /#/jobs
Vary: Accept-Encoding

<html>
  <head>
    <meta charset='utf-8'> 
    <title>UnauthorizedError: No Authorization header was found</title>
```

**Purple Team Analysis:**
- **Defensive Behavior:** Access control is strictly enforced on `/api/Users`.
- **Mechanism:** JWT authentication middleware validates the `Authorization: Bearer <token>` header and returns `401 Unauthorized` when absent.

---

## 3. Telemetry & Attack Surface Summary Matrix

| Probe / Action | Target URI | HTTP Status | WAF Score / Rule | Telemetry Location | Finding Summary |
|---|---|:---:|:---:|---|---|
| HTTP Headers | `/` | `200` | Score 0 | Nginx Access Log | `nginx/1.24.0 (Ubuntu)` banner and `/#/jobs` disclosed |
| Robot Policy | `/robots.txt` | `200` | Score 0 | Nginx Access Log | Exposes `/ftp` path |
| Directory Index | `/ftp` | `200` | Score 0 | Nginx Access Log | Full directory listing enabled |
| Document Download | `/ftp/legal.md` | `200` | Score 0 | Nginx Access Log | Allowed |
| Backup Extension | `/ftp/package.json.bak` | `403` | Score 10 (Rule 949110) | Nginx Error Log | Blocked by WAF restricted extension policy |
| Password Vault | `/ftp/incident-support.kdbx` | `200` | Score 0 | Nginx Access Log | **CRITICAL LEAK**: `.kdbx` file downloaded without credentials |
| Client Route Mining | `/main.js` | `200` | Score 0 | Nginx Access Log | 41 SPA routes discovered (`administration`, `score-board`) |
| API Route Mining | `/main.js` | `200` | Score 0 | Nginx Access Log | 35+ REST APIs identified (`/api/Users`, `/rest/admin`) |
| Config Disclosure | `/rest/admin/application-configuration` | `200` | Score 0 | Nginx Access Log | **INFO DISCLOSURE**: Unauthenticated config JSON dump |
| User Table Query | `/api/Users` | `401` | Score 0 | Nginx Access Log | Backend middleware rejects unauthenticated queries |

---

## 4. Remediation & Hardening Actions

1. **Suppress Web Server Banners:**
   Add `server_tokens off;` in `/etc/nginx/nginx.conf` on `BLACKFORGE-DMZ-01`.
2. **Restrict Access to `/ftp` Directory:**
   Block direct access to the `/ftp` directory in Nginx:
   ```nginx
   location /ftp {
       deny all;
       return 404;
   }
   ```
3. **Extend WAF Extension Policy:**
   Add `.kdbx` to the ModSecurity restricted extensions list in `/etc/modsecurity/crs-setup.conf` or a custom rule:
   ```apache
   SecRule REQUEST_FILENAME "@rx \.(?i:kdbx)$" \
       "id:1000001,phase:2,deny,status:403,log,msg:'Restricted KeePass Database Extension'"
   ```
4. **Enforce Authentication on Administrative Endpoints:**
   Require authentication tokens for `/rest/admin/*` routes.
