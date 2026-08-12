# Lab 25 - Intrusion Detection System

## Objective

Build a dedicated monitoring sensor, separate from the attacker workstation, capable of receiving centralized firewall logs and performing signature-based intrusion detection on live network traffic. This lab also establishes the foundation host that later SIEM work (Lab 26) will build on.

## Design Decision: Dedicated Sensor VM

Earlier labs in this roadmap ran logging and detection tooling directly on the Kali attacker box. This lab intentionally moves that role to a separate host for two reasons: it reflects how real security monitoring is architected, where the system generating attack traffic is never the same system responsible for detecting it, and it establishes a natural home for the SIEM platform planned in the next lab. A new Ubuntu Server 24.04.4 LTS VM was built for this purpose and added to the topology on SW-HQ, addressed via DHCP from R1's existing pool.

## Topology

| Node | Role | IP Address |
|---|---|---|
| Kali-Linux-1 | Attacker / traffic generator | 192.168.1.11 |
| Ubuntu-1 | Dedicated log receiver and IDS sensor | 192.168.1.12 |
| R1 | HQ router, firewall logging source | 192.168.1.1 |
| Metasploitable2-1 | Target, Branch site | 192.168.2.13 |

## Part 1 - Centralized Firewall Logging

R1's syslog destination, originally pointed at Kali in Lab 24, was updated to point at the new dedicated sensor:

```
no logging host 192.168.1.13
logging host 192.168.1.12
```

rsyslog on Ubuntu was configured to accept remote syslog input, which is disabled by default:

```
module(load="imudp")
input(type="imudp" port="514")
```

### Troubleshooting encountered

The interface receiving DHCP addressing on the new VM (`enp0s8`) initially came up in a down state with no address assigned, requiring manual netplan configuration before it would request a lease. Separately, the rsyslog configuration change did not initially take effect because the edited lines remained commented out in the saved file, which was only caught by directly inspecting the file content rather than assuming the edit had been applied. After correcting both issues, `ss -uln` confirmed rsyslog listening on UDP 514, and R1's access list deny events began arriving on the sensor in real time, confirmed via live `tail -f` of the syslog file during a fresh Telnet attempt from Kali.

## Part 2 - Suricata IDS

Suricata was installed on the same sensor:

```
sudo apt install -y suricata jq
```

The capture interface was set to the sensor's lab-facing NIC:

```yaml
af-packet:
  - interface: enp0s8
```

### Troubleshooting encountered

`suricata-update` initially failed with a permission error while attempting to write the downloaded ruleset to `/var/lib/suricata/`, because it had been run without `sudo`. The ruleset download itself succeeded, but installation silently failed each time until this was corrected. Once rules were successfully installed, a second issue emerged: the running Suricata process had been started before the rules existed, and updating the ruleset on disk did not affect the already-running daemon. Suricata is managed as a systemd service on this host, so restarting it (rather than manually killing and relaunching the binary, which conflicted with the existing pidfile) was required to load the current ruleset into a live process.

**Interface capture was confirmed functional independent of rule matching.** Suricata's own `stats` output in `eve.json` showed a non-zero `kernel_packets` count with zero `kernel_drops` and zero `errors` during test traffic, proving the af-packet capture layer itself was healthy well before any alert ever appeared. This separated "is Suricata seeing the traffic" from "is a rule matching the traffic" as two independent questions, which turned out to matter.

**Root cause of the missing scan alerts: signature aging, not a broken pipeline.** A plain `nmap -sS` scan (SYN, Xmas, and Null variants) against the sensor produced no alerts despite confirmed packet capture. Packet inspection with `tcpdump` on the sensor showed Nmap 7.95's default SYN packets carrying TCP window size 1024. The bundled Emerging Threats Open signature intended to catch this (`sid:2000537`, "ET SCAN NMAP -sS window 2048") hard-codes a window size of 2048 - a value tied to Nmap defaults from when the rule was authored in 2010. Fifteen years of Nmap version changes had quietly invalidated the signature. This was confirmed conclusively by manually crafting a SYN packet with `hping3 -S -w 2048` from Kali toward the sensor, which did **not** initially alert either, surfacing a second, independent problem below.

