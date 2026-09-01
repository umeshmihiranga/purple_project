# BLACKFORGE Lab Journal

## Date

2026-09-01

## Lab

SSH Access & Infrastructure Baseline

## Phase

BLACKFORGE Phase 0 — Secure Access & Infrastructure Baseline

## Objective

Establish and verify SSH administrative access to both
AWS servers on TCP/2244 and begin the infrastructure baseline.

---

## Environment

| Field          | Value                                    |
| -------------- | ---------------------------------------- |
| Attacker       | Kali Linux (192.248.9.136)               |
| Target 1       | AWS #1 — ip-172-26-1-48 (172.26.1.48)   |
| Target 2       | AWS #2 — ip-172-26-6-116 (172.26.6.116) |
| SSH Port       | 2244                                     |
| Authentication | SSH key (RSA)                            |
| OS             | Ubuntu 24.04.4 LTS (both)               |
| Kernel         | 6.17.0-1019-aws (both)                  |
| Instance Type  | t3.micro (AWS #1), t3.nano (AWS #2)     |

---

## Initial State

...

---

## Reconnaissance

...

---

## Actions Performed

...

---

## Problems Encountered

...

---

## Troubleshooting

...

---

## Technical Understanding

### SSH socket state — AWS #1

AWS #1 was listening on both port 22 and port 2244 at time of capture.
This indicates SSH was mid-configuration or port 22 had not yet been removed.

```text
sudo ss -tulpn (AWS #1)

tcp  LISTEN  0.0.0.0:22    users:(("sshd",pid=77686,fd=5))
tcp  LISTEN  0.0.0.0:2244  users:(("sshd",pid=77686,fd=3))
tcp  LISTEN  [::]:22       users:(("sshd",pid=77686,fd=6))
tcp  LISTEN  [::]:2244     users:(("sshd",pid=77686,fd=4))
```

### SSH socket state — AWS #2

AWS #2 was listening on port 2244 only. Port 22 was absent.

```text
sudo ss -tulpn (AWS #2)

tcp  LISTEN  0.0.0.0:2244  users:(("sshd",pid=34198,fd=3))
tcp  LISTEN  [::]:2244     users:(("sshd",pid=34198,fd=4))
```

### UFW state — both servers

```text
Status: inactive
```

The OS firewall is not running. The network perimeter is the AWS Security Group.

### SSH service restart — AWS #1

The journal shows sshd was restarted at 10:39:48 during this session.

```text
Sep 01 10:39:48  sshd[75698]: Received signal 15; terminating.
Sep 01 10:39:48  systemd[1]: Stopped ssh.service
Sep 01 10:39:48  sshd[77686]: Server listening on 0.0.0.0 port 2244.
Sep 01 10:39:48  sshd[77686]: Server listening on :: port 2244.
Sep 01 10:39:48  sshd[77686]: Server listening on 0.0.0.0 port 22.
Sep 01 10:39:48  sshd[77686]: Server listening on :: port 22.
```

After restart both port 22 and 2244 were active, which is consistent
with the `ss` output captured at this time.

---

## Security Perspective

### 🔴 Red Team

Attack path confirmed:

```
Kali (192.248.9.136)
  │
  ├── ssh -p 2244 → AWS #1 (172.26.1.48)   ✓ Accepted publickey
  └── ssh -p 2244 → AWS #2 (172.26.6.116)  ✓ (to verify)
```

AWS Security Group is the only network perimeter — UFW is inactive on both hosts.
AWS #1 still has port 22 open. This is an unnecessary additional attack surface.

### 🔵 Blue Team

#### SSH journal — AWS #1

```text
sudo journalctl -u ssh --since "1 hour ago"
```

Full output:

```text
Sep 01 10:04:36  sshd[77381]: Connection closed by 143.105.203.122 port 55934
Sep 01 10:06:31  sshd[77385]: Connection closed by 143.105.203.122 port 24503
Sep 01 10:16:01  sshd[77397]: Connection closed by 181.111.98.146 port 47394
Sep 01 10:16:05  sshd[77398]: Connection closed by 181.111.98.146 port 50760
Sep 01 10:25:39  sshd[77438]: Accepted publickey for ubuntu from 192.248.9.136 port 61030 ssh2: RSA SHA256:MOzbPjtbuDEfzxzWAB3QuJHufGjfxRm56sH4eVnwkck
Sep 01 10:25:39  sshd[77438]: pam_unix(sshd:session): session opened for user ubuntu(uid=1000) by ubuntu(uid=0)
Sep 01 10:39:48  sshd[75698]: Received signal 15; terminating.
Sep 01 10:39:48  systemd[1]: Stopping ssh.service - OpenBSD Secure Shell server...
Sep 01 10:39:48  systemd[1]: ssh.service: Deactivated successfully.
Sep 01 10:39:48  systemd[1]: Stopped ssh.service - OpenBSD Secure Shell server.
Sep 01 10:39:48  sshd[77686]: Server listening on 0.0.0.0 port 2244.
Sep 01 10:39:48  sshd[77686]: Server listening on :: port 2244.
Sep 01 10:39:48  systemd[1]: Started ssh.service - OpenBSD Secure Shell server.
Sep 01 10:39:48  sshd[77686]: Server listening on 0.0.0.0 port 22.
Sep 01 10:39:48  sshd[77686]: Server listening on :: port 22.
Sep 01 10:40:48  sshd[77696]: Connection closed by authenticating user ubuntu 192.248.9.136 port 30897 [preauth]
Sep 01 10:41:31  sshd[77710]: Accepted publickey for ubuntu from 192.248.9.136 port 62647 ssh2: RSA SHA256:MOzbPjtbuDEfzxzWAB3QuJHufGjfxRm56sH4eVnwkck
Sep 01 10:41:31  sshd[77710]: pam_unix(sshd:session): session opened for user ubuntu(uid=1000) by ubuntu(uid=0)
Sep 01 10:46:22  sshd[77804]: Connection reset by 147.185.132.201 port 57412 [preauth]
Sep 01 10:50:43  sshd[77811]: Accepted publickey for ubuntu from 192.248.9.136 port 43939 ssh2: RSA SHA256:MOzbPjtbuDEfzxzWAB3QuJHufGjfxRm56sH4eVnwkck
Sep 01 10:50:43  sshd[77811]: pam_unix(sshd:session): session opened for user ubuntu(uid=1000) by ubuntu(uid=0)
Sep 01 10:55:52  sshd[77929]: Invalid user  from 47.253.138.43 port 56926
Sep 01 10:55:52  sshd[77929]: Connection closed by invalid user  47.253.138.43 port 56926 [preauth]
```

#### Event classification

| Time     | Source IP       | Classification          | Notes                                |
| -------- | --------------- | ----------------------- | ------------------------------------ |
| 10:04:36 | 143.105.203.122 | Unauthorized — scanner  | Two connections, no auth attempted   |
| 10:06:31 | 143.105.203.122 | Unauthorized — scanner  | Same source, second probe            |
| 10:16:01 | 181.111.98.146  | Unauthorized — scanner  | Two rapid connections                |
| 10:16:05 | 181.111.98.146  | Unauthorized — scanner  | Same source, second probe            |
| 10:25:39 | 192.248.9.136   | ✅ Authorized — Kali    | publickey accepted, session opened   |
| 10:39:48 | —               | SSH service restart     | Config change in progress            |
| 10:40:48 | 192.248.9.136   | Drop — preauth          | Key not yet loaded after restart     |
| 10:41:31 | 192.248.9.136   | ✅ Authorized — Kali    | publickey accepted, session opened   |
| 10:46:22 | 147.185.132.201 | Unauthorized — scanner  | Connection reset before auth         |
| 10:50:43 | 192.248.9.136   | ✅ Authorized — Kali    | publickey accepted, session opened   |
| 10:55:52 | 47.253.138.43   | Unauthorized — scanner  | Invalid user, no username supplied   |

#### Kali key fingerprint

```text
RSA SHA256:MOzbPjtbuDEfzxzWAB3QuJHufGjfxRm56sH4eVnwkck
```

#### Observation

The server is reachable from the internet despite port 2244 being non-default.
Four unique external IPs probed the host within one hour without being
blocked at the network layer. This is consistent with internet-wide port
scanning (e.g. Shodan, Masscan, Censys). The AWS Security Group is not
restricting inbound to Kali-only, or it was not in place at capture time.

### SSH journal — AWS #2

```text
ubuntu@ip-172-26-6-116:~$ sudo journalctl -u ssh --since "1 hour ago"
Sep 01 10:45:00 ip-172-26-6-116 sshd[34595]: Accepted publickey for ubuntu from 192.248.9.136 port 2008 ssh2: RSA SHA256:MOzbPjtbuDEfzxzWAB3QuJHufGjfxRm56sH4eVnwkck
Sep 01 10:45:00 ip-172-26-6-116 sshd[34595]: pam_unix(sshd:session): session opened for user ubuntu(uid=1000) by ubuntu(uid=0)
```

#### Event classification

| Time     | Source IP     | Classification       | Notes                              |
| -------- | ------------- | -------------------- | ---------------------------------- |
| 10:45:00 | 192.248.9.136 | ✅ Authorized — Kali | publickey accepted, session opened |

#### Observation

AWS #2 shows a clean log — one entry, one authorized connection, zero scanner
noise. Contrast with AWS #1 which had four external IPs probing it.

Likely reason: AWS #2 has no port 22 open. Many internet scanners target
port 22 specifically. With only port 2244 exposed (and depending on the
Security Group state at the time), AWS #2 attracted no unauthorized traffic
during this window.

Both servers share the same Kali key fingerprint:

```text
RSA SHA256:MOzbPjtbuDEfzxzWAB3QuJHufGjfxRm56sH4eVnwkck
```

---

## Evidence

### AWS #1 network state

```text
ubuntu@ip-172-26-1-48:~$ hostname
ip-172-26-1-48

ubuntu@ip-172-26-1-48:~$ ip -br addr
lo    UNKNOWN  127.0.0.1/8 ::1/128
ens5  UP       172.26.1.48/20

ubuntu@ip-172-26-1-48:~$ ip route
default via 172.26.0.1 dev ens5
172.26.0.0/20 dev ens5 src 172.26.1.48

ubuntu@ip-172-26-1-48:~$ sudo ss -tlnp | grep 2244
LISTEN 0  4096  0.0.0.0:2244  0.0.0.0:*  users:(("sshd",pid=77686,fd=3))
LISTEN 0  4096     [::]:2244     [::]:*  users:(("sshd",pid=77686,fd=4))

ubuntu@ip-172-26-1-48:~$ sudo ufw status verbose
Status: inactive
```

### AWS #2 network state

```text
ubuntu@ip-172-26-6-116:~$ hostname
ip-172-26-6-116

ubuntu@ip-172-26-6-116:~$ ip -br addr
lo    UNKNOWN  127.0.0.1/8 ::1/128
ens5  UP       172.26.6.116/20

ubuntu@ip-172-26-6-116:~$ ip route
default via 172.26.0.1 dev ens5
172.26.0.0/20 dev ens5 src 172.26.6.116

ubuntu@ip-172-26-6-116:~$ sudo ss -tlnp | grep 2244
LISTEN 0  4096  0.0.0.0:2244  0.0.0.0:*  users:(("sshd",pid=34198,fd=3))
LISTEN 0  4096     [::]:2244     [::]:*  users:(("sshd",pid=34198,fd=4))

ubuntu@ip-172-26-6-116:~$ sudo ufw status verbose
Status: inactive
```

---

## Lessons Learned

- AWS #1 still had port 22 open at time of capture. Port 22 should be
  removed from `sshd_config` and the Security Group once port 2244 is
  confirmed stable.

- The preauth drop at 10:40:48 after the SSH restart shows that the
  client reconnection failed before the key was accepted. This is
  expected behaviour during a service restart but worth noting.

- Internet scanners found port 2244 within minutes of it being open.
  Port obscurity does not provide meaningful protection. Security Group
  source restriction is the real control.

- Both servers share the same subnet (172.26.0.0/20) and the same
  default gateway (172.26.0.1). Lateral movement between them is
  possible at the network layer without additional segmentation.

---

## What I Still Don't Understand

...

---

## Status

- [x] SSH #1 verified
- [x] SSH #2 verified
- [ ] Network baseline collected
- [x] SSH logs examined (AWS #1 + AWS #2)
- [ ] Connectivity matrix completed
- [ ] Phase 0 exit criteria completed
