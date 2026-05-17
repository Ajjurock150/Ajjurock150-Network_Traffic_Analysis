# 🟡 Network Traffic Analysis — Wireshark Intrusion Detection

![Wireshark](https://img.shields.io/badge/Tool-Wireshark-blue?style=flat-square&logo=wireshark)
![Platform](https://img.shields.io/badge/Platform-Kali%20Linux-purple?style=flat-square&logo=kalilinux)
![Type](https://img.shields.io/badge/Type-Network%20Forensics-yellow?style=flat-square)
![Status](https://img.shields.io/badge/Status-Completed-green?style=flat-square)

---

## 📌 Project Overview

This project focuses on **network traffic analysis and intrusion detection** using Wireshark in a controlled lab environment. I captured and analysed both live traffic and pre-recorded `.pcap` files to identify malicious activity — port scans, ARP spoofing, cleartext credential leakage, and protocol abuse — simulating the traffic analysis workflow of a SOC analyst or network defender.

---

## 🧰 Tools & Technologies

| Tool | Purpose |
|------|---------|
| Wireshark | Primary packet capture and analysis tool |
| Nmap | Port scanning (generating detectable traffic) |
| Kali Linux | Attacker machine for simulating malicious traffic |
| Ettercap | ARP spoofing simulation for MITM detection practice |
| tcpdump | CLI-based packet capture for automation |

---

## ⚙️ Lab Environment

```
Attacker Machine  : Kali Linux (192.168.1.10)
Victim Machine    : Windows 10 (192.168.1.20)
FTP Server        : Metasploitable 2 (192.168.1.30)
Network           : Host-only / NAT (VirtualBox)
Capture Interface : eth0 on Kali
```

---

## 🎯 Attack Story — What Did the Traffic Reveal?

The `.pcap` capture was taken during a simulated insider threat / external intrusion scenario. Wireshark analysis revealed a **multi-stage attack** hidden within normal-looking traffic:

```
Stage 1 — Reconnaissance  : Nmap SYN scan targeting the subnet
Stage 2 — Credential Theft: FTP login captured in cleartext
Stage 3 — MITM Setup      : ARP spoofing — attacker poisoned gateway ARP table
Stage 4 — Data Interception: HTTP POST with login form captured mid-stream
Stage 5 — C2 Beacon       : Periodic outbound connections to external IP on non-standard port
```

---

## 🔍 Detection Logic — Wireshark Filters & Analysis

### 1. Detecting Port Scan (Nmap SYN Scan)
**Indicator:** Large number of TCP SYN packets to sequential ports, no SYN-ACK responses (RST replies indicate closed ports)

```wireshark
tcp.flags.syn == 1 and tcp.flags.ack == 0
```

**What I found:**
- 1,000+ SYN packets from `192.168.1.10` to `.20` in under 2 seconds
- Ports 21, 22, 23, 80, 443, 3306, 8080 probed in sequence
- Pattern consistent with `nmap -sS` (stealth SYN scan)

---

### 2. Cleartext Credential Detection — FTP
**Indicator:** FTP transmits username and password as plaintext over port 21

```wireshark
ftp.request.command == "USER" or ftp.request.command == "PASS"
```

**What I found:**
- FTP session from `192.168.1.10` to `192.168.1.30`
- Credentials visible in plaintext: `USER msfadmin` / `PASS msfadmin`
- Full FTP session reconstructed via `Follow TCP Stream`

---

### 3. ARP Spoofing Detection
**Indicator:** Multiple ARP replies from the same MAC address claiming different IPs (gateway IP reassigned to attacker MAC)

```wireshark
arp.opcode == 2
```

**Then checked:** `Edit → Find Packet` for duplicate IP-to-MAC mappings

**What I found:**
- Gateway IP `192.168.1.1` appeared in ARP replies from two different MACs
- Attacker MAC `aa:bb:cc:dd:ee:ff` was claiming to be the gateway
- Classic ARP cache poisoning — all victim traffic was routing through attacker

---

### 4. HTTP POST — Cleartext Login Form Capture
**Indicator:** HTTP POST requests containing form data (username/password fields) — visible in plaintext since HTTP is unencrypted

```wireshark
http.request.method == "POST"
```

**Follow TCP Stream** revealed:
```
POST /login HTTP/1.1
Host: 192.168.1.20
Content-Type: application/x-www-form-urlencoded

username=admin&password=admin123
```

---

### 5. C2 Beacon Detection — Periodic Outbound Connections
**Indicator:** Regular, timed outbound connections to external IP on uncommon port — beacon interval pattern

```wireshark
ip.dst == 45.33.32.156 and tcp.port == 4444
```

**What I found:**
- Connection to `45.33.32.156:4444` every ~60 seconds
- Small, consistent packet sizes (heartbeat pattern)
- Port 4444 is a known Metasploit default listener port
- Flagged as potential reverse shell C2 channel

---

### 6. DNS Exfiltration Check
**Indicator:** Unusually long DNS query names or high volume of DNS requests to a single domain

```wireshark
dns and frame.len > 100
```

```wireshark
dns.qry.name contains ".xyz" or dns.qry.name contains ".top"
```

---

## 📋 Findings Summary

| # | Threat | Protocol | Source IP | Destination | Severity |
|---|--------|----------|-----------|-------------|----------|
| 1 | Port scan (SYN) | TCP | 192.168.1.10 | 192.168.1.20 | 🟡 Medium |
| 2 | Cleartext FTP credentials | FTP | 192.168.1.10 | 192.168.1.30 | 🔴 Critical |
| 3 | ARP spoofing / MITM | ARP | 192.168.1.10 | Broadcast | 🔴 Critical |
| 4 | HTTP cleartext credentials | HTTP | 192.168.1.10 | 192.168.1.20 | 🔴 Critical |
| 5 | C2 beacon (port 4444) | TCP | 192.168.1.20 | 45.33.32.156 | 🔴 Critical |

---

## 🧯 Remediation Recommendations

| Finding | Recommended Action |
|---------|--------------------|
| Port scan detected | Enable IDS rules (Snort/Suricata) for SYN scan signatures; rate-limit inbound SYN packets |
| FTP cleartext credentials | Disable FTP; replace with SFTP or FTPS; enforce encrypted transfer protocols |
| ARP spoofing / MITM | Enable Dynamic ARP Inspection (DAI) on managed switches; use static ARP entries for critical hosts |
| HTTP cleartext login | Enforce HTTPS (TLS 1.2+) across all web services; implement HSTS headers |
| C2 beacon (port 4444) | Block outbound connections on non-standard ports; deploy EDR to detect reverse shells; isolate host |

---

## 🧠 Key Wireshark Skills Demonstrated

| Skill | Details |
|-------|---------|
| Display Filters | Protocol, port, flag, and IP-based filters |
| Follow TCP Stream | Reconstructing sessions to read payload |
| IO Graphs | Visualising traffic spikes during scan |
| ARP Analysis | Detecting duplicate IP-MAC mappings |
| Export Objects | Extracting files transferred over HTTP/FTP |
| Statistics → Conversations | Identifying top talkers and connection pairs |

---

## 🔗 References
- [MITRE ATT&CK — Network Sniffing (T1040)](https://attack.mitre.org/techniques/T1040/)
- [MITRE ATT&CK — ARP Cache Poisoning (T1557.002)](https://attack.mitre.org/techniques/T1557/002/)
- [Wireshark Display Filter Reference](https://www.wireshark.org/docs/dfref/)
