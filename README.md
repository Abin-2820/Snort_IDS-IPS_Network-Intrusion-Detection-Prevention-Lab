# Snort IDS / IPS — Network Intrusion Detection & Prevention Lab

Hands-on project deploying Snort in two operating modes — passive Intrusion Detection (IDS) and active Intrusion Prevention (IPS) — to detect reconnaissance/authentication activity and block a simulated Denial-of-Service attack in real time.

## Scenario

A Kali Linux attacker machine was used to simulate common network attack behavior (ICMP probing, SSH/FTP connection attempts, and a SYN flood DoS) against an Ubuntu endpoint running Snort. The goal was to demonstrate both passive alerting (IDS) and active inline blocking (IPS) using custom-written Snort rules.

## Tools Used

- **Snort 2 (classic)** — signature-based IDS/IPS engine
- **ping (ICMP)** — reconnaissance simulation
- **ssh / ftp** — connection attempt simulation
- **hping3** — SYN flood DoS traffic generation

## Skills Demonstrated

- Custom Snort rule writing (`alert` and `drop` actions)
- Threshold-based rule logic to reduce false positives
- IDS vs. IPS deployment modes and operational trade-offs
- Inline traffic blocking and DoS mitigation
- Attack simulation (ping, SSH, FTP, hping3)
- MITRE ATT&CK technique mapping for network-based detections

---

## Part 1 — Snort as IDS (Detection Mode)

In this mode, Snort passively monitors traffic and generates alerts for matched signatures without interfering with the traffic itself.

### Rules Used (`local.rules`)
### 1.1 ICMP Ping Detection

**Attack command (Kali):**

```bash
ping 192.168.90.6
```

**Rule triggered:** `sid:1000001`

**Result:** Snort generated an alert for each ICMP echo request received from the Kali attacker, confirming basic reconnaissance/host-discovery activity is visible to the IDS.

![ICMP alert triggered](screenshots/01_icmp_alert.png)

---

### 1.2 SSH Connection Attempt Detection

**Attack command (Kali):**

```bash
ssh root@192.168.90.6
```

**Rule triggered:** `sid:1000002`

**Result:** Snort detected and alerted on the inbound TCP connection attempt to port 22, regardless of whether authentication succeeded — confirming visibility into SSH access attempts at the network layer.

![SSH alert triggered](screenshots/02_ssh_alert.png)

---

### 1.3 FTP Authentication Attempt Detection

**Attack command (Kali):**

```bash
ftp 192.168.90.6
```

**Rule triggered:** `sid:1000003`

**Result:** Snort detected and alerted on the inbound connection to port 21, flagging the FTP authentication attempt from the attacking host.

![FTP alert triggered](screenshots/03_ftp_alert.png)

---

## Part 2 — Snort as IPS (Prevention Mode)

Unlike IDS mode, IPS mode runs Snort inline using a DAQ configuration that places it directly in the traffic path, allowing it to actively **drop** packets that match a rule — not just alert on them.

### 2.1 SYN Flood DoS Prevention

**Rule used:**
**Attack command (Kali):**

```bash
hping3 -S -p 80 --flood 192.168.90.3
```

**Rule breakdown:**

- `drop` — instructs Snort's inline engine to actively discard matching packets rather than just log an alert
- `flags:S` — matches on TCP SYN packets specifically, the hallmark of a SYN flood
- `threshold: type both, track by_src, count 100, seconds 10` — triggers the rule only once a single source IP sends 100+ SYN packets within a 10-second window, filtering out normal connection attempts while catching flood-level traffic

**Result:** Once the SYN packet rate from the attacking host exceeded the threshold, Snort actively dropped the offending traffic inline. The targeted service on `192.168.90.3:80` remained reachable and responsive throughout the attack, confirming the IPS rule successfully **prevented** the DoS condition rather than merely detecting it.

![Snort IPS drop log showing SYN Flood Blocked alerts](screenshots/04_ips_drop_log.png)

![Target service availability confirmed during attack](screenshots/05_service_availability.png)

---

## Findings Summary

| # | Scenario | Mode | Rule (sid) | Result |
|---|----------|------|------------|--------|
| 1 | ICMP ping sweep | IDS | 1000001 | Alert generated for each ICMP echo request |
| 2 | SSH connection attempt | IDS | 1000002 | Alert generated on inbound TCP/22 connection |
| 3 | FTP authentication attempt | IDS | 1000003 | Alert generated on inbound TCP/21 connection |
| 4 | SYN flood DoS (hping3) | IPS | 1000001 | Traffic actively dropped; target service stayed available |

---
## Recommendations

1. Tune IDS thresholds in production to reduce alert fatigue from legitimate SSH/FTP administrative traffic (e.g. allow-list known admin source IPs).
2. Extend IPS rules to cover additional DoS vectors (UDP flood, ICMP flood) using similar threshold-based drop rules.
3. Pair Snort alerts with a SIEM (e.g. Wazuh, Splunk) for centralized correlation, long-term retention, and dashboarding.
4. Regularly update and version-control custom rule sets (`local.rules`) alongside official Snort community/registered rule feeds.
5. Test IPS rules in a staging environment before production deployment to avoid inadvertently dropping legitimate traffic (false positives).
6. 
---

## About

Documented as part of a hands-on blue team / SOC skills-building project, alongside related work in network traffic forensics (Wireshark), SIEM detection engineering (Wazuh), and IDS log forwarding (Snort + Splunk).

**Author:** Abin Watson — Penetration Tester (eJPT) building defensive security skills.