**A custom detection rule (`sid:9000001`) was written to validate the pipeline independent of the aging vendor signature**, matching on TCP SYN flags and window size 2048 without relying on the `$HOME_NET`/`$EXTERNAL_NET` directionality used by most bundled rules (see next finding). Writing and loading this rule surfaced two further configuration issues:

- The rule file (`/etc/suricata/rules/local.rules`) was not referenced anywhere in `suricata.yaml`'s `rule-files:` list, so it was silently never loaded regardless of its content or location. Suricata's own `-T` test-mode validation flag (`suricata -T -c /etc/suricata/suricata.yaml -v`) was adopted from this point forward as a mandatory pre-restart check, since it surfaces both missing-file and malformed-signature problems before they cost a debugging cycle.
- The rule itself initially failed to parse (`Signature missing required value "sid"`) because a soft-wrapped line in a text editor introduced an unintended line break mid-signature, splitting one rule into two invalid fragments. This was only caught by running `-T` validation before restarting the service, rather than assuming a saved file was syntactically intact.

**A related and now-documented directionality issue:** `HOME_NET` on the sensor is scoped to `192.168.1.0/24`, which includes Kali itself. Most ET/GPL signatures are written as `$EXTERNAL_NET any -> $HOME_NET any`, and since `EXTERNAL_NET` is defined as `!$HOME_NET`, attacker traffic originating from inside the same lab subnet as the sensor will never satisfy that directionality condition, independent of whether the signature content itself would otherwise match. This is expected behavior for a flat single-subnet lab topology and is noted here as a design constraint to keep in mind for any future signature authored or enabled on this sensor, rather than a bug to fix.

**Detection was conclusively confirmed** once the rule file was correctly referenced and validated: five `hping3 -S -p 80 -w 2048` packets from Kali produced five corresponding `CUSTOM SCAN - Nmap SYN scan window 2048 detected` (`sid:9000001`) alerts in `fast.log`, in a clean 1:1 timing correlation with the source traffic.

**Persistence across a VM restart was also verified.** After a full shutdown/restart of the GNS3 project and both VMs, `systemctl status suricata` confirmed the service returned to an active state automatically, the sensor retained its DHCP-assigned address (192.168.1.12), the af-packet interface configuration was unchanged, and the custom rule remained loaded. A fresh `hping3` test post-restart produced a new alert with a same-day timestamp, confirming the detection pipeline survives a cold restart without manual reconfiguration - R1 was separately confirmed to retain its `logging host 192.168.1.12` configuration across the same restart via its own boot log (`%SYS-6-LOGGINGHOST_STARTSTOP`).

## Part 3 - Correlation Across Two Independent Detection Sources

With both logging paths (Suricata alerts and R1's centralized syslog) confirmed live, a Telnet attempt from Kali to R1 (`telnet 192.168.1.1`) was used as a correlation test, reusing the Telnet-deny ACL (list 150) established in Lab 24. This produced an immediate, expected `No route to host` result on Kali rather than a timeout, which was initially treated as a possible new fault before being correctly identified as the expected client-side behavior when a router responds with an ICMP administratively-prohibited message to a denied TCP connection. `show access-lists` on R1 confirmed the deny rule fired (`10 deny tcp any any eq telnet log (1 match)`), and the corresponding `%SEC-6-IPACCESSLOGP` event was confirmed arriving on the sensor's syslog in real time - independently of, and at the same time as, Suricata's own alert stream. This is the intended end-state of the lab: the same attacker action observed and logged by two architecturally separate detection mechanisms.

## Open Issue - Cross-Segment Visibility (Carried Forward)

Traffic from Kali directed at the Branch-side host (`192.168.2.13`, Metasploitable2) is **not currently visible to the sensor**, regardless of protocol. This was tested with both ICMP (`ping`) and a TCP SYN probe (`hping3`) and produced no capture-side evidence on the sensor in either case, which rules out the earlier signature-related findings above as the cause here - this is a distinct, unresolved problem.

The suspected root cause is topological rather than configuration-related: Kali and the sensor sit on the same access switch (`SW-HQ`), but traffic to `192.168.2.13` is routed `Kali -> SW-HQ -> R1 -> R-ISP -> R2 -> SW-Branch -> Metasploitable2`. This is unicast traffic addressed directly to R1's MAC address at the SW-HQ hop; a standard learning switch (which is what GNS3's built-in Ethernet Switch node is) will forward it only out the port toward R1, not out the sensor's port, since the destination MAC is known and traffic mirroring/SPAN is not a feature of that node type. This differs from the Kali-to-sensor traffic used throughout Part 2, which is directly addressed to the sensor and therefore always visible to it regardless of switch behavior.

