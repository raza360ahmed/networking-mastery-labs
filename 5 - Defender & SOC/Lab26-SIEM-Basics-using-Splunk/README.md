# Lab 26 — SIEM Basics using Splunk

**Date completed:** August 4, 2026
**Environment:** GNS3 home lab — Ubuntu-1 sensor VM (192.168.1.12), Kali Linux (192.168.1.11)
**Objective:** Stand up Splunk Enterprise on the existing IDS/log sensor VM, ingest the router firewall logs and Suricata alerts collected in Lab 25, and prove out centralized search and correlation across both sources.

---

## 1. Overview

This lab is the payoff of Lab 25. The sensor VM was already collecting two independent data streams:

- **R1 firewall logs** via rsyslog (ACL 150 deny events, forwarded from the Cisco router)
- **Suricata IDS events** (`eve.json` structured JSON, `fast.log` plain text)

Up to this point, correlating those two sources meant manually `grep`-ing two separate files and eyeballing timestamps. This lab installs Splunk Enterprise on top of that same sensor VM so both sources can be indexed, searched, and correlated through one interface using SPL (Search Processing Language) instead of chained Unix pipes.

---

## 2. Installation

### 2.1 Download

Splunk no longer serves a stable `/releases/latest/` redirect for direct `wget` downloads — that path now returns a 404. The working approach is to pin the exact version and build hash:

```bash
wget -O splunk.deb "https://download.splunk.com/products/splunk/releases/10.4.0/linux/splunk-10.4.0-f798d4d49089-linux-amd64.deb"
```

Installed version: **Splunk Enterprise 10.4.0**

### 2.2 Install

```bash
sudo dpkg -i splunk.deb
```

### 2.3 First-run setup

```bash
sudo /opt/splunk/bin/splunk start --accept-license --no-prompt --answer-yes --run-as-root
```

- Admin account created interactively (username: `ahmed`)
- `--run-as-root` was required because Splunk 10.x refuses to start as root without it when invoked via `sudo`
- Enabled boot-start so Splunk survives VM reboots:

```bash
sudo /opt/splunk/bin/splunk enable boot-start --run-as-root
```

### 2.4 Confirm running state

```bash
sudo /opt/splunk/bin/splunk status
# splunkd is running (PID: xxxx)
```

Web interface: **http://192.168.1.12:8000** (default Splunk port is **8000**, not 8080 — the lab plan's original port assumption was wrong and would have collided with an unrelated netcat listener anyway).

---

## 3. Troubleshooting Log

Documenting these because none of them were exotic — they're exactly the kind of friction a junior analyst hits in the first real deployment, and it's worth having the fixes on record.

| Issue | Root Cause | Fix |
|---|---|---|
| `wget: invalid option -- '0'` | Typed `-0` (dash-zero) instead of `-O` (dash-capital-O) | Used `--output-document=` to avoid the ambiguous short flag entirely |
| `404 Not Found` on `/releases/latest/` | Splunk deprecated the generic "latest" redirect for direct download | Pinned exact version + build hash: `10.4.0-f798d4d49089` |
| `lzma error: compressed data is corrupt` on `dpkg -i` | A `wget -c` (resume) was performed on top of a `.deb` that was already truncated by a mid-download VM crash — size matched post-resume but content didn't | Deleted the file and re-downloaded fresh (no `-c`) rather than resuming |
| Kernel panic during download (`do_syscall_64`, OOM-adjacent trace) | VM under memory/disk pressure while downloading a 1.2GB file on an 11GB disk | Force-restarted the VM; underlying disk capacity issue addressed properly later (see below) |
| `splunk: command not found` | Ran the start command before installation had actually completed | Confirmed `dpkg -i` finished with `complete` before attempting to start |
| Start command exits instantly with a deprecation notice, no daemon starts | Running Splunk as root via `sudo` is deprecated in 10.x and blocked without an explicit override | Added `--run-as-root` flag |
| `Search not executed: minimum free disk space (5000MB) reached` | Splunk refuses to index/search below a 5GB free-space floor by default | Root cause was the disk itself being full (100% used, `df` confirmed) — freeing files was a stopgap; real fix was resizing the disk (§4) |
| `systemd-journald: Failed to create new system journal: No space left on device` | Same root disk-full condition, surfaced at the OS level | `journalctl --vacuum-size=50M` bought ~112MB temporarily but wasn't sufficient long-term |

**Lesson:** disk sizing should be planned before installing anything Splunk-sized. An 11–12GB root disk is workable for the networking/IDS labs but is genuinely too small once Splunk, its indexes, and Suricata's `eve.json` growth are added on top.

---

## 4. Disk Resize (VirtualBox + LVM)

