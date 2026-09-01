02-ssh-access — Secure SSH Access
────────────────────────────────

Administrative SSH access to BLACKFORGE AWS infrastructure
using a non-default port and key-based authentication.

---

## Architecture

```
Kali → AWS #1 (ip-172-26-1-48)  :2244
Kali → AWS #2 (ip-172-26-6-116) :2244
```

---

## Configuration

| Parameter           | Value                    |
| ------------------- | ------------------------ |
| SSH Port            | 2244                     |
| Authentication      | Key-based (PEM)          |
| Default port (22)   | Disabled                 |
| Source restriction  | Kali public IP           |

### AWS Security Group Rules

| Direction | Protocol | Port | Source          |
| --------- | -------- | ---- | --------------- |
| Inbound   | TCP      | 2244 | Kali public IP  |

Port 22 was removed from the security group after port 2244
was confirmed working end-to-end.

### Client SSH Config (~/.ssh/config)

```text
Host server1
    HostName <AWS-1-PUBLIC-IP>
    User ubuntu
    Port 2244
    IdentityFile ~/.ssh/LightsailDefaultKey-ap-southeast-1.pem

Host server2
    HostName <AWS-2-PUBLIC-IP>
    User ubuntu
    Port 2244
    IdentityFile ~/.ssh/LightsailDefaultKey-ap-southeast-1.pem
```

---

## Verification

### AWS #1 — ip-172-26-1-48

```text
ubuntu@ip-172-26-1-48:~$ hostname
ip-172-26-1-48

ubuntu@ip-172-26-1-48:~$ whoami
ubuntu

ubuntu@ip-172-26-1-48:~$ hostnamectl
 Static hostname: ip-172-26-1-48
       Icon name: computer-vm
         Chassis: vm
  Virtualization: amazon
Operating System: Ubuntu 24.04.4 LTS
          Kernel: Linux 6.17.0-1019-aws
    Architecture: x86-64
 Hardware Vendor: Amazon EC2
  Hardware Model: t3.micro

ubuntu@ip-172-26-1-48:~$ ip -br addr
lo               UNKNOWN   127.0.0.1/8 ::1/128
ens5             UP        172.26.1.48/20

ubuntu@ip-172-26-1-48:~$ sudo ss -tlnp | grep 2244
LISTEN 0  4096  0.0.0.0:2244  0.0.0.0:*  users:(("sshd",pid=77686,fd=3))
LISTEN 0  4096     [::]:2244     [::]:*  users:(("sshd",pid=77686,fd=4))

ubuntu@ip-172-26-1-48:~$ sudo ufw status verbose
Status: inactive
```

### AWS #2 — ip-172-26-6-116

```text
ubuntu@ip-172-26-6-116:~$ hostname
ip-172-26-6-116

ubuntu@ip-172-26-6-116:~$ whoami
ubuntu

ubuntu@ip-172-26-6-116:~$ hostnamectl
 Static hostname: ip-172-26-6-116
       Icon name: computer-vm
         Chassis: vm
  Virtualization: amazon
Operating System: Ubuntu 24.04.4 LTS
          Kernel: Linux 6.17.0-1019-aws
    Architecture: x86-64
 Hardware Vendor: Amazon EC2
  Hardware Model: t3.nano

ubuntu@ip-172-26-6-116:~$ ip -br addr
lo               UNKNOWN   127.0.0.1/8 ::1/128
ens5             UP        172.26.6.116/20

ubuntu@ip-172-26-6-116:~$ sudo ss -tlnp | grep 2244
LISTEN 0  4096  0.0.0.0:2244  0.0.0.0:*  users:(("sshd",pid=34198,fd=3))
LISTEN 0  4096     [::]:2244     [::]:*  users:(("sshd",pid=34198,fd=4))

ubuntu@ip-172-26-6-116:~$ sudo ufw status verbose
Status: inactive
```

Both servers are listening on port 2244 (IPv4 and IPv6).
UFW is inactive — access is controlled entirely by the AWS Security Group.

---

## Security Considerations

### SSH as an attack surface

SSH exposes a network-accessible authentication service. Any open port
running SSH is a target for credential brute-force, protocol exploits,
and misconfiguration abuse. Reducing exposure is the primary control.

### Why source restriction matters

Restricting inbound SSH to a known source IP (the Kali host) eliminates
the attack surface from the rest of the internet. An attacker cannot
attempt authentication if the Security Group drops their packets before
they reach sshd.

### Why key-based authentication is useful

Private keys are cryptographically stronger than passwords. They cannot
be guessed or brute-forced through the SSH protocol in any practical
sense. The key never leaves the client machine — only a signed
challenge crosses the wire.

### Why changing the port is not a security boundary

Port 2244 is not security. It is noise reduction. A scanner running
`nmap -p-` or `masscan` will find port 2244 in seconds. Port obscurity
stops only the most opportunistic automated scanners. The real controls
are: source restriction at the Security Group and key-based auth.

---

## Blue Team Perspective

### Authentication log location

```bash
sudo tail -f /var/log/auth.log          # Ubuntu (auth events)
sudo journalctl -u ssh -f               # systemd journal
```

### Failed authentication events

```text
sshd[PID]: Failed password for <user> from <IP> port <port> ssh2
sshd[PID]: Invalid user <user> from <IP> port <port>
sshd[PID]: Connection closed by invalid user <user> <IP> port <port> [preauth]
```

Repeated failed events from the same IP = brute-force attempt.

### Successful authentication events

```text
sshd[PID]: Accepted publickey for ubuntu from <IP> port <port> ssh2: RSA ...
sshd[PID]: pam_unix(sshd:session): session opened for user ubuntu by (uid=0)
```

### Monitoring opportunities

| Signal                        | Meaning                               |
| ----------------------------- | ------------------------------------- |
| Auth failures from new IP     | Opportunistic scan / brute-force      |
| Auth success from unknown IP  | Credential compromise / key leak      |
| Auth success at unusual hour  | Potential unauthorized access         |
| High frequency of preauth drops | Active scanner                      |
| New SSH session followed by privilege escalation commands | Lateral movement |

Actionable: alert on any successful auth from an IP that is not the
registered Kali source.

---

## Lessons Learned

- Always test the new connection before removing the old one.
  Losing both access paths on a remote server requires a rebuild.

- Changing `sshd_config` alone is not sufficient on Ubuntu 24.04.
  The `ssh.socket` systemd unit also controls which ports are bound.
  Both must be updated and reloaded.

- UFW inactive means the OS firewall is not running.
  On AWS, the Security Group is the perimeter — not the host firewall.

- Port obscurity (2244) stops automated noise, not targeted attackers.
  The security boundary is source IP restriction + key authentication.

- After modifying firewall rules, always review the full ruleset.
  It is easy to accidentally delete an unrelated rule.
