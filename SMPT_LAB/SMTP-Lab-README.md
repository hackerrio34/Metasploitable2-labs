# Metasploitable 2 — SMTP Penetration Test Lab Writeup

> **Disclaimer:** This penetration test was performed on a deliberately vulnerable virtual machine (Metasploitable 2) in an isolated lab environment. All techniques demonstrated here are for educational purposes only. Never attempt these techniques on systems you do not own or have explicit written permission to test.

---

## Lab Environment

| Item | Detail |
|---|---|
| Attacker Machine | Kali Linux (Nmap 7.95 / Metasploit Framework) |
| Target Machine | Metasploitable 2 (VirtualBox VM) |
| Target IP | 192.168.0.102 |
| Scope | SMTP service (port 25) only |
| Goal | Enumerate valid OS users via SMTP misconfiguration |

---

## Table of Contents

1. [What is SMTP?](#what-is-smtp)
2. [Phase 1 — Reconnaissance](#phase-1--reconnaissance)
3. [Phase 2 — Manual Enumeration (Netcat)](#phase-2--manual-enumeration)
4. [Phase 3 — Vulnerability Analysis](#phase-3--vulnerability-analysis)
5. [Phase 4 — Automated Enumeration (Metasploit)](#phase-4--automated-enumeration)
6. [Phase 5 — Findings & User List](#phase-5--findings--user-list)
7. [Attack Chain Summary](#attack-chain-summary)
8. [Vulnerabilities Identified](#vulnerabilities-identified)
9. [Remediation Recommendations](#remediation-recommendations)
10. [Key Lessons Learned](#key-lessons-learned)

---

## What is SMTP?

### Definition

**SMTP (Simple Mail Transfer Protocol)** is an application-layer protocol used to **send and transfer email messages** between servers over the internet. It is defined in RFC 5321 and has been the standard for email transmission since 1982.

> Think of SMTP as the **postal system of the internet** — it handles the delivery of your email from one mail server to another, just like a post office routes a letter from sender to recipient.

### How SMTP Works — Step by Step

When you hit "Send" on an email, here is what actually happens:

```
[Your Email Client (Gmail, Outlook)]
        |
        | (1) SMTP connection opened
        v
[Your Mail Server (MTA — Mail Transfer Agent)]
        |
        | (2) SMTP EHLO handshake
        | (3) MAIL FROM: sender@domain.com
        | (4) RCPT TO: recipient@otherdomain.com
        | (5) DATA: email body + headers
        |
        v
[DNS MX Record Lookup — finds recipient mail server]
        |
        v
[Recipient Mail Server (MTA)]
        |
        v
[Mail Delivery Agent (MDA) — stores in mailbox]
        |
        v
[Recipient reads via POP3 / IMAP]
```

**Step-by-step breakdown:**

1. **Connection** — Your email client (MUA) connects to your mail server (MTA) via TCP
2. **Handshake** — Client sends `EHLO` to greet the server and discover supported features
3. **Envelope** — Client declares sender (`MAIL FROM`) and recipient (`RCPT TO`)
4. **Data** — Client sends email body after `DATA` command, ending with a `.` on a new line
5. **Routing** — Server queries DNS MX records to find the recipient's mail server
6. **Relay** — Email is forwarded (relayed) between MTAs until it reaches the destination
7. **Delivery** — Final MTA hands off to the Mail Delivery Agent (MDA) which stores it in the mailbox
8. **Retrieval** — Recipient fetches the email using POP3 or IMAP (not SMTP)

> **Key point:** SMTP is **only for sending**. POP3 and IMAP are used for **receiving/reading** email. SMTP does not handle what you download into your inbox.

---

### SMTP Ports — When to Use Which

| Port | Name | Security | Used For | Status |
|---|---|---|---|---|
| **25** | Default SMTP | No encryption (plaintext) | Server-to-server relay (MTA to MTA) | Original standard — used by Metasploitable 2 |
| **587** | Submission Port | STARTTLS (upgrades to TLS) | Client to mail server (modern default) | Current recommended standard |
| **465** | SMTPS | Implicit SSL/TLS | Legacy secure mail submission | Deprecated, then restored for some providers |
| **2525** | Alternative | STARTTLS | Fallback when ISPs block other ports | Unofficial, used when 25/587 are blocked |

**In pentesting context:** Port **25 is almost always the target** — it is the server-to-server relay port, typically open on internet-facing mail servers, and often runs with weak configurations on older systems.

---

### SMTP Components — Who Does What?

| Component | Full Name | Role | Examples |
|---|---|---|---|
| **MUA** | Mail User Agent | Email client you use to compose/read mail | Gmail, Outlook, Thunderbird |
| **MTA** | Mail Transfer Agent | Sends and relays email between servers | Postfix, Sendmail, Exim |
| **MDA** | Mail Delivery Agent | Delivers email to the final mailbox | Dovecot, Procmail |
| **MSA** | Mail Submission Agent | Accepts email from MUA and passes to MTA | Often part of the MTA |

---

### SMTP Commands You Must Know

These are the raw SMTP commands used in the protocol — and abused in pentesting:

| Command | Purpose | Pentesting Relevance |
|---|---|---|
| `EHLO` / `HELO` | Greet the server, lists capabilities | Reveals supported features (`VRFY`, `EXPN`, `AUTH`) |
| `MAIL FROM:` | Declare sender address | Can be spoofed — basis for email spoofing attacks |
| `RCPT TO:` | Declare recipient address | Abused for user enumeration (see Phase 4) |
| `DATA` | Begin sending email body | Used in relay abuse and phishing tests |
| `VRFY <user>` | Verify if a user exists | **Directly leaks OS usernames** — major security issue |
| `EXPN <list>` | Expand mailing list to show members | Leaks hidden email addresses |
| `QUIT` | Close the connection | Normal termination |
| `STARTTLS` | Upgrade plaintext connection to TLS | Upgrade security mid-session |

---

### Types of SMTP Servers

| Type | Description | Example |
|---|---|---|
| **Open Relay** | Accepts and forwards mail for anyone — dangerous | Misconfigured Postfix |
| **Closed Relay** | Only relays mail from authenticated/trusted sources | Modern production servers |
| **Internal SMTP** | Used within an organization only | Microsoft Exchange (internal) |
| **Cloud SMTP** | Managed mail delivery services | SendGrid, Mailgun, Amazon SES |
| **MTA Software** | Software that implements SMTP on Linux | Postfix, Sendmail, Exim, qmail |

> On Metasploitable 2, **Postfix** is running as an **open-relay-style** MTA with VRFY enabled — the worst-case misconfiguration for a production server.

---

### Why SMTP Matters in Pentesting

SMTP is one of the most valuable early-stage recon targets because:

1. **User enumeration** — VRFY/RCPT TO reveals valid OS usernames with no authentication
2. **OS fingerprinting** — Banner reveals MTA software name, version, and OS
3. **Open relay testing** — A misconfigured SMTP can be abused to send spam or phishing emails
4. **Information disclosure** — Domain names, internal hostnames, and version numbers leaked in headers
5. **Pivot point** — Valid usernames gathered via SMTP are used to attack SSH, FTP, databases

---

## Phase 1 — Reconnaissance

### Objective
Identify whether SMTP is running, grab the service banner, and check which SMTP commands are enabled.

### Command Used

```bash
nmap -p 25 192.168.0.102 -sC -sV
```

### Output

```
PORT   STATE SERVICE VERSION
25/tcp open  smtp    Postfix smtpd
|_smtp-commands: metasploitable.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN,
                 STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN
| ssl-cert: Subject: commonName=ubuntu804-base.localdomain/organizationName=OCOSA
| Not valid before: 2010-03-17T14:07:45
|_Not valid after:  2010-04-16T14:07:45
|   SSLv2 supported
MAC Address: 08:00:27:3A:FD:C5 (Oracle VirtualBox)
Service Info: Host:  metasploitable.localdomain
```

### Findings

- **Port 25/tcp** open running **Postfix SMTP** on Ubuntu (Metasploitable 2)
- `VRFY` command is **enabled** — directly leaks whether OS usernames exist
- `SSLv2` is supported — severely outdated, vulnerable to DROWN attack
- SSL certificate expired in **April 2010** — confirms completely unpatched system
- Hostname `metasploitable.localdomain` leaked — aids target profiling

> **Key insight:** The `VRFY` command appearing in `smtp-commands` output is the green light for user enumeration. A hardened server disables VRFY and EXPN entirely.

---

## Phase 2 — Manual Enumeration

### Objective
Manually verify the VRFY vulnerability using Netcat before automating. Understanding the raw protocol is essential before using tools.

### Tool
Netcat (`nc`) — raw TCP connection to interact with SMTP directly

### Connection

```bash
nc -v 192.168.0.102 25
```

### SMTP Session

```
220 metasploitable.localdomain ESMTP Postfix (Ubuntu)
EHLO kali
250-metasploitable.localdomain
250-VRFY
250 DSN
VRFY root
252 2.0.0 root
VRFY msfadmin
252 2.0.0 msfadmin
VRFY admin
550 5.1.1 <admin>: Recipient address rejected: User unknown in local recipient table
QUIT
```

### Response Code Analysis

| Code | Meaning | Example |
|---|---|---|
| `252 2.0.0 <user>` | User **EXISTS** on the system | `VRFY root` → `252 2.0.0 root` |
| `550 5.1.1 ...` | User **DOES NOT EXIST** | `VRFY admin` → `550` |
| `502 5.5.2` | Command not recognized | Typo — must be uppercase `VRFY` |

### Notes Learned

- SMTP commands are **case-sensitive** — `verify root` fails; `VRFY root` works
- Must run `EHLO` greeting **before** using VRFY
- Linux usernames are lowercase — `VRFY ADMIN` and `VRFY admin` both return 550 here
- `252` means the server confirms the user exists but cannot guarantee delivery

---

## Phase 3 — Vulnerability Analysis

### SMTP Commands That Leak Information

| Command | Risk | Status on Target |
|---|---|---|
| `VRFY <user>` | Confirms OS username existence | **ENABLED** (vulnerable) |
| `EXPN <list>` | Expands mailing list to reveal member addresses | Not tested |
| `RCPT TO:<user>` | Delivery check — also leaks valid users | Used by Metasploit module |

### Weaknesses Identified

| # | Issue | Type | Severity |
|---|---|---|---|
| 1 | VRFY command enabled | SMTP misconfiguration | Medium |
| 2 | SSLv2 supported | Outdated protocol (DROWN) | High |
| 3 | Expired SSL certificate (2010) | Certificate management failure | Medium |
| 4 | Banner reveals OS and MTA version | Information disclosure | Low |
| 5 | Postfix on unpatched Ubuntu 8.04 | Outdated software | High |

### Why VRFY Enumeration Matters

VRFY was designed as a debugging feature — never meant to be enabled in production. On Linux systems, SMTP users map directly to OS accounts, meaning VRFY hands an attacker the complete list of valid login targets for SSH, FTP, or database attacks.

---

## Phase 4 — Automated Enumeration

### Objective
Use Metasploit smtp_enum to automatically test a large wordlist and collect all valid users.

### Tool Discovery (Self-Research Method)

```bash
msfconsole -q
msf> search smtp
msf> use auxiliary/scanner/smtp/smtp_enum
msf> info
msf> show options
ls /usr/share/metasploit-framework/data/wordlists/ | grep user
```

### Module Configuration

```bash
msf> use auxiliary/scanner/smtp/smtp_enum
msf> set RHOSTS 192.168.0.102
msf> set USER_FILE /usr/share/metasploit-framework/data/wordlists/unix_users.txt
msf> set UNIXONLY false
msf> set VERBOSE true
msf> set THREADS 1
msf> run
```

### How the Module Works

Uses the **RCPT TO method**:
```
MAIL FROM: root@metasploitable.localdomain
RCPT TO: <username>
```
- `250 OK` → user exists
- `550` → user does not exist

The module auto-reconnects when Postfix rate-limits the connection ("SMTP server annoyed...reconnecting") — demonstrating real-world resilience in automated enumeration tools.

### Result

```
[+] 192.168.0.102:25 - Users found: backup, bin, daemon, distccd, ftp, games, gnats,
    irc, libuuid, list, lp, mail, man, mysql, news, nobody, postfix, postgres,
    postmaster, proxy, service, sshd, sync, sys, syslog, user, uucp, www-data
[*] Scanned 1 of 1 hosts (100% complete)
```

---

## Phase 5 — Findings & User List

### Complete Valid User List (29 accounts)

| User | Type | Attack Priority |
|---|---|---|
| `user` | Interactive login account | Critical |
| `msfadmin` | Default admin account | Critical |
| `service` | Service account with shell | High |
| `postgres` | PostgreSQL service account | High |
| `mysql` | MySQL service account | High |
| `distccd` | Compiler daemon — known RCE (CVE-2004-2687) | High |
| `www-data` | Web server process account | Medium |
| `sshd` | SSH daemon account | Medium |
| `ftp` | FTP service account | Medium |
| `backup`, `bin`, `daemon`, `sync`, `sys` | System utility accounts | Low |
| `nobody`, `gnats`, `uucp`, `libuuid` | No-login system accounts | Informational |

### Pivot Opportunities

```
smtp_enum → 29 valid users
    |
    |---> SSH brute-force (Hydra)
    |     hydra -L users.txt -P rockyou.txt ssh://192.168.0.102
    |
    |---> PostgreSQL attack
    |     msf> use auxiliary/scanner/postgres/postgres_login
    |
    |---> distccd RCE (no password needed)
    |     msf> use exploit/unix/misc/distcc_exec
    |
    └---> MySQL attack
          msf> use auxiliary/scanner/mysql/mysql_login
```

---

## Attack Chain Summary

```
[1] Nmap scan — port 25 SMTP open
        |
        v
[2] Banner: Postfix on Ubuntu, VRFY enabled
        |
        v
[3] Manual VRFY via Netcat
        |    root, msfadmin confirmed EXIST
        |    admin confirmed DOES NOT EXIST
        v
[4] Metasploit smtp_enum (RCPT TO method)
        |    29 valid OS usernames discovered
        v
[5] User list saved → pivot to SSH / distccd / PostgreSQL
```

**Total tools used:** nmap, netcat, Metasploit (smtp_enum)  
**Credentials required:** None — pure protocol-level enumeration  
**Time to complete:** ~30 minutes  

---

## Vulnerabilities Identified

| # | Vulnerability | Type | Severity | Impact |
|---|---|---|---|---|
| 1 | VRFY command enabled | SMTP misconfiguration | Medium | OS user enumeration |
| 2 | RCPT TO user disclosure | SMTP misconfiguration | Medium | 29 accounts leaked |
| 3 | SSLv2 supported (DROWN) | Outdated protocol | High | Email traffic decryption risk |
| 4 | Expired SSL certificate | Certificate failure | Medium | No trust validation |
| 5 | Banner reveals OS version | Information disclosure | Low | OS fingerprinting |
| 6 | Postfix on Ubuntu 8.04 (EOL 2013) | Outdated software | High | Multiple unpatched CVEs |

---

## Remediation Recommendations

### Immediate Actions

1. **Disable VRFY** in `/etc/postfix/main.cf`:
   ```
   disable_vrfy_command = yes
   ```
   ```bash
   sudo systemctl restart postfix
   ```

2. **Disable SSLv2/SSLv3**, enforce TLS 1.2+:
   ```
   smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
   ```

3. **Replace the expired SSL certificate** — use Let's Encrypt for free, auto-renewing certs.

4. **Suppress the SMTP banner** to hide OS/version info:
   ```
   smtpd_banner = $myhostname ESMTP
   ```

### Medium-Term Actions

5. **Upgrade Ubuntu 8.04** — EOL since May 2013.

6. **Rate-limit SMTP connections** per IP to slow enumeration:
   ```
   smtpd_client_connection_rate_limit = 10
   ```

### Long-Term Hardening

7. **Monitor logs** for RCPT TO enumeration patterns:
   ```bash
   grep "RCPT" /var/log/mail.log | awk '{print $7}' | sort | uniq -c | sort -rn
   ```

8. **Implement SPF, DKIM, and DMARC** to prevent domain spoofing.

---

## Key Lessons Learned

### For Attackers (Pentesters)

- **SMTP enumeration needs no CVE** — it abuses legitimate protocol features left misconfigured
- **Always check smtp-commands in Nmap output** — `VRFY` in that list = user enum is possible
- **RCPT TO is more reliable than VRFY** — many servers disable VRFY but forget RCPT TO
- **Netcat before tools** — understanding the raw conversation makes you a better pentester
- **User lists are force multipliers** — 29 targeted users vs brute-forcing blind is the difference between noise and precision

### For Defenders (Blue Team)

- **SMTP is a recon goldmine if misconfigured** — one Postfix server can hand attackers your entire user directory
- **`disable_vrfy_command = yes` is a one-line fix** with zero impact on legitimate mail delivery
- **Banner suppression matters** — version leaks save attackers hours of fingerprinting
- **SSLv2 in 2026 means zero maintenance** — treat it as critically unpatched
- **Rate limiting is your first defense** — forces attackers to slow down and triggers IDS/IPS alerts

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nmap -sC -sV` | Port scanning, service detection, SMTP script scanning |
| `netcat (nc)` | Manual SMTP session for raw VRFY testing |
| Metasploit `smtp_enum` | Automated RCPT TO user enumeration with wordlist |
| `smtp-user-enum` | Alternative standalone VRFY/EXPN/RCPT enumeration tool |

---

## References

- [Postfix VRFY Hardening](http://www.postfix.org/postconf.5.html#disable_vrfy_command)
- [SMTP User Enumeration — PentestMonkey](https://pentestmonkey.net/tools/user-enumeration/smtp-user-enum)
- [Metasploit SMTP Scanner Modules — OffSec](https://www.offsec.com/metasploit-unleashed/scanner-smtp-auxiliary-modules/)
- [OWASP Testing Guide — SMTP Enumeration](https://owasp.org/www-project-web-security-testing-guide/)
- [DROWN Attack — CVE-2016-0800](https://drownattack.com/)
- [RFC 5321 — SMTP Standard](https://datatracker.ietf.org/doc/html/rfc5321)

---

*Writeup by: rio1 | Lab date: May 2026 | Platform: Metasploitable 2*