The sensor VM's disk was resized from ~11GB to ~29GB to give Splunk sustainable headroom.

**On the Windows host** (VM powered off, PowerShell from the VirtualBox install directory):

```powershell
.\VBoxManage.exe modifyhd "D:\Ubuntu\Ubuntu\Ubuntu.vdi" --resize 30000
```

**Inside the Ubuntu VM** (LVM-backed root, so no reboot into a live CD was needed):

```bash
sudo apt install -y cloud-guest-utils
sudo growpart /dev/sda 3
sudo pvresize /dev/sda3
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
df -h /
```

Result: `/dev/mapper/ubuntu--vg-ubuntu--lv` grew from 12G to the full resized disk, clearing the 98%+ usage warning and unblocking Splunk searches.

---

## 5. Data Ingestion (Add Data → Monitor → Files & Directories)

Three sources monitored, all indexed into `main`:

| Source | Sourcetype | Notes |
|---|---|---|
| `/var/log/syslog` | `syslog` | Auto-detected correctly |
| `/var/log/suricata/eve.json` | `_json` | Structured JSON — Splunk auto-parses fields like `event_type`, `src_ip`, `dest_ip`, `app_proto` |
| `/var/log/suricata/fast.log` | `syslog` (fallback) | Plain text, not as cleanly field-extracted as `eve.json` |

`eve.json` is the more valuable source of the two Suricata logs specifically because its structured format lets Splunk index fields natively, rather than requiring regex extraction at search time the way `fast.log` does.

---

## 6. Searches

### 6.1 Confirm ingestion

```spl
index=main source="/var/log/syslog"
```
Result: 3,403 events indexed successfully once disk space was resolved.

### 6.2 R1 firewall deny events

```spl
source="/var/log/syslog" "list 150"
```
Result: 4 matching events, all `%SEC-6-IPACCESSLOGP: list 150 denied tcp 192.168.1.12(...) -> 192.168.2.13(23)` — repeated denied Telnet (port 23) attempts from the HQ Kali host toward the Branch LAN, blocked by ACL 150 on R1.

### 6.3 Suricata flow / alert events

```spl
source="/var/log/suricata/eve.json" event_type=flow
source="/var/log/suricata/eve.json" event_type=alert
```

### 6.4 Correlated timeline (both sources, single query)

```spl
(source="/var/log/syslog" "list 150") OR (source="/var/log/suricata/eve.json" event_type=alert) | sort - _time
```
Result: 111 combined events, chronologically interleaved — the single clearest before/after demonstration of what centralized SIEM correlation buys over manually cross-referencing two log files.

---

## 7. Dashboard — "SOC Dashboard"

Two panels built under **Dashboards → Create New Dashboard**:

**Panel 1 — Denied traffic over time:**
```spl
source="/var/log/syslog" "list 150" | timechart count
```

**Panel 2 — Top source IPs hitting the deny rule:**
```spl
source="/var/log/syslog" "list 150" | rex field=_raw "denied tcp (?<src_ip>[0-9.]+)" | stats count by src_ip | sort -count
```
Result: `192.168.1.12` → 4 denied connection attempts, all toward `192.168.2.13:23`.

**Conceptual note:** the `rex` (regex field extraction) + `stats` combination in Panel 2 is doing programmatically, inside Splunk, exactly what the manual `grep -oP | sort | uniq -c | sort -rn` pipeline did by hand back in Lab 24. This is the direct, visible link between understanding what a SIEM does under the hood and being able to operate one at scale.

---

## 8. Key Takeaways

1. **Splunk's SPL is a superset of the Unix log-analysis muscle memory already built in Labs 24–25** — `timechart`, `stats`, and `rex` map directly onto `sort | uniq -c` and `grep -oP` workflows, just indexed and queryable instead of re-parsed on every run.
2. **`eve.json` over `fast.log`** is the right call whenever a source supports structured output — field-native search beats regex-at-query-time for anything that will be searched repeatedly.
3. **Disk sizing is a prerequisite, not an afterthought**, once a real indexer is in the stack. A lab environment that was comfortable for Kali + Metasploitable2 + a sensor VM was not comfortable once Splunk's own storage footprint was added.
4. **`--run-as-root` and `--no-prompt`/`--answer-yes`** are now required flags for any Splunk automation on 10.x when running under `sudo` — worth remembering for future CSOC/Wazuh-adjacent automation work.

---

## Screenshots

- [x] `01-splunk-installed-first-login.png`
- [x] `02-data-inputs-configured.png`
- [x] `03-search-firewall-denies.png`
- [x] `04-search-suricata-events.png`
- [x] `05-dashboard-denied-traffic-timechart.png`
- [x] `06-dashboard-top-source-ips.png`
- [x] `07-correlated-search-both-sources.png`
