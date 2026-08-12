# Master Mode — Roadmap Synthesis

**Status:** 30-lab roadmap complete (Levels 1–5)
**Environment:** GNS3 home lab, HQ/Branch topology, VirtualBox-based Kali, Metasploitable2, and Ubuntu sensor
**Author:** Ahmed Raza

---

## Purpose of This Document

The original roadmap closed with a synthesis checkpoint rather than a hands-on lab: build the reference architecture below from memory, then answer three questions using specifics from the actual environment rather than textbook generalities. This document is that checkpoint, written against the real lab built across Labs 1–30 rather than a hypothetical network.

```
Users → Switches → VLANs → Router → Firewall → Internet
              ↓
        IDS / SIEM Monitoring
```

## The Architecture, As Actually Built

```
HQ Site (192.168.1.0/24)                          Branch Site (192.168.2.0/24)
┌─────────────┐                                    ┌───────────────────┐
│ Kali-Linux-1│                                    │ Metasploitable2-1  │
│ (attacker)  │                                    │ (vulnerable target)│
└──────┬──────┘                                    └──────────┬─────────┘
       │                                                       │
   ┌───┴────┐        ┌────────┐   ┌────────┐        ┌────────┴──┐
   │ SW-HQ  │────────│   R1   │───│ R-ISP  │─────────│ SW-Branch │
   └───┬────┘        │ (HQ GW)│   │(transit)│        └────┬──────┘
       │              └───┬────┘   └───┬────┘              │
   ┌───┴────┐              │            │              ┌────┴────┐
   │HQ-PC1/2│         IPsec VPN tunnel (R1↔R2)          │Branch-PC│
   └────────┘         ACL 150 (Telnet deny+log)         └─────────┘
       │
  ┌────┴─────┐
  │ Ubuntu-1  │  ← dedicated sensor: rsyslog + Suricata IDS + Splunk SIEM
  │(monitoring)│    receives R1's firewall logs and IDS alerts, correlates both
  └───────────┘
```

This is not a simulated reference diagram — every device, IP, ACL, and service in it corresponds to a real, configured node in the GNS3 topology, documented individually across the 30 lab READMEs in this repository.

---

## Question 1: What Happens If the Router Fails?

**In this specific topology, R1's failure is a single point of failure with no redundancy — and that absence was never accidental; it was a scope decision, not an oversight, worth stating plainly rather than pretending otherwise.**

Concretely, if R1 goes down:

- **HQ loses its default gateway.** HQ-PC1, HQ-PC2, Kali, and the Ubuntu sensor all lose the ability to reach anything outside 192.168.1.0/24 — including the Branch site, since the only path between sites is the IPsec tunnel terminating on R1.
- **The VPN tunnel drops entirely.** Phase 1/Phase 2 SAs are stateful and tied to R1's crypto engine; there is no failover peer, so the HQ↔Branch encrypted path simply ceases to exist until R1 recovers.
- **Centralized logging stops arriving, but does not vanish.** R1's own ACL 150 deny events obviously stop being generated (there's no router to generate them), but critically, the Ubuntu sensor's own local capability — Suricata still watching the HQ segment directly, Splunk still running — is unaffected, because the sensor was deliberately built as an independent host rather than something dependent on R1 for its own operation. This was the core design decision in Lab 25: separating the monitoring plane from any single device that could fail or be compromised.
- **What this build lacks, honestly:** no HSRP/VRRP peer router, no redundant WAN link, no routing protocol (OSPF/EIGRP) with an alternate path — this environment used static routing throughout (Lab 18), which is appropriate for a two-site lab but is precisely the kind of single-point-of-failure design a real enterprise network would not accept for a device this critical. If this were being handed off as a production design rather than a training environment, this is the first gap I would flag.

**What a hardened version would add:** a second router at each site running HSRP for LAN-side gateway redundancy, and either a second WAN transit path or a routing protocol capable of converging around a failure, rather than a single static default route per site.

---

## Question 2: How Does an Attacker Get In?

This roadmap did not answer this question hypothetically — it was executed, end to end, and the results are the evidence base for the Lab 28 incident report. The actual chain, reconstructed from real logs:

1. **Reconnaissance (Lab 19).** Nmap service-version scanning against 192.168.2.13 identified vsftpd 2.3.4, an outdated Samba build, and distcc — not guessed, directly fingerprinted from banner/version responses.
2. **Exploitation (Lab 23).** Each identified service maps to a distinct entry mechanism: vsftpd 2.3.4 carried a maliciously planted backdoor (not a code flaw — a supply-chain compromise of a specific historical release) triggered via a crafted FTP username, yielding an immediate unauthenticated root shell on TCP/6200. Samba's usermap_script vulnerability (CVE-2007-2447) allowed shell metacharacter injection through malformed authentication. distcc's lack of access control allowed arbitrary command execution by any client that could reach it.
3. **Lateral reach across the VPN (Labs 18/21).** Once the site-to-site tunnel existed, the encrypted path between HQ and Branch also became the path an attacker positioned at HQ could use to reach Branch-side targets — the same tunnel protecting legitimate traffic was, by design, transparent to any traffic matching the crypto ACL, attacker traffic included, once the attacker was already inside one site.
4. **Credential attack against a second vector (Lab 27).** In parallel with direct exploitation, a Hydra-driven password-list attack against SSH recovered a valid credential (`msfadmin`), demonstrating that even a host with an already-exploited FTP/Samba surface can be entered a second, independent way if credential hygiene is weak.
5. **The common thread across all four vectors:** every single one traces back to outdated software or absent access control, not to a sophisticated technique. This is the single most important finding across the whole roadmap, and it is the honest answer to "how does an attacker get in": not through novel exploitation, but through patch management and configuration hygiene that had not been maintained. Lab 30's hardening work (SSH-only management, SNMPv3, disabled CDP/HTTP, VTY ACLs) exists specifically because the same category of exposure — unencrypted management protocols, unnecessary open services — was present on the network infrastructure itself, not just the target host.

