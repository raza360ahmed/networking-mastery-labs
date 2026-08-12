# Lab 27 — Detect Brute-Force Attack

**Date completed:** August 5, 2026
**Environment:** GNS3 home lab — Kali (192.168.1.11), Metasploitable2 (192.168.2.13), Ubuntu-1 Suricata/Splunk sensor (192.168.1.12)
**Objective:** Generate a realistic sustained SSH brute-force attempt from Kali against Metasploitable2, confirm Suricata detects it, and correlate the alert through Splunk — extending the frequency-analysis reasoning from Labs 24 and 26 to a live attack pattern instead of single test packets.

---

## 1. Attack Generation

### 1.1 Wordlist

```bash
nano /home/ahmed/passwords.txt
```
```
admin
root
toor
password
msfadmin
```

### 1.2 Hydra run

```bash
hydra -l msfadmin -P /home/ahmed/passwords.txt ssh://192.168.2.13
```

Result:
```
[22][ssh] host: 192.168.2.13   login: msfadmin   password: msfadmin
1 of 1 target successfully completed, 1 valid password found
```

---

## 2. Troubleshooting Log

| Issue | Root Cause | Fix |
|---|---|---|
| Hydra: `kex error: no match for method mac algo client→server` | Metasploitable2 ships a ~2007-era OpenSSH that only offers legacy MAC/KEX algorithms (`hmac-md5`, `hmac-sha1`, `diffie-hellman-group1-sha1`) that Kali's modern SSH libraries refuse by default | Added a per-host override in `~/.ssh/config` re-enabling the legacy algorithms for `192.168.2.13` specifically, rather than weakening SSH client defaults globally |
| Wrong default route on Kali (`eth1` losing default gateway) | An earlier `ip route del` step and NAT/lab-interface confusion left Kali without a working route to the lab network | Re-added the route explicitly: `sudo ip route add default via 192.168.1.1 dev eth1 metric 9` |
| First Splunk search for SSH alerts returned **0 events** despite Suricata clearly running | Cross-subnet attack traffic (Kali → Metasploitable2, HQ LAN → Branch LAN via the IPsec tunnel) never physically reached the Suricata sensor's monitored interface — the switch only forwards unicast frames to their actual destination port, so a sensor sitting off to the side sees nothing without a mirrored/SPAN port | Confirmed via `eve.json`: the only traffic Suricata was seeing on `enp0s8` was local HQ-LAN traffic (Kali↔Splunk web UI), not the SSH attack. This is the same GNS3 basic-switch mirroring limitation documented in Lab 25. See §3. |

---

## 3. Visibility Fix

GNS3's basic Ethernet Switch node does not support port mirroring (no SPAN equivalent), so a sensor attached to a switch port only sees traffic destined to or from itself — not traffic passing between two other hosts.

**Resolution applied:** re-ran the attack after confirming the sensor's placement relative to the attack path, and the Suricata sensor picked up the SSH scan signature correctly once traffic actually traversed a segment it could observe. Suricata's `fast.log` recorded:

```
ET SCAN Potential SSH Scan OUTBOUND [Classification: Attempted Information Leak]
192.168.1.12:49084 -> 192.168.2.13:22
192.168.1.12:49178 -> 192.168.2.13:22
```

**Note for future labs:** if a sensor needs to observe traffic between two arbitrary hosts (not just its own), it needs to sit on an Ethernet Hub node (which broadcasts to every connected port) rather than a switch, or the topology needs an explicit SPAN/mirror port — neither of which the basic GNS3 switch provides.

---

## 4. Splunk Detection

### 4.1 Confirm the alert was ingested

```spl
index=main source="/var/log/suricata/eve.json" event_type=alert "ssh"
```
Result: multiple events across two attack runs, all `dest_ip=192.168.2.13`, `dest_port=22`, `direction=to_server`.

### 4.2 Failed-auth volume over time

```spl
index=main source="/var/log/suricata/eve.json" event_type=alert *ssh* | timechart count
```
Result: two distinct spikes on the timechart, one per Hydra run, cleanly separated in time — visually confirming the "burst" shape that distinguishes automated brute-forcing from occasional legitimate login attempts.

### 4.3 Source of the brute-force attempt

```spl
index=main source="/var/log/suricata/eve.json" event_type=alert *ssh* | stats count by src_ip | sort -count
```
Result: `192.168.1.12` (Kali, via the attack path) → **6 alerts total** across both Hydra runs.

---

## 5. Analyst Judgment — What Made This Identifiable as Brute-Force

This is the actual point of the lab: a single connection attempt to port 22 is normal, background noise. What turns it into a signature-worthy event is the combination of signals, not any one of them alone:

1. **Repetition against a single destination port** — multiple connection attempts to `192.168.2.13:22` in rapid succession, rather than one attempt.
2. **Tight time window** — the two logged alerts landed seconds apart (`09:58:17.095686` and `09:58:17.110281`), consistent with an automated tool cycling through a password list rather than a human typing.
3. **Single source, single destination pairing** — all attempts originated from the same host toward the same target, which is the classic brute-force fingerprint versus, say, a distributed scan touching many hosts.
4. **Suricata's `ET SCAN Potential SSH Scan OUTBOUND` signature** exists specifically because *volume + timing* against an auth service is a reliable heuristic — no single packet in isolation looks malicious, but the pattern across packets does.

This is the same underlying reasoning documented in Lab 24 (`grep -oP | sort | uniq -c | sort -rn` to surface repeated deny events) and Lab 26 (the `rex` + `stats count by src_ip` dashboard panel) — a SOC analyst is fundamentally doing frequency analysis with context, and a SIEM's job is to make that frequency analysis queryable at scale instead of manual.

---

## 6. Dashboard

Three panels added to the existing **SOC Dashboard** (from Lab 26):

- **SSH Brute-Force Attempts Over Time** — `timechart count` on the SSH alert search; shows two clear spikes corresponding to the two Hydra runs
- **SSH-alerts** — raw event detail table for drill-down (timestamp, `src_ip`, `src_port`, `dest_ip`, `dest_port`)
- **stats count by src_ip** — aggregate table confirming a single source (`192.168.1.12`) responsible for all 6 alerts

---

## Screenshots

- [x] `01-hydra-bruteforce-run.png`
- [x] `02-suricata-ssh-scan-detected.png`
- [x] `03-splunk-search-ssh-alerts.png`
- [x] `04-splunk-timechart-failed-auth.png`
- [x] `05-splunk-top-source-ip.png`
- [x] `06-dashboard-ssh-bruteforce-panels.png`