A GNS3 Hub node (`Hub1`) has since been added to the topology as a planned fix, on the basis that a hub floods all traffic to all connected ports rather than learning MAC-to-port mappings, which would make R1-bound traffic passing through that segment visible to the sensor. Wiring this in without disturbing the working Part 1/2 configuration, and re-confirming both local and cross-segment detection afterward, is the next step and is being carried forward rather than treated as resolved in this writeup.

## Tests Performed

1. Confirmed rsyslog actively listening on UDP 514 on the sensor.
2. Confirmed R1's access list deny events (from the Lab 24 Telnet block rule) arriving on the sensor in real time.
3. Confirmed Suricata running as an active systemd service with 52,099+ rules successfully loaded from the Emerging Threats Open ruleset.
4. Confirmed raw TCP connectivity and payload delivery between Kali and the sensor using netcat, independent of Suricata.
5. Confirmed af-packet capture layer healthy via Suricata's own `kernel_packets`/`kernel_drops` stats, independent of rule matching.
6. Identified and documented signature aging against the bundled Nmap SYN-scan rule (window size 2048 vs. modern Nmap's default of 1024) as the cause of missing alerts on real scan traffic.
7. Authored, debugged, and validated a custom local detection rule (`sid:9000001`), confirmed firing on Kali-to-sensor traffic with correct 1:1 packet-to-alert correlation.
8. Confirmed detection pipeline (service state, interface config, rule loading, DHCP addressing) survives a full VM/project restart without manual reconfiguration.
9. Confirmed dual, independent, correlated detection of the same Telnet attempt via R1's ACL syslog and Suricata's own alert stream.
10. Identified that Branch-side traffic (192.168.2.13) is not visible to the sensor, isolated the cause to switch MAC-learning behavior on SW-HQ rather than any Suricata- or rule-side issue, and staged a hub-based topology fix - not yet re-verified.

## Key Learning

- Separating the attacker host from the detection host is a deliberate architectural choice, not just a convenience. A compromised or disabled attacker box should never be able to affect the sensor watching it.
- A syslog destination configuration change on a network device and a listener configuration change on the receiving host are two independent points of failure. Both were verified separately rather than assumed to work together.
- Installing or updating detection rules and having a running detection process actually use them are two different operations. A long-running daemon does not automatically reload rules that change on disk; it must be restarted, and on a systemd-managed service this should be done through systemctl rather than manual process management, to avoid pidfile conflicts.
- "Packets are being captured" and "a rule is matching those packets" are separate layers that fail independently and need to be verified separately. Suricata's own stats counters (`kernel_packets`, `kernel_drops`) are the right tool to isolate a capture-layer problem from a detection-layer one before assuming the whole pipeline is broken.
- Vendor detection signatures are not permanently correct. A signature written against a specific tool's default behavior (in this case, Nmap's TCP window size) can silently stop matching years later as that tool's defaults change, with no error or warning - the rule simply never fires. Confirming a "known-bad" signature still triggers against a manually crafted packet matching its original assumptions is a useful way to isolate this from a genuine capture or configuration fault.
- A new local rule file must be explicitly referenced in `suricata.yaml`'s `rule-files:` list; its mere existence on disk and correct location under `default-rule-path` is not sufficient for it to be loaded.
- Soft line-wrapping in a text editor can silently split a single rule into multiple invalid fragments. Suricata's `-T` test-mode flag catches this and other configuration errors before a restart, and has been adopted as a standard pre-restart check going forward.
- `$HOME_NET`/`$EXTERNAL_NET` directionality in bundled signatures assumes attacker and target sit on different sides of that boundary. In a flat single-subnet lab where the attacker and the sensor share the same `HOME_NET` scope, externally-directional rules will never fire against internally-sourced traffic regardless of packet content - a constraint to design around when writing or enabling future signatures on this sensor.
- Confirming a monitoring pipeline works end to end requires correct timing between generating test traffic and actively watching for the result, not just checking logs after the fact. Several early troubleshooting steps in this lab produced misleading "no results" outcomes purely because the log was checked before or after the relevant traffic, not during it - stale log entries from a previous session were, on at least one occasion, mistaken for a fresh, still-broken result.
- A working detection setup should be re-verified after any environment restart rather than assumed to persist; in this case it did, but confirming that explicitly (rather than assuming it) is now part of the standard workflow.
- Visibility gaps in a monitored network are as often a switching/topology problem as a detection-tooling problem. A standard learning switch will not forward unicast traffic to a port that isn't part of that traffic's path, which limits IDS visibility to locally-addressed traffic unless mirroring, a hub, or a SPAN-capable device is deliberately placed in the path.
- Standard external IDS sanity-check techniques, such as fetching a known test URL, assume outbound internet access from the test host. This lab environment's simulated WAN does not provide that, so an equivalent test was constructed using tools already present in the lab (netcat) instead.