---

## Question 3: How Does a Defender Detect It?

Detection in this environment was built in deliberate layers, and each layer's actual contribution is traceable to a specific lab rather than assumed:

**Layer 1 — Perimeter logging (Lab 24).** R1's ACL 150 provided ground-truth, device-level evidence of denied traffic (Telnet attempts specifically), forwarded via syslog. This layer's strength is that it comes directly from the enforcement point; its weakness is that it only sees what it was explicitly configured to log, and only traffic that actually reaches R1's interface.

**Layer 2 — Network-based signature detection (Lab 25).** Suricata, on an independent sensor, caught the SYN scan pattern (Lab 19) and the SSH brute-force pattern (Lab 27) through behavioral/volumetric signatures — not because any single packet looked malicious, but because the *pattern* of packets did. This layer's real, documented limitation: it only sees traffic that physically crosses a segment it can observe. The Branch-side visibility gap identified in Labs 25/27/28 — a direct consequence of GNS3's basic switch providing no SPAN/mirror capability — meant this layer had a genuine, acknowledged blind spot for traffic confined to the Branch segment.

**Layer 3 — Centralized correlation (Lab 26).** Splunk's value was never generating new detections — it was making Layer 1 and Layer 2's independent findings queryable together, in one timeline, by one analyst, instead of requiring manual cross-referencing of two separate log files by hand (which is exactly what Lab 24 did before Splunk existed in the environment). The 111-event correlated search in Lab 26/27 is the concrete proof that this layer adds value distinct from either source alone.

**Layer 4 — Analyst judgment (Lab 27, Section 3.3 of the Lab 28 report).** No log line by itself proved an attack. What made the evidence conclusive was the combination of repetition, tight timing, consistent source/destination pairing, and escalating severity across stages — the same reasoning applied by hand in Lab 24's `grep | sort | uniq -c` and formalized in Splunk's `stats count by src_ip` panels in Lab 26.

**The honest limitation across all four layers:** detection here was entirely reactive and signature/pattern-based. Nothing in this build does behavioral baselining or anomaly detection against a learned "normal" — a sufficiently patient or low-and-slow attacker, operating below the volume/timing thresholds that made Lab 27's brute-force pattern obvious, would likely not have been caught by anything built in this roadmap.

---

## What This Roadmap Actually Demonstrates

Across 30 labs, five levels, and roughly a dozen distinct tools, the throughline is narrower than it might first appear: **most of security work is not exotic technique — it is disciplined configuration, honest logging, and the patience to correlate small signals into a real finding.** The single most repeated lesson across this entire body of work, showing up independently in Lab 18 (VPN ACL direction), Lab 20 (interface selection), Lab 24 (log destination correctness), and Lab 25 (rules-loaded-but-not-restarted) was the same category of error each time: a configuration that looked correct on inspection but was subtly misaligned with what was actually running — caught not by knowing more theory, but by systematically checking assumptions against live evidence rather than trusting that a config file matched reality.

## Acknowledged Open Items

In the interest of the same honesty this document has tried to maintain throughout, two items from the roadmap remain explicitly open rather than quietly dropped:

- **Lab 22 — VLAN Hopping Simulation:** deferred pending access to a licensed Layer 2 switch image (IOSvL2 or equivalent), since GNS3's basic Ethernet switch node cannot perform real VLAN tagging or DTP negotiation.
- **Branch-side sensor visibility:** the switch-mirroring limitation identified in Lab 25 and carried into the Lab 28 incident report's Lessons Learned section. A hub-based or SPAN-equivalent tap was identified as the fix but not yet implemented and verified at the time of this writing.

## Repository Index

| Level | Labs | Focus |
|---|---|---|
| 1 — Foundation | 1–6 | Basic networking, IP addressing, DHCP/DNS, routing fundamentals |
| 2 — Switching & Routing | 7–12 | VLANs, trunking, OSPF, ACLs, NAT |
| 3 — Enterprise Network | 13–18 | Multi-site design, redundancy, wireless, monitoring, VPN |
| 4 — Attacker Mindset | 19–24 | Scanning, sniffing, MITM, exploitation, log analysis |
| 5 — Defender / SOC | 25–30 | IDS, SIEM, brute-force detection, incident response, hardening |
