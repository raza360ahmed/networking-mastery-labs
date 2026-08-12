# Networking & Cybersecurity Home Lab

A complete, hands-on 30-lab progression through networking and cybersecurity fundamentals, built and documented end to end in a self-hosted GNS3 environment. This repository is a record of practical work, not a tutorial follow-along: every configuration, attack, and detection in it was built, broken, debugged, and re-verified in a real lab rather than copied from a guide.

**Status:** Roadmap complete (30/30 labs, Levels 1-5) plus a synthesis document tying the full build together. See [`MASTER-MODE.md`](./MASTER-MODE.md) for the closing architecture review and answers to the three capstone questions the roadmap was built around.

## Objective

To build practical, defensible expertise in network engineering, offensive security, and defensive security by designing, attacking, and securing a multi-site enterprise-style network from the ground up, and to document that process honestly enough that the repository itself demonstrates the reasoning, not just the results.

## Environment

The lab runs on GNS3 with a mix of Cisco IOS devices and real VirtualBox-based hosts, rather than relying solely on simulated devices. This distinction matters: several of the more advanced labs (packet sniffing, ARP spoofing, exploitation, IDS/SIEM correlation) require genuine operating systems and genuine network stacks to be meaningful, which a pure simulator cannot provide.

```
HQ Site (192.168.1.0/24)                          Branch Site (192.168.2.0/24)
┌─────────────┐                                    ┌───────────────────┐
│ Kali Linux  │                                    │ Metasploitable2   │
│ (attacker)  │                                    │ (vulnerable target)│
└──────┬──────┘                                    └──────────┬─────────┘
       │                                                       │
   ┌───┴────┐        ┌────────┐   ┌────────┐        ┌────────┴──┐
   │Switch  │────────│  R1    │───│ R-ISP  │─────────│  Switch   │
   └───┬────┘        │(HQ GW) │   │(transit)│        └────┬──────┘
       │              └───┬────┘   └───┬────┘              │
   ┌───┴────┐              │            │              ┌────┴────┐
   │End-user│         Site-to-site IPsec VPN            │End-user │
   │  PCs   │         Firewall ACL logging               │  PCs   │
   └────────┘                                             └────────┘
       │
  ┌────┴──────┐
  │  Ubuntu    │  Dedicated sensor: rsyslog + Suricata IDS + Splunk SIEM
  │  Server    │  Independent of the attacker host by design
  └────────────┘
```

## What This Repository Includes

- A structured 30-lab progression, beginner through advanced, each with its own dated README, configuration excerpts, and screenshots
- Real network design work: VLANs, inter-VLAN routing, OSPF, NAT, ACLs, and a working site-to-site IPsec VPN between two simulated sites
- Offensive security carried out against an intentionally vulnerable target (Metasploitable2), never against any system outside this isolated environment: reconnaissance, packet sniffing, ARP-spoofing-based MITM, and exploitation of real, identifiable vulnerabilities
- Defensive security built as an independent monitoring stack: centralized firewall logging, a dedicated Suricata IDS sensor, and Splunk SIEM correlation across both sources
- A formal, NIST-aligned incident response report synthesizing a real multi-stage attack chain reconstructed from the lab's own logs
- A final hardening pass that closes the specific exposures the earlier offensive labs relied on

## Skills Demonstrated

**Network Fundamentals** — OSI model, TCP/IP, subnetting, DHCP, DNS, static and dynamic routing

**Routing & Switching** — VLANs, trunking, inter-VLAN routing, OSPF, NAT/PAT, standard and extended ACLs

**Network Security** — site-to-site IPsec VPN (ISAKMP/IPsec, correctly scoped crypto ACLs), SNMPv3, SSH-only device management, VTY access restriction, port security

**Offensive Security** — Nmap reconnaissance and service fingerprinting, Wireshark and tcpdump traffic analysis, ARP spoofing and man-in-the-middle interception, exploitation via Metasploit against real, version-identified vulnerabilities (a planted backdoor, a genuine CVE, and a service misconfiguration, treated as three distinct root causes rather than one category)

**Defensive Security** — firewall log analysis and manual frequency-based triage, Suricata IDS deployment and rule management, Splunk SIEM ingestion, correlation, and dashboarding, brute-force detection, formal incident response reporting, and infrastructure hardening

## Tools & Technologies

GNS3 · Cisco IOS (routers and switches) · VirtualBox · Kali Linux · Metasploitable2 · Ubuntu Server · Wireshark · tcpdump · Nmap · Metasploit Framework · Hydra · Suricata · Splunk Enterprise · rsyslog

## Repository Structure

| Level | Labs | Focus |
|---|---|---|
| 1 — Foundation | 1-6 | Basic network setup, OSI model, subnetting, DHCP/DNS, routing, troubleshooting |
| 2 — Switching & Routing | 7-12 | VLANs, inter-VLAN routing, trunking/STP, OSPF, ACLs, NAT |
| 3 — Enterprise Network | 13-18 | Multi-site design, redundancy, wireless, monitoring (SNMP/Syslog/NetFlow/NTP), site-to-site VPN |
| 4 — Attacker Mindset | 19-24 | Nmap scanning, packet sniffing, ARP spoofing/MITM, exploitation, firewall log analysis |
| 5 — Defender / SOC | 25-30 | IDS deployment, SIEM correlation, brute-force detection, incident response, network hardening |

Each lab folder contains a dated README documenting objective, configuration, verification steps, troubleshooting actually encountered (including dead ends and fixes, not just the final working state), key concepts, and supporting screenshots.

**Note on Lab 22 (VLAN Hopping Simulation):** deferred. This lab requires a licensed Layer 2 switch image (Cisco IOSvL2 or equivalent) with real VLAN tagging and DTP support, which GNS3's built-in Ethernet switch node cannot provide. Documented as an open item rather than skipped silently.

## A Note on How This Was Built

This repository does not present a clean, first-attempt version of events. Several labs — the VPN crypto ACL misconfiguration in Lab 18, the interface mismatches across Labs 20 and 25, an unsaved router configuration that was lost between sessions and had to be honestly reflected in later addressing rather than retroactively hidden — are documented including the debugging process, not just the resolution. That is a deliberate choice: the troubleshooting is as much a demonstration of practical skill as the final working configuration is.

## Author

Ahmed Raza — Final-year BS Digital Forensics & Cyber Security, Riphah International University, Hamdard Campus, Islamabad. Built as hands-on preparation toward CompTIA Security+ and SOC analyst / penetration testing roles.

GitHub: [`raza360ahmed`](https://github.com/raza360ahmed)
