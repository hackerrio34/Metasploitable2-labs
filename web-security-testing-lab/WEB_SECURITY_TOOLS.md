# 🛠️ Web Security Testing — Tools Reference Guide

> A complete reference guide for web security scanning tools, their purpose, usage, and when to choose them during penetration testing.

⚠️ **Disclaimer:** All tools listed here must only be used on systems you own or have explicit written permission to test.

---

## 📚 Table of Contents

1. [Penetration Testing Phases](#1-penetration-testing-phases)
2. [Reconnaissance Tools](#2-reconnaissance-tools)
3. [Scanning Tools](#3-scanning-tools)
4. [Enumeration Tools](#4-enumeration-tools)
5. [Vulnerability Analysis Tools](#5-vulnerability-analysis-tools)
6. [Exploitation Tools](#6-exploitation-tools)
7. [Password Cracking Tools](#7-password-cracking-tools)
8. [Traffic Interception Tools](#8-traffic-interception-tools)
9. [Quick Reference — What to Use When](#9-quick-reference--what-to-use-when)

---

## 1. Penetration Testing Phases

```
┌──────────────────────────────────────────────────────┐
│              WEB PENTEST METHODOLOGY                 │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Phase 1 → RECONNAISSANCE                            │
│            Passive & active info gathering           │
│                                                      │
│  Phase 2 → SCANNING                                  │
│            Find open ports, services, vulns          │
│                                                      │
│  Phase 3 → ENUMERATION                               │
│            Extract detailed info from services       │
│                                                      │
│  Phase 4 → VULNERABILITY ANALYSIS                    │
│            Identify exploitable weaknesses           │
│                                                      │
│  Phase 5 → EXPLOITATION                              │
│            Safely demonstrate vulnerabilities        │
│                                                      │
│  Phase 6 → REPORTING                                 │
│            Document findings & recommendations       │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## 2. Reconnaissance Tools

Reconnaissance is about gathering information before touching the target.

---

### 🔹 Nmap
**Purpose:** Network port scanner and service detector

**Install:**
```bash
sudo apt install nmap
```

**Basic Usage:**
```bash
# Quick scan
nmap 192.168.0.106

# Service version detection
nmap -sV 192.168.0.106

# Full scan with scripts
nmap -sV -sC -p- 192.168.0.106

# Specific ports only
nmap -p 80,443,8080,3306 192.168.0.106
```

**What it finds:**
- Open/closed ports
- Running services
- Service versions
- Operating system
- Basic vulnerabilities via scripts

**Best for:** First step in any pentest — always run nmap first.

---

### 🔹 Curl
**Purpose:** HTTP request inspection tool

**Install:** Pre-installed on Kali Linux

**Basic Usage:**
```bash
# Get HTTP headers only
curl -I http://target.com

# Full verbose request
curl -v http://target.com

# Send POST request
curl -X POST -d "username=admin&password=test" http://target.com/login

# Save cookies
curl -c cookies.txt http://target.com

# Use saved cookies
curl -b cookies.txt http://target.com
```

**What it finds:**
- Server version from headers
- PHP/framework version
- Missing security headers
- Cookie configuration
- HTTP methods allowed

**Best for:** Quick manual HTTP inspection and header analysis.

---

### 🔹 Whatweb
**Purpose:** Web technology fingerprinting

**Install:**
```bash
sudo apt install whatweb
```

**Basic Usage:**
```bash
whatweb http://192.168.0.106
whatweb -a 3 http://192.168.0.106  # aggressive mode
```

**What it finds:**
- CMS (WordPress, Joomla etc)
- Framework versions
- Server technologies
- JavaScript libraries

**Best for:** Quickly identifying what technology stack a website uses.

---

### 🔹 Whois
**Purpose:** Domain registration information

**Install:** Pre-installed on Kali Linux

**Basic Usage:**
```bash
whois target.com
```

**What it finds:**
- Domain owner information
- Registration dates
- Name servers
- Contact emails

**Best for:** External/public targets — gathering organizational info.

---

## 3. Scanning Tools

Scanning actively probes the target to find vulnerabilities.

---

### 🔹 Nikto
**Purpose:** Web server vulnerability scanner

**Install:**
```bash
sudo apt install nikto
```

**Basic Usage:**
```bash
# Basic scan
nikto -h http://192.168.0.106

# Scan specific port
nikto -h http://192.168.0.106 -p 8080

# Save output
nikto -h http://192.168.0.106 -o output.txt

# Scan with SSL
nikto -h https://192.168.0.106 -ssl
```

**What it finds:**
- Outdated server software
- Dangerous HTTP methods (TRACE, PUT)
- Exposed sensitive files (phpinfo.php)
- Missing security headers
- Default credentials
- Known CVEs

**Best for:** Quick automated web vulnerability scan — always run after nmap.

---

### 🔹 Dirb
**Purpose:** Directory and file brute forcer

**Install:**
```bash
sudo apt install dirb
```

**Basic Usage:**
```bash
# Basic scan
dirb http://192.168.0.106

# Custom wordlist
dirb http://192.168.0.106 /usr/share/wordlists/dirb/big.txt

# Scan with extension
dirb http://192.168.0.106 -X .php,.txt,.html

# Ignore specific response code
dirb http://192.168.0.106 -N 404
```

**What it finds:**
- Hidden directories
- Backup files
- Admin panels
- Configuration files
- Sensitive files

**Best for:** Finding hidden paths on web servers.

---

### 🔹 Gobuster
**Purpose:** Faster directory and DNS brute forcer

**Install:**
```bash
sudo apt install gobuster
```

**Basic Usage:**
```bash
# Directory mode
gobuster dir -u http://192.168.0.106 -w /usr/share/wordlists/dirb/common.txt

# With file extensions
gobuster dir -u http://192.168.0.106 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html

# DNS mode
gobuster dns -d target.com -w /usr/share/wordlists/dirb/common.txt

# With threads (faster)
gobuster dir -u http://192.168.0.106 -w wordlist.txt -t 50
```

**What it finds:** Same as dirb but much faster

**Best for:** When you need faster directory brute forcing than dirb.

| Feature | Dirb | Gobuster |
|---------|------|----------|
| Speed | Slower | Faster |
| Ease of use | Simpler | More options |
| DNS brute force | No | Yes |
| Best for | Beginners | Advanced users |

---

### 🔹 SSLScan
**Purpose:** SSL/TLS configuration scanner

**Install:**
```bash
sudo apt install sslscan
```

**Basic Usage:**
```bash
sslscan https://target.com
```

**What it finds:**
- SSL/TLS version support
- Weak cipher suites
- Certificate details
- HEARTBLEED vulnerability
- POODLE vulnerability

**Best for:** Testing HTTPS configuration on websites.

---

## 4. Enumeration Tools

Enumeration extracts detailed information from discovered services.

---

### 🔹 MySQL Client
**Purpose:** Direct MySQL database access

**Install:** Pre-installed on Kali Linux

**Basic Usage:**
```bash
# Connect to remote MySQL
mysql -h 192.168.0.106 -u root --ssl=0

# Connect with password
mysql -h 192.168.0.106 -u root -p

# Run query directly
mysql -h 192.168.0.106 -u root -e "show databases;"
```

**Useful SQL Commands:**
```sql
-- List all databases
show databases;

-- Select database
use dvwa;

-- List tables
show tables;

-- View users
select user, password from mysql.user;

-- View table structure
describe users;
```

**Best for:** When MySQL port is open and accessible.

---

### 🔹 Hash-Identifier
**Purpose:** Identify password hash types

**Install:**
```bash
sudo apt install hash-identifier
```

**Basic Usage:**
```bash
hash-identifier <hash_value>
hash-identifier 5f4dcc3b5aa765d61d8327deb882cf99
```

**Identifies:**
- MD5
- SHA1, SHA256, SHA512
- NTLM
- bcrypt
- And many more

**Best for:** First step before password cracking — must know hash type.

---

### 🔹 Enum4linux
**Purpose:** Linux/Samba enumeration

**Install:**
```bash
sudo apt install enum4linux
```

**Basic Usage:**
```bash
enum4linux -a 192.168.0.106
```

**What it finds:**
- User accounts
- Shared folders
- Password policies
- OS information

**Best for:** When SMB/Samba ports (139, 445) are open.

---

## 5. Vulnerability Analysis Tools

---

### 🔹 OpenVAS
**Purpose:** Full vulnerability assessment scanner

**Install:**
```bash
sudo apt install openvas
sudo gvm-setup
sudo gvm-start
```

**What it finds:**
- Known CVEs
- Misconfigurations
- Missing patches
- Weak credentials

**Best for:** Comprehensive vulnerability assessment of entire networks.

---

### 🔹 Wpscan
**Purpose:** WordPress vulnerability scanner

**Install:**
```bash
sudo apt install wpscan
```

**Basic Usage:**
```bash
# Basic scan
wpscan --url http://target.com

# Enumerate users
wpscan --url http://target.com --enumerate u

# Enumerate plugins
wpscan --url http://target.com --enumerate p
```

**What it finds:**
- WordPress version
- Vulnerable plugins
- Vulnerable themes
- User accounts

**Best for:** Any target running WordPress CMS.

---

## 6. Exploitation Tools

---

### 🔹 SQLMap
**Purpose:** Automated SQL injection testing

**Install:**
```bash
sudo apt install sqlmap
```

**Basic Usage:**
```bash
# Basic test
sqlmap -u "http://192.168.0.106/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit"

# With cookies (authenticated)
sqlmap -u "http://target.com/page?id=1" --cookie="PHPSESSID=abc123"

# Dump database
sqlmap -u "http://target.com/page?id=1" --dbs

# Dump specific table
sqlmap -u "http://target.com/page?id=1" -D dvwa -T users --dump
```

**What it does:**
- Detects SQL injection points
- Extracts database names
- Dumps tables and data
- Supports many DB types (MySQL, MSSQL, Oracle)

**Best for:** Testing and demonstrating SQL injection vulnerabilities.

---

### 🔹 Metasploit Framework
**Purpose:** Exploitation framework

**Install:** Pre-installed on Kali Linux

**Basic Usage:**
```bash
# Start metasploit
msfconsole

# Search for module
search apache 2.2

# Use a module
use exploit/multi/http/php_cgi_arg_injection

# Show options
show options

# Set target
set RHOSTS 192.168.0.106

# Run
run
```

**Best for:** Testing known CVEs and exploits against services.

---

### 🔹 XSStrike
**Purpose:** Advanced XSS scanner

**Install:**
```bash
git clone https://github.com/s0md3v/XSStrike
cd XSStrike
pip3 install -r requirements.txt
```

**Basic Usage:**
```bash
python3 xsstrike.py -u "http://target.com/page?q=test"
```

**What it finds:**
- Reflected XSS
- Stored XSS
- DOM XSS
- Bypasses for filters

**Best for:** Thorough XSS testing beyond manual injection.

---

## 7. Password Cracking Tools

---

### 🔹 John the Ripper
**Purpose:** Password hash cracker

**Install:**
```bash
sudo apt install john
```

**Basic Usage:**
```bash
# Crack MD5 hashes
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt

# Crack SHA1 hashes
john --format=raw-sha1 --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt

# Show cracked passwords
john --show hashes.txt

# Identify hash format automatically
john --list=formats | grep md5
```

**Best for:** Cracking password hashes extracted from databases.

---

### 🔹 Hashcat
**Purpose:** GPU accelerated password cracker (faster than John)

**Install:**
```bash
sudo apt install hashcat
```

**Basic Usage:**
```bash
# Crack MD5 (mode 0)
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt

# Crack SHA1 (mode 100)
hashcat -m 100 hashes.txt /usr/share/wordlists/rockyou.txt

# Crack bcrypt (mode 3200)
hashcat -m 3200 hashes.txt /usr/share/wordlists/rockyou.txt
```

**Common Hash Modes:**

| Mode | Hash Type |
|------|-----------|
| 0 | MD5 |
| 100 | SHA1 |
| 1400 | SHA256 |
| 1800 | SHA512 |
| 3200 | bcrypt |
| 1000 | NTLM |

**Best for:** Large hash lists — much faster than John the Ripper using GPU.

| Feature | John the Ripper | Hashcat |
|---------|----------------|---------|
| Speed | Moderate | Very fast (GPU) |
| Ease of use | Easier | More complex |
| Auto detection | Yes | Manual mode |
| Best for | Beginners | Large hash lists |

---

### 🔹 Hydra
**Purpose:** Online brute force tool

**Install:**
```bash
sudo apt install hydra
```

**Basic Usage:**
```bash
# HTTP POST form brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.0.106 http-post-form "/login:username=^USER^&password=^PASS^:Invalid"

# SSH brute force
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.0.106

# MySQL brute force
hydra -l root -P /usr/share/wordlists/rockyou.txt mysql://192.168.0.106
```

**Best for:** Brute forcing login forms and network services.

---

## 8. Traffic Interception Tools

---

### 🔹 Burp Suite
**Purpose:** Web application proxy and testing platform

**Install:** Pre-installed on Kali Linux

**Setup:**
1. Open Burp Suite
2. Go to Proxy → Options → set port 8080
3. Configure browser proxy to 127.0.0.1:8080
4. Enable Intercept

**Key Features:**

| Feature | Purpose |
|---------|---------|
| Proxy | Intercept HTTP/HTTPS traffic |
| Repeater | Modify and resend requests |
| Intruder | Automated attack tool |
| Scanner | Vulnerability scanner (Pro) |
| Decoder | Encode/decode data |

**Basic Usage:**
```
1. Intercept login request
2. See plaintext credentials
3. Modify request parameters
4. Forward modified request
5. Observe response
```

**Best for:** Manual web application testing — the most important web security tool.

---

### 🔹 Wireshark
**Purpose:** Network traffic analyzer

**Install:**
```bash
sudo apt install wireshark
```

**Basic Usage:**
```bash
sudo wireshark &
```

**Useful Filters:**
```
http                          # All HTTP traffic
http.request.method == "POST" # POST requests only
ip.addr == 192.168.0.106      # Traffic to/from target
http contains "password"      # Requests containing password
```

**What it captures:**
- Plaintext HTTP credentials
- Session cookies
- All unencrypted network data

**Best for:** Demonstrating why HTTP is dangerous — captures plaintext passwords.

---

## 9. Quick Reference — What to Use When

### By Vulnerability Type

| Vulnerability | Best Tool | Command |
|--------------|-----------|---------|
| SQL Injection | SQLMap | `sqlmap -u "url?id=1"` |
| XSS | XSStrike | `python3 xsstrike.py -u url` |
| Directory discovery | Gobuster | `gobuster dir -u url -w wordlist` |
| Web vulnerabilities | Nikto | `nikto -h url` |
| Password hashes (offline) | Hashcat | `hashcat -m 0 hashes.txt wordlist` |
| Login brute force | Hydra | `hydra -l user -P wordlist target service` |
| HTTP traffic | Burp Suite | Configure browser proxy |
| Network traffic | Wireshark | Filter: `http.request.method==POST` |
| SSL/TLS issues | SSLScan | `sslscan https://target` |
| WordPress | WPScan | `wpscan --url target` |

---

### By Phase

| Phase | Tools to Use |
|-------|-------------|
| Reconnaissance | Nmap, Curl, Whatweb, Whois |
| Scanning | Nikto, Dirb, Gobuster, SSLScan |
| Enumeration | MySQL client, Enum4linux, Hash-identifier |
| Vulnerability Analysis | OpenVAS, WPScan, Nikto |
| Exploitation | SQLMap, Metasploit, XSStrike, Hydra |
| Post Exploitation | John, Hashcat, Burp Suite |

---

### By Finding

| What You Found | What to Do Next | Tool |
|---------------|-----------------|------|
| Open port 80 | Scan for web vulns | Nikto |
| Open port 80 | Find hidden dirs | Gobuster |
| Open port 3306 | Connect to MySQL | MySQL client |
| Login page | Brute force | Hydra |
| Password hash | Identify type | Hash-identifier |
| MD5 hash | Crack password | John / Hashcat |
| Input field | Test SQL injection | SQLMap |
| Input field | Test XSS | XSStrike |
| File upload | Test file restrictions | Manual / Burp Suite |
| HTTP traffic | Intercept requests | Burp Suite |
| WordPress site | Full WP scan | WPScan |
| SSL/HTTPS | Test TLS config | SSLScan |

---

## 📖 Key Concepts

### Why HTTP is Dangerous
```
HTTP  → Data travels in PLAINTEXT → Anyone can read it
HTTPS → Data travels ENCRYPTED   → Nobody can read it
```

### Password Hash Strength
```
MD5     → Broken — cracked in seconds
SHA1    → Weak  — avoid
SHA256  → Better but needs salting
bcrypt  → Strong — recommended
argon2  → Strongest — best choice
```

### SQL Injection Prevention
```python
# WRONG — vulnerable
query = "SELECT * FROM users WHERE id=" + user_input

# RIGHT — safe
query = "SELECT * FROM users WHERE id=?"
cursor.execute(query, (user_input,))
```

### XSS Prevention
```python
# WRONG — vulnerable
output = "<h1>" + user_input + "</h1>"

# RIGHT — safe
output = "<h1>" + html.escape(user_input) + "</h1>"
```

---

*This guide is for educational purposes only. Always obtain written permission before testing any system.*