## Status

Firewall log centralization to the dedicated sensor is complete and verified, including post-restart persistence. Suricata is installed, correctly configured, running with a full ruleset loaded, and alert generation has been conclusively confirmed for locally-addressed (Kali-to-sensor) traffic via a custom rule, after isolating and documenting a vendor signature-aging issue as the root cause of the original non-detection. Dual-source correlation (R1 ACL syslog + Suricata) against the same attacker action has been verified. Cross-segment detection (Branch-side traffic) remains unresolved pending a topology change (hub insertion) and is the next item to close out before this lab is considered fully complete.

## Screenshots

See the `screenshots/` folder:

- `01-ubuntu-vm-dhcp-address-confirmed.png` - Sensor VM addressing on the lab segment
- `02-rsyslog-listening-confirmed.png` - rsyslog bound to UDP 514
- `03-r1-logs-received-on-sensor.png` - Live firewall deny events arriving on the sensor
- `04-suricata-installed-buildinfo.png` - Suricata installation confirmation
- `05-suricata-interface-config.png` - af-packet interface configuration
- `06-suricata-update-rules-pulled.png` - Ruleset download and installation, including the permission error and its resolution
- `07-suricata-service-status.png` - Suricata running as an active systemd service with rules loaded
- `08-netcat-payload-delivery.png` - Confirmed raw connectivity test between Kali and the sensor
- `09-signature-aging-tcpdump-window-size.png` - tcpdump evidence of Nmap's actual window size vs. the outdated signature's expected value
- `10-custom-rule-parse-error.png` - Suricata `-T` validation catching the soft-wrap line-split error
- `11-custom-rule-detection-confirmed.png` - Five hping3 packets, five corresponding sid:9000001 alerts
- `12-post-restart-persistence-confirmed.png` - Fresh same-day alert after a full VM/project restart
- `13-correlation-telnet-acl-and-suricata.png` - R1 ACL deny + syslog entry alongside the corresponding sensor-side view
- `14-branch-side-visibility-gap.png` - No capture evidence for 192.168.2.13-directed traffic on the sensor (open issue)
