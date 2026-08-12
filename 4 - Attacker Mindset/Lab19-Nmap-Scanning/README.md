# Lab 19 - Network Scanning (Nmap)

## Objective

Learn network reconnaissance techniques using Nmap from an attacker's perspective, using a real Linux target with genuine running services, to understand what information a network exposes during the scanning phase of an attack and why detection of these techniques matters.

## Topology Addition

| Node | Role | IP Address |
|---|---|---|
| Kali-Linux-1 | Attacker workstation | Assigned via lab network |
| Metasploitable2-1 | Real scan target | 192.168.2.13 (DHCP) |

Both nodes were added as VirtualBox-based GNS3 nodes rather than VPCS, since VPCS has no real service stack and produces no meaningful scan results.

Connectivity was verified end-to-end before scanning began, including across the existing Lab 18 IPsec VPN tunnel (192.168.1.1 to 192.168.2.1 and to the target host).

## Commands Used

```
nmap -sn 192.168.2.0/24        Host discovery (ping sweep)
nmap 192.168.2.13              Default top-1000 port scan
nmap -sV 192.168.2.13          Service version detection
nmap -p- 192.168.2.13          Full 65535 port scan
sudo nmap -O 192.168.2.13      OS fingerprinting
sudo nmap -A 192.168.2.13      Aggressive combined scan
sudo nmap -sS 192.168.2.13     SYN stealth scan
```

## Tests Performed

1. Host discovery confirmed all live hosts on the 192.168.2.0/24 segment, including the real target.
2. Default scan against the real target returned genuine open ports (FTP, SSH, Telnet, HTTP, and others), unlike earlier attempts against VPCS nodes which returned no meaningful results.
3. Service version detection identified specific, real software versions running on the target.
4. Full port range scan confirmed no additional services hidden outside the default top-1000 ports.
5. OS fingerprinting identified the target's operating system family.
6. Aggressive scan combined OS detection, version detection, and script scanning in a single pass.
7. SYN stealth scan completed without full TCP handshakes being established.
8. Packet capture on the Kali-to-switch link during the SYN scan confirmed the expected pattern: SYN sent, SYN-ACK received, RST sent, with no completed three-way handshake.

## Key Learning

- Reconnaissance is the first phase of the attack lifecycle in frameworks such as the Cyber Kill Chain and MITRE ATT&CK.
- A real target with genuine services is necessary to produce meaningful scan results; lightweight IP stack simulators such as VPCS do not run real services and are not suitable for this type of lab.
- Service version detection is the pivot point between reconnaissance and exploitation, since specific software versions can be checked against known vulnerabilities.
- A SYN scan avoids completing the full TCP handshake, which historically avoided some connection logging mechanisms, though modern IDS/IPS systems are built to detect this pattern regardless.
- Scanning any network without explicit authorization is illegal in most jurisdictions. All scanning in this lab was performed exclusively against an isolated, self-hosted lab environment built specifically for this purpose.

## Screenshots

See the `screenshots/` folder:

- `01-topology-lab19.png` - GNS3 topology showing Kali-Linux-1 and Metasploitable2-1 added to the network
- `02-host-discovery.png` - Nmap ping sweep results
- `03-basic-scan.png` - Default scan against the real target
- `04-service-version-detection.png` - Service version detection results
- `05-full-port-scan.png` - Full 65535 port scan
- `06-os-detection.png` - OS fingerprinting result
- `07-aggressive-scan.png` - Combined aggressive scan output
- `08-syn-stealth-scan.png` - SYN stealth scan output
- `09-wireshark-syn-capture.png` - Packet capture showing incomplete TCP handshakes during the SYN scan
