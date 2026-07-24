# Lab 21 - Man-in-the-Middle Attack (ARP Spoofing)

## Objective

Demonstrate how an attacker can position themselves on the traffic path between a victim host and its default gateway using ARP spoofing, without the victim being aware their traffic is being intercepted. This lab builds directly on the passive sniffing concepts from Lab 20 by showing how traffic that would not naturally pass an attacker can be redirected to do so.

## Note on Addressing

The addressing used in this lab differs from Labs 18 through 20. Between sessions, the lab environment lost its running configuration because changes had not been saved with `write memory` before the devices restarted. The topology was rebuilt from scratch, and during the rebuild the HQ and Branch subnets were assigned in reverse of the original layout. Rather than reworking the entire lab history to match the original numbering, the current live addressing was adopted as the new baseline going forward. All configuration is internally consistent as documented below.

## Topology

Existing lab topology reused, no new nodes required:

| Node | Role | IP Address | Switch |
|---|---|---|---|
| Kali-Linux-1 | Attacker | 192.168.1.13 | SW-HQ |
| HQ-PC2 | Victim | 192.168.1.12 | SW-HQ |
| R1 | Default gateway being impersonated | 192.168.1.1 | SW-HQ |
| Metasploitable2-1 | Cross-segment host used to demonstrate full interception | 192.168.2.13 | SW-Branch |

The attack targets the relationship between the victim and its default gateway rather than a single peer-to-peer connection. This was a deliberate adjustment from the original lab plan: after the rebuild, Kali and Metasploitable2 ended up on different switch segments connected only through routing, and ARP spoofing only functions within a single broadcast segment. Targeting the gateway relationship is also the more realistic and more commonly used version of this attack in practice, since it exposes everything the victim sends anywhere, not just traffic to one specific host.

## Tools Used

- `arpspoof` (part of the `dsniff` package)
- `sysctl` for IP forwarding control
- Wireshark for traffic verification

## Commands Used

```
sudo sysctl -w net.ipv4.ip_forward=1
sudo arpspoof -i eth1 -t 192.168.1.12 192.168.1.1
sudo arpspoof -i eth1 -t 192.168.1.1 192.168.1.12
sudo wireshark
sudo sysctl -w net.ipv4.ip_forward=0
```

## Procedure

1. Enabled IP forwarding on the attacker host. This is what distinguishes a man-in-the-middle attack from a denial-of-service condition: without forwarding enabled, traffic redirected to the attacker is simply dropped and the victim's connection breaks, which would be immediately noticeable.
2. Ran arpspoof in both directions simultaneously: one process told the victim (HQ-PC2) that the attacker's MAC address belonged to the gateway (192.168.1.1), and a second process told the gateway that the attacker's MAC address belonged to the victim.
3. Confirmed the victim's ARP cache was successfully poisoned by generating traffic from the victim and then inspecting its ARP table directly.
4. Captured traffic on the attacker's interface with Wireshark to confirm that traffic from the victim, destined for a host on a different subnet, was passing through the attacker's machine before continuing to its real destination.
5. Observed an additional artifact during capture: ICMP Redirect messages generated automatically by the attacker's own Linux kernel, a side effect of forwarding traffic back out the same interface it arrived on.
6. Stopped the attack and disabled IP forwarding, allowing arpspoof's built-in cleanup behavior to restore the correct ARP mappings on both hosts.

## Tests Performed

1. Confirmed both arpspoof processes ran concurrently, one for each direction of the victim-to-gateway relationship.
2. Generated ICMP traffic from the victim to its gateway, then inspected the victim's ARP table directly. The table resolved both the gateway's IP address and the attacker's own IP address to the identical MAC address, confirming the victim's ARP cache had been successfully poisoned rather than merely observing the attacker's outbound spoof traffic.
3. Generated ICMP traffic from the victim to a host on a separate subnet reachable only through the gateway. Wireshark confirmed this traffic was visible on the attacker's interface, proving genuine interception rather than a simple broken connection.
4. Observed the arpspoof cleanup sequence on exit, confirming that stopping the tool automatically restored the legitimate ARP mappings rather than leaving the segment in a poisoned state.
5. Confirmed IP forwarding was disabled after the attack concluded.

## Key Learning

- ARP has no built-in authentication. Any host on a local segment can claim ownership of any IP address, and other hosts will accept that claim without verification.
- Because ARP is not routed, ARP spoofing is inherently a local-segment attack. It cannot be performed against a remote network across a router without the attacker already having a foothold on that specific segment. This constraint directly shaped the target selection in this lab after the environment's addressing changed.
- Targeting a victim's relationship with its default gateway, rather than a single peer connection, is the more realistic and higher-value version of this attack, since it exposes all of the victim's outbound traffic rather than a single conversation.
- IP forwarding is the operational detail that keeps a man-in-the-middle attack covert. Without it, the attack degrades into a denial-of-service condition that is immediately visible to the victim.
- A poisoned victim's ARP table is direct, victim-side evidence of a successful attack and is stronger evidence than observing the attacker's own spoofed packets in isolation.
- Real ARP spoofing tools send corrective ARP packets on exit to restore legitimate mappings. Interrupting the tool improperly can leave a network segment in a broken state even after the attack has stopped, which is itself a detectable and disruptive artifact.
- ARP spoofing on a Linux-based attacker host can produce secondary artifacts, such as automatically generated ICMP Redirect messages, when the attacker forwards traffic back out the interface it arrived on. These artifacts represent a real detection opportunity for a defender monitoring for unexpected ICMP redirect traffic.
- The defensive countermeasures for this attack class include Dynamic ARP Inspection on managed switches and static ARP entries for critical infrastructure hosts, both of which are relevant to the hardening work planned for Level 5 of this roadmap.

## Note on Scope

All activity in this lab was performed exclusively within an isolated, self-hosted lab environment built specifically for this training roadmap. ARP spoofing and man-in-the-middle techniques against any network without explicit authorization are illegal in most jurisdictions.

## Screenshots

See the `screenshots/` folder:

- `01-topology.png` - Current GNS3 topology showing all lab nodes
- `02-ip-forwarding-enabled.png` - Confirmation that IP forwarding was enabled on the attacker host
- `03-arpspoof-poisoning-victim.png` - arpspoof poisoning the victim's view of the gateway
- `03-arpspoof-poisoning-gateway.png` - arpspoof poisoning the gateway's view of the victim
- `04-arp-table-poisoned-victim-confirmed.png` - Victim's ARP table showing both the gateway and the attacker's own IP resolving to the attacker's MAC address
- `05-wireshark-mitm-capture.png` - Cross-subnet ICMP traffic from the victim visible on the attacker's interface, including the ICMP Redirect artifact
- `06-arpspoof-cleanup-on-exit.png` - Automatic restoration of legitimate ARP mappings when arpspoof was stopped
- `07-ip-forwarding-disabled.png` - IP forwarding disabled after the attack concluded
