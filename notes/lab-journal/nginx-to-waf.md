
# BLACKFORGE — Nginx Reverse Proxy to Web Application Firewall

> **Project:** BLACKFORGE  
> **Environment:** AWS Singapore (`ap-southeast-1`)  
> **Purpose:** Purple Team cybersecurity lab  
> **Application:** OWASP Juice Shop  
> **Reverse Proxy:** Nginx  
> **Web Application Firewall (WAF):** ModSecurity  
> **Rule Set:** OWASP Core Rule Set (CRS)

---

# 1. Objective

The objective of this stage of BLACKFORGE was to build a controlled public-facing web application architecture where:

```text
Internet
   |
   | HTTP
   v
BLACKFORGE-DMZ-01
Nginx Reverse Proxy
ModSecurity WAF
   |
   | HTTP
   v
BLACKFORGE-PRIVATE-01
OWASP Juice Shop
Docker :3000
````

The private application should not be directly exposed to the Internet.

Instead, all public HTTP traffic should enter through the DMZ server and pass through:

1. Nginx
2. ModSecurity
3. OWASP CRS (Core Rule Set)
4. Private Juice Shop application

This gives BLACKFORGE a controlled security boundary where malicious HTTP requests can be detected and blocked before reaching the application.

---

# 2. BLACKFORGE Network Architecture

## AWS Region

```text
ap-southeast-1
Singapore
```

## VPC

```text
BLACKFORGE-VPC
10.50.0.0/16
```

## DMZ Subnet

```text
BLACKFORGE-DMZ
10.50.10.0/24
```

Server:

```text
BLACKFORGE-DMZ-01

Private IP:
10.50.10.116

Public IP:
13.229.212.104
```

## Private Subnet

```text
BLACKFORGE-PRIVATE
10.50.20.0/24
```

Server:

```text
BLACKFORGE-PRIVATE-01

Private IP:
10.50.20.32

Public IP:
None
```

---

# 3. Network Traffic Flow

Normal web traffic follows this path:

```text
Kali Linux
10.16.204.46
       |
       | HTTP :80
       v
13.229.212.104
       |
       v
BLACKFORGE-DMZ-01
       |
       | Nginx
       | ModSecurity
       | OWASP CRS
       v
10.50.20.32:3000
       |
       v
OWASP Juice Shop
```

The important security concept is:

> The application server is private. Nginx is the controlled public entry point.

---

# 4. Why Use a Reverse Proxy?

A reverse proxy is a server that accepts client requests and forwards them to an internal application server.

In BLACKFORGE:

```text
Client
   |
   v
Nginx
   |
   v
Juice Shop
```

The client does not directly communicate with:

```text
10.50.20.32:3000
```

Instead, the client communicates with:

```text
13.229.212.104:80
```

Nginx receives the request and forwards it internally.

This gives us a security control point.

Future controls can be placed at this point:

```text
Internet
   |
   v
Nginx
   |
   +-- TLS (Transport Layer Security)
   |
   +-- WAF (Web Application Firewall)
   |
   +-- Rate Limiting
   |
   +-- Security Headers
   |
   +-- Logging
   |
   v
Private Application
```

---

# 5. Installing Nginx

Nginx was installed on the DMZ server.

```bash
sudo apt update
sudo apt install -y nginx
```

Verify:

```bash
sudo systemctl status nginx --no-pager
```

Expected:

```text
Active: active (running)
```

---

# 6. Nginx Reverse Proxy Configuration

The BLACKFORGE Nginx site configuration:

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://10.50.20.32:3000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

# 7. Understanding the Nginx Configuration

## `listen 80`

```nginx
listen 80;
```

Nginx listens for HTTP (Hypertext Transfer Protocol) traffic on TCP port 80.

---

## `server_name _`

```nginx
server_name _;
```

The `_` acts as a catch-all server name.

This allows the server block to respond even when the request does not contain a configured domain name.

---

## `location /`

```nginx
location / {
```

This matches requests beginning at the root of the website.

For example:

```text
/
```

```text
/rest/products/search
```

```text
/api/Users
```

---

## `proxy_pass`

```nginx
proxy_pass http://10.50.20.32:3000;
```

This tells Nginx:

> Forward the request to the Juice Shop server.

Therefore:

```text
Client
   |
   | HTTP
   v
Nginx :80
   |
   | HTTP
   v
10.50.20.32:3000
```

---

# 8. Proxy Headers

Nginx also sends important information to the backend.

## Host

```nginx
proxy_set_header Host $host;
```

Passes the original Host header.

---

## X-Real-IP

```nginx
proxy_set_header X-Real-IP $remote_addr;
```

Passes the client's IP address.

---

## X-Forwarded-For

```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

Maintains the client IP forwarding chain.

This is important when applications need to know the actual client IP rather than only seeing the reverse proxy.

---

## X-Forwarded-Proto

```nginx
proxy_set_header X-Forwarded-Proto $scheme;
```

Tells the backend whether the original request used:

```text
http
```

or:

```text
https
```

---

# 9. Nginx Configuration Validation

Before applying configuration changes:

```bash
sudo nginx -t
```

Expected:

```text
syntax is ok
test is successful
```

Then reload:

```bash
sudo systemctl reload nginx
```

Reloading applies the configuration without unnecessarily stopping the service.

---

# 10. Deploying OWASP Juice Shop

The intentionally vulnerable application used by BLACKFORGE is OWASP Juice Shop.

It runs inside Docker on the private server.

```bash
docker run -d \
  --name juice-shop \
  -p 3000:3000 \
  bkimminich/juice-shop
```

The application listens on:

```text
10.50.20.32:3000
```

It is not assigned a public IP address.

---

# 11. Why Juice Shop Is Private

The application is intentionally vulnerable.

Therefore, exposing it directly to the Internet would create an unnecessary security risk.

Instead:

```text
Internet
    |
    v
DMZ
    |
    | Security controls
    v
Private Juice Shop
```

The DMZ server becomes the security boundary.

---

# 12. Testing the Reverse Proxy

From Kali:

```bash
curl -I http://13.229.212.104
```

Result:

```text
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Content-Type: text/html; charset=UTF-8
```

This proves that:

```text
Kali
  |
  v
Public IP
  |
  v
Nginx
  |
  v
Juice Shop
```

is working.

---

# 13. Full HTTP Verification

Run:

```bash
curl -v http://13.229.212.104
```

The response contained:

```html
<title>OWASP Juice Shop</title>
```

This confirmed that the application being served through the public Nginx endpoint is OWASP Juice Shop.

The response also exposed application headers such as:

```text
Access-Control-Allow-Origin: *
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Feature-Policy: payment 'self'
X-Recruiting: /#/jobs
```

These headers can later be investigated as part of the BLACKFORGE web security assessment.

---

# 14. Introducing ModSecurity

ModSecurity is an open-source Web Application Firewall (WAF).

A WAF analyzes HTTP requests and responses and can detect patterns associated with attacks against web applications.

Examples include:

```text
SQL Injection (SQLi)
Cross-Site Scripting (XSS)
Local File Inclusion (LFI)
Remote File Inclusion (RFI)
Remote Code Execution (RCE)
Protocol attacks
```

In BLACKFORGE:

```text
Internet
   |
   v
Nginx
   |
   v
ModSecurity
   |
   v
OWASP CRS
   |
   v
Juice Shop
```

---

# 15. Installing ModSecurity and OWASP CRS

Packages installed:

```bash
sudo apt install -y \
    libnginx-mod-http-modsecurity \
    modsecurity-crs
```

Installed components include:

```text
libmodsecurity3
libnginx-mod-http-modsecurity
modsecurity-crs
```

---

# 16. ModSecurity Nginx Module

The Nginx ModSecurity module was enabled through:

```text
/etc/nginx/modules-enabled/50-mod-http-modsecurity.conf
```

The configuration contains:

```nginx
load_module modules/ngx_http_modsecurity_module.so;
```

This loads the ModSecurity module into Nginx.

---

# 17. ModSecurity Configuration

The main BLACKFORGE ModSecurity configuration is:

```text
/etc/nginx/modsecurity.conf
```

Current baseline:

```apache
SecRuleEngine On

SecRequestBodyAccess On

SecPcreMatchLimit 1000
SecPcreMatchLimitRecursion 1000

SecAuditEngine RelevantOnly
SecAuditLogRelevantStatus "^(?:5|4(?!04))"

SecAuditLogParts ABIJDEFHZ
SecAuditLogType Serial
SecAuditLog /var/log/nginx/modsec_audit.log

SecUnicodeMapFile unicode.mapping 20127

Include /etc/nginx/crs-blackforge.conf
```

---

# 18. Understanding `SecRuleEngine`

The most important setting is:

```apache
SecRuleEngine
```

It controls how ModSecurity processes rules.

Common modes include:

```text
DetectionOnly
On
Off
```

---

## DetectionOnly

```apache
SecRuleEngine DetectionOnly
```

The WAF evaluates requests but does not actively block them.

This is useful during initial deployment.

Conceptually:

```text
Request
   |
   v
WAF
   |
   +--> Detect attack
   |
   +--> Log attack
   |
   v
Application still receives request
```

---

## On

```apache
SecRuleEngine On
```

The WAF can actively interfere with malicious requests.

Conceptually:

```text
Request
   |
   v
WAF
   |
   +--> Legitimate ---> Application
   |
   +--> Malicious ---> BLOCK
```

BLACKFORGE is currently configured with:

```apache
SecRuleEngine On
```

---

# 19. Audit Logging

ModSecurity logs relevant security events to:

```text
/var/log/nginx/modsec_audit.log
```

The audit log provides evidence of:

```text
Which rule triggered
What request triggered it
Which parameter contained suspicious data
What URI was requested
What action was taken
```

This is important for the Blue Team side of BLACKFORGE.

---

# 20. OWASP Core Rule Set

OWASP CRS (Core Rule Set) is a collection of generic security rules designed to detect common web attacks.

The CRS rules are located under:

```text
/usr/share/modsecurity-crs/rules/
```

Important rule categories include:

```text
REQUEST-920-PROTOCOL-ENFORCEMENT.conf
REQUEST-921-PROTOCOL-ATTACK.conf
REQUEST-930-APPLICATION-ATTACK-LFI.conf
REQUEST-931-APPLICATION-ATTACK-RFI.conf
REQUEST-932-APPLICATION-ATTACK-RCE.conf
REQUEST-941-APPLICATION-ATTACK-XSS.conf
REQUEST-942-APPLICATION-ATTACK-SQLI.conf
REQUEST-943-APPLICATION-ATTACK-SESSION-FIXATION.conf
REQUEST-944-APPLICATION-ATTACK-JAVA.conf
REQUEST-949-BLOCKING-EVALUATION.conf
```

---

# 21. CRS Loader

The packaged CRS loader is:

```text
/usr/share/modsecurity-crs/owasp-crs.load
```

It contains Apache-style directives such as:

```apache
Include
IncludeOptional
```

However, the Nginx ModSecurity parser did not accept the `IncludeOptional` directive in this setup.

Therefore, a Nginx-compatible CRS loader was created.

---

# 22. BLACKFORGE CRS Loader

File:

```text
/etc/nginx/crs-blackforge.conf
```

Contents:

```apache
Include /etc/modsecurity/crs/crs-setup.conf
Include /etc/modsecurity/crs/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
Include /usr/share/modsecurity-crs/rules/*.conf
Include /etc/modsecurity/crs/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf
```

This loads:

1. CRS setup
2. Local pre-CRS exclusions
3. CRS rules
4. Local post-CRS exclusions

---

# 23. Loading the CRS into ModSecurity

The loader is included from:

```text
/etc/nginx/modsecurity.conf
```

Using:

```apache
Include /etc/nginx/crs-blackforge.conf
```

Therefore the rule-loading chain is:

```text
modsecurity.conf
       |
       v
crs-blackforge.conf
       |
       +--> crs-setup.conf
       |
       +--> REQUEST-900...
       |
       +--> CRS rules
       |
       +--> RESPONSE-999...
```

---

# 24. Verifying CRS Rule Loading

Run:

```bash
sudo nginx -t
```

The successful test showed:

```text
ModSecurity-nginx v1.0.3
rules loaded inline/local/remote: 0/921/0
syntax is ok
test is successful
```

This confirmed that the CRS rules were successfully loaded.

---

# 25. Attaching ModSecurity to Nginx

The Nginx server block contains:

```nginx
modsecurity on;
modsecurity_rules_file /etc/nginx/modsecurity.conf;
```

This means:

```text
Nginx
   |
   v
ModSecurity enabled
   |
   v
/etc/nginx/modsecurity.conf
```

The configuration then loads the CRS.

---

# 26. Testing Normal Traffic

From Kali:

```bash
curl -I http://13.229.212.104
```

Initially, the WAF detected:

```text
[id "920350"]
[msg "Host header is a numeric IP address"]
```

The reason was that the request used:

```text
Host: 13.229.212.104
```

The CRS rule identifies a numeric IP address in the Host header.

---

# 27. Understanding CRS Rule 920350

The rule was inspected with:

```bash
sudo grep -n -A 12 -B 5 \
'id:920350' \
/usr/share/modsecurity-crs/rules/REQUEST-920-PROTOCOL-ENFORCEMENT.conf
```

The relevant rule:

```apache
SecRule REQUEST_HEADERS:Host "@rx ^[\d.:]+$" \
    "id:920350,\
    phase:2,\
    block,\
    t:none,\
    msg:'Host header is a numeric IP address',\
    ..."
```

---

# 28. Understanding the Rule

## `REQUEST_HEADERS:Host`

This tells ModSecurity to inspect:

```text
Host
```

from the HTTP request.

---

## `@rx`

```text
@rx
```

means regular expression matching.

---

## Regular Expression

```text
^[\d.:]+$
```

This matches a Host value containing only:

```text
digits
.
:
```

Therefore:

```text
13.229.212.104
```

matches.

So does:

```text
13.229.212.104:80
```

---

# 29. Why This Was a False Positive

Using a raw public IP address is legitimate in our BLACKFORGE lab.

The WAF interpreted it as suspicious because production websites normally use hostnames such as:

```text
example.com
```

rather than:

```text
13.229.212.104
```

Therefore, this was treated as a lab-specific false positive.

The correct solution was not to disable the entire CRS rule.

Instead, a targeted exclusion was created.

---

# 30. CRS Exclusion Strategy

The CRS provides:

```text
REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
```

This file is intended for local exclusions that must happen before CRS rules execute.

This is important because:

> Security exclusions should be as narrow as possible.

We do not want to disable protection globally.

Bad approach:

```text
Disable CRS
```

Better approach:

```text
Disable only rule 920350
for only our known Host header
```

---

# 31. BLACKFORGE Custom Exclusion

The following rule was added to:

```text
/etc/modsecurity/crs/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
```

```apache
SecRule REQUEST_HEADERS:Host "@streq 13.229.212.104" \
    "id:1001,\
    phase:1,\
    pass,\
    nolog,\
    ctl:ruleRemoveById=920350"
```

---

# 32. Understanding the Custom Rule

## Rule ID

```text
id:1001
```

This is the local BLACKFORGE rule identifier.

---

## `phase:1`

```text
phase:1
```

The rule executes during the request-header processing phase.

This allows the exclusion to be applied before the CRS rule that we want to remove.

---

## `@streq`

```text
@streq 13.229.212.104
```

Means:

> Match the Host header exactly against this value.

This is much narrower than matching every request.

---

## `ctl:ruleRemoveById`

```text
ctl:ruleRemoveById=920350
```

This tells ModSecurity to remove rule:

```text
920350
```

for the current transaction.

It does **not** disable the entire CRS.

---

## `nolog`

```text
nolog
```

Prevents the custom exclusion rule itself from creating unnecessary audit log entries.

---

# 33. Testing the Exclusion

After modifying the CRS exclusion file:

```bash
sudo nginx -t
```

Then:

```bash
sudo systemctl reload nginx
```

For clean testing, the previous audit log was cleared:

```bash
sudo truncate -s 0 /var/log/nginx/modsec_audit.log
```

Then:

```bash
curl -I http://13.229.212.104
```

The request returned:

```text
HTTP/1.1 200 OK
```

The custom rule was temporarily changed to logging mode during troubleshooting.

The audit log showed:

```text
[id "1001"]
[file "/etc/modsecurity/crs/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf"]
[uri "/"]
```

Most importantly, there was no new:

```text
920350
```

event.

Therefore:

```text
920350 successfully excluded
```

while the rest of the CRS remained active.

---

# 34. Why We Must Not Disable 920350 Globally

A global rule removal would look conceptually like:

```text
Remove rule 920350 everywhere
```

That is undesirable because it reduces protection for every request.

Our solution instead says:

```text
IF Host == 13.229.212.104
THEN remove 920350 for this request
```

This is a much more controlled security exception.

---

# 35. SQL Injection Testing

After confirming the false-positive exclusion, the next test was SQL Injection (SQLi).

SQL Injection is an attack where an attacker attempts to manipulate an application's database query through untrusted input.

The Juice Shop endpoint used for testing was:

```text
/rest/products/search
```

The parameter:

```text
q
```

is processed by the application.

---

# 36. Initial SQLi Test

A harmless test was first attempted:

```bash
curl -i \
"http://13.229.212.104/rest/products/search?q='"
```

This did not trigger the SQL Injection rule.

This demonstrates an important principle:

> A single suspicious character is not necessarily enough for a WAF to classify a request as SQL Injection.

---

# 37. Controlled SQLi Test

A controlled SQL Injection test was performed:

```bash
curl -i \
"http://13.229.212.104/rest/products/search?q=%27%20OR%201%3D1--"
```

URL-decoded, the parameter is approximately:

```text
' OR 1=1--
```

This is a classic SQL Injection pattern used in the controlled lab environment.

---

# 38. SQLi Detection in DetectionOnly Mode

When ModSecurity was configured as:

```apache
SecRuleEngine DetectionOnly
```

the WAF detected the request but allowed it to continue.

The audit log showed:

```text
[id "942100"]
[msg "SQL Injection Attack Detected via libinjection"]
[data "Matched Data: s&1c found within ARGS:q: ' OR 1=1--"]
[severity "2"]
[uri "/rest/products/search"]
```

This demonstrated that the WAF was successfully identifying the attack.

However, because the engine was in:

```text
DetectionOnly
```

the request still reached Juice Shop.

---

# 39. Application-Side Result

Juice Shop returned:

```text
HTTP/1.1 500 Internal Server Error
```

with an error indicating:

```text
SQLITE_ERROR: incomplete input
```

This was an important security observation.

The WAF detected the attack, but in detection-only mode the request reached the application.

The application then generated a database error.

This demonstrates why detection alone is not equivalent to prevention.

---

# 40. Detection vs Blocking

## DetectionOnly

```text
Attacker
   |
   v
WAF
   |
   +--> Detect
   |
   +--> Log
   |
   v
Application
```

The application receives the request.

---

## Blocking

```text
Attacker
   |
   v
WAF
   |
   +--> Detect
   |
   +--> Increase anomaly score
   |
   +--> Block
   |
   X
```

The malicious request never reaches the application.

---

# 41. Enabling Blocking Mode

The configuration was changed from:

```apache
SecRuleEngine DetectionOnly
```

to:

```apache
SecRuleEngine On
```

Then:

```bash
sudo nginx -t
```

and:

```bash
sudo systemctl reload nginx
```

---

# 42. SQLi Blocking Test

The same controlled request was executed:

```bash
curl -i \
"http://13.229.212.104/rest/products/search?q=%27%20OR%201%3D1--"
```

The response changed to:

```text
HTTP/1.1 403 Forbidden
```

This demonstrated that the WAF was now actively blocking the request.

---

# 43. Understanding Rule 942100

The audit log showed:

```text
[id "942100"]
[msg "SQL Injection Attack Detected via libinjection"]
```

Rule:

```text
942100
```

belongs to:

```text
REQUEST-942-APPLICATION-ATTACK-SQLI.conf
```

This rule detects SQL Injection patterns.

---

# 44. Libinjection

The SQL Injection rule used:

```text
libinjection
```

Libinjection is a detection engine designed to identify SQL Injection and Cross-Site Scripting patterns.

In this test, it identified the SQL Injection pattern inside:

```text
ARGS:q
```

which represents the HTTP request parameter:

```text
q
```

---

# 45. Anomaly Scoring

The audit log also showed:

```text
[id "949110"]
[msg "Inbound Anomaly Score Exceeded"]
```

and:

```text
TX:ANOMALY_SCORE = 5
```

The CRS uses anomaly scoring rather than relying only on a single rule to decide whether a request should be blocked.

Conceptually:

```text
Request
   |
   +--> Rule 942100
   |       |
   |       +--> SQLi detected
   |       +--> Score increases
   |
   v
949110
   |
   +--> Evaluate total score
   |
   +--> Score exceeds threshold
   |
   v
BLOCK
```

---

# 46. Why 949110 Appears

The rule:

```text
949110
```

is part of the blocking evaluation stage.

It evaluates the accumulated inbound anomaly score.

In the final SQLi test:

```text
ANOMALY_SCORE = 5
```

The score exceeded the configured blocking threshold.

Therefore:

```text
HTTP 403 Forbidden
```

was returned.

---

# 47. Final SQLi Verification

The final controlled SQL Injection test produced:

```text
HTTP/1.1 403 Forbidden
```

The audit log contained:

```text
[id "1001"]
```

followed by:

```text
[id "942100"]
SQL Injection Attack Detected via libinjection
```

and:

```text
[id "949110"]
Inbound Anomaly Score Exceeded
```

with:

```text
TX:ANOMALY_SCORE = 5
```

This proved two things simultaneously:

1. The `920350` false positive was successfully excluded.
2. SQL Injection protection remained active.

---

# 48. Important Security Lesson

The exclusion:

```text
ctl:ruleRemoveById=920350
```

did not disable ModSecurity.

It only removed one specific rule for a matching request.

The SQL Injection rule:

```text
942100
```

continued to operate.

Therefore:

```text
Legitimate lab traffic
        |
        v
920350 excluded
        |
        v
Allowed


SQLi traffic
        |
        v
942100 detects SQLi
        |
        v
949110 evaluates anomaly score
        |
        v
403 Forbidden
```

This is exactly the behavior we wanted.

---

# 49. Security Boundary After WAF Deployment

The BLACKFORGE web architecture is now:

```text
                    INTERNET
                        |
                        |
                  Public IP
              13.229.212.104
                        |
                        v
             +-------------------+
             | BLACKFORGE-DMZ-01 |
             |                   |
             | Nginx             |
             | ModSecurity       |
             | OWASP CRS         |
             +-------------------+
                        |
                        |
                Private Network
                        |
                        v
             +-------------------+
             | BLACKFORGE-       |
             | PRIVATE-01        |
             |                   |
             | Docker            |
             | Juice Shop :3000  |
             +-------------------+
```

---

# 50. Current ModSecurity Processing Flow

A normal request:

```text
Client
  |
  v
Nginx
  |
  v
ModSecurity
  |
  v
CRS
  |
  v
No malicious pattern
  |
  v
Juice Shop
```

A malicious request:

```text
Client
  |
  v
Nginx
  |
  v
ModSecurity
  |
  v
CRS
  |
  +--> Rule detects attack
  |
  +--> Anomaly score increases
  |
  v
Blocking evaluation
  |
  v
403 Forbidden
```

---

# 51. WAF Logging

Audit log:

```text
/var/log/nginx/modsec_audit.log
```

Useful command:

```bash
sudo tail -f /var/log/nginx/modsec_audit.log
```

This allows the Blue Team to observe WAF events in real time.

For example:

```text
942100
```

indicates SQL Injection detection.

```text
949110
```

indicates blocking evaluation based on anomaly score.

---

# 52. Useful Investigation Commands

Check Nginx configuration:

```bash
sudo nginx -t
```

Check Nginx service:

```bash
sudo systemctl status nginx --no-pager
```

Check Nginx listening ports:

```bash
sudo ss -lntup
```

View ModSecurity audit log:

```bash
sudo less /var/log/nginx/modsec_audit.log
```

Follow ModSecurity events:

```bash
sudo tail -f /var/log/nginx/modsec_audit.log
```

Search for SQL Injection detections:

```bash
sudo grep -i "942100" /var/log/nginx/modsec_audit.log
```

Search for anomaly blocking:

```bash
sudo grep -i "949110" /var/log/nginx/modsec_audit.log
```

Search for the false-positive rule:

```bash
sudo grep -i "920350" /var/log/nginx/modsec_audit.log
```

---

# 53. Configuration Files

Important BLACKFORGE files:

```text
/etc/nginx/sites-available/blackforge
```

Nginx reverse proxy configuration.

```text
/etc/nginx/modsecurity.conf
```

Main ModSecurity configuration.

```text
/etc/nginx/crs-blackforge.conf
```

BLACKFORGE CRS loader.

```text
/etc/modsecurity/crs/crs-setup.conf
```

CRS configuration.

```text
/etc/modsecurity/crs/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
```

Local pre-CRS exclusions.

```text
/etc/modsecurity/crs/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf
```

Local post-CRS exclusions.

```text
/usr/share/modsecurity-crs/rules/
```

OWASP CRS rules.

```text
/var/log/nginx/modsec_audit.log
```

ModSecurity audit log.

---

# 54. Current BLACKFORGE WAF Configuration

## ModSecurity

```text
Enabled
```

## Rule Engine

```text
On
```

## OWASP CRS

```text
Loaded
```

## CRS Rules Loaded

```text
921+ rules
```

## Audit Logging

```text
Enabled
```

## SQL Injection Detection

```text
Enabled
```

## SQL Injection Blocking

```text
Verified
```

## 920350 False Positive

```text
Excluded for BLACKFORGE public IP
```

## Application

```text
OWASP Juice Shop
```

## Application Exposure

```text
Private only
```

---

# 55. Verification Summary

| Test                             | Result |
| -------------------------------- | ------ |
| Nginx installed                  | PASS   |
| Nginx running                    | PASS   |
| Reverse proxy configured         | PASS   |
| Public HTTP access               | PASS   |
| Private Juice Shop               | PASS   |
| ModSecurity installed            | PASS   |
| ModSecurity enabled              | PASS   |
| OWASP CRS loaded                 | PASS   |
| CRS rules verified               | PASS   |
| Audit logging                    | PASS   |
| SQL Injection detection          | PASS   |
| SQL Injection blocking           | PASS   |
| Anomaly scoring                  | PASS   |
| 920350 false positive identified | PASS   |
| Targeted 920350 exclusion        | PASS   |
| SQLi protection after exclusion  | PASS   |

---

# 56. Lessons Learned

## Reverse Proxy

A reverse proxy allows the public-facing server to control access to an internal application.

```text
Client -> Nginx -> Application
```

---

## WAF

A Web Application Firewall analyzes HTTP traffic for malicious patterns.

```text
Client -> WAF -> Application
```

---

## Detection vs Prevention

Detection-only mode:

```text
Detect + Log
```

Blocking mode:

```text
Detect + Log + Block
```

---

## False Positives

Security rules can sometimes identify legitimate traffic as suspicious.

The correct response is usually:

```text
Understand the rule
       |
       v
Confirm false positive
       |
       v
Create narrow exception
       |
       v
Retest security controls
```

Not:

```text
Disable WAF
```

---

## Anomaly Scoring

CRS can combine multiple rule detections into an overall transaction score.

```text
Detection
    |
    v
Anomaly Score
    |
    v
Blocking Evaluation
    |
    v
Allow / Block
```

---

## Layered Security

The BLACKFORGE architecture now uses multiple layers:

```text
AWS Security Group
        |
        v
Network Segmentation
        |
        v
Private Application
        |
        v
Nginx Reverse Proxy
        |
        v
ModSecurity WAF
        |
        v
OWASP CRS
        |
        v
Application
```

No single control is expected to provide complete protection.

---

# 57. Purple Team Perspective

BLACKFORGE is designed to demonstrate both offensive and defensive perspectives.

## Red Team

The attacker can:

```text
Send malicious HTTP requests
        |
        v
Attempt SQL Injection
        |
        v
Observe responses
```

## Blue Team

The defender can:

```text
Monitor WAF logs
        |
        v
Identify triggered rules
        |
        v
Investigate anomaly scores
        |
        v
Tune false positives
        |
        v
Verify blocking
```

## Purple Team

The same attack is used to validate the defensive control:

```text
Attack
  |
  v
Detection
  |
  v
Investigation
  |
  v
Tuning
  |
  v
Retest
  |
  v
Verified Defense
```

---

# 58. Current Project Status

### Network

```text
[✓] AWS VPC
[✓] DMZ subnet
[✓] Private subnet
[✓] DMZ routing
[✓] Private routing
[✓] IPv4 forwarding
[✓] NAT
```

### SSH

```text
[✓] DMZ SSH access
[✓] Private SSH through ProxyJump
```

### Application

```text
[✓] Docker
[✓] OWASP Juice Shop
[✓] Private application exposure
```

### Reverse Proxy

```text
[✓] Nginx installed
[✓] Reverse proxy configured
[✓] Nginx configuration validated
[✓] Public -> Nginx -> private application verified
```

### WAF

```text
[✓] ModSecurity installed
[✓] ModSecurity Nginx module enabled
[✓] OWASP CRS installed
[✓] CRS loaded
[✓] Audit logging enabled
[✓] DetectionOnly tested
[✓] Blocking mode tested
[✓] SQL Injection detection verified
[✓] SQL Injection blocking verified
[✓] Anomaly scoring observed
[✓] False positive identified
[✓] Targeted CRS exclusion implemented
[✓] SQL Injection protection verified after exclusion
```

---

# 59. Next BLACKFORGE WAF Test

The next planned security test is:

```text
Cross-Site Scripting (XSS)
```

Relevant OWASP CRS rules are located in:

```text
REQUEST-941-APPLICATION-ATTACK-XSS.conf
```

The next stage will follow the same methodology:

```text
1. Understand XSS theory
2. Identify a Juice Shop input
3. Send a controlled XSS test
4. Observe ModSecurity
5. Identify the CRS rule
6. Read the rule
7. Understand the detection
8. Test DetectionOnly behavior
9. Test blocking behavior
10. Inspect the audit log
11. Investigate false positives if any
12. Document the result
```

---

# 60. WAF Milestone

The BLACKFORGE WAF milestone is now:

```text
Nginx
   |
   v
ModSecurity
   |
   v
OWASP CRS
   |
   +--> False Positive Handling
   |
   +--> SQL Injection Detection
   |
   +--> Anomaly Scoring
   |
   +--> SQL Injection Blocking
   |
   v
Private Juice Shop
```

### Status

```text
CONFIGURED   ✓
VERIFIED     ✓
UNDERSTOOD   ✓
DOCUMENTED   ✓
```