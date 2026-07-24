# Lab 21 - Man-in-the-Middle Attack (ARP Spoofing)

## Objective

Demonstrate how an attacker can position themselves on the traffic path between two hosts on the same local network segment using ARP spoofing, without either host being aware their traffic is being intercepted. This lab builds directly on the passive sniffing concepts from Lab 20 by showing how traffic that would not naturally pass an attacker can be redirected to do so.

## Topology

Existing lab topology reused, no new nodes required:

| Node | Role | IP Address |
|---|---|---|
| Kali-Linux-1 | Attacker | Lab network interface (eth1) |
| HQ-PC2 | Victim | 192.168.2.11 |
| Metasploitable2-1 | Target the victim communicates with | 192.168.2.13 |

All three hosts reside on the same broadcast segment (SW-HQ), which is a requirement for ARP spoofing to function, since ARP traffic is not routed beyond the local segment.

## Tools Used

- `arpspoof` (part of the `dsniff` package)
- `sysctl` for IP forwarding control
- Wireshark for traffic verification

## Commands Used

```
sudo sysctl -w net.ipv4.ip_forward=1
sudo arpspoof -i eth1 -t 192.168.2.11 192.168.2.13
sudo arpspoof -i eth1 -t 192.168.2.13 192.168.2.11
arp -a
sudo wireshark
sudo sysctl -w net.ipv4.ip_forward=0
```

## Procedure

1. Recorded the legitimate ARP mapping between the victim and target before the attack, to serve as a baseline for comparison.
2. Enabled IP forwarding on the attacker host. This step is what distinguishes a man-in-the-middle attack from a denial-of-service condition: without forwarding enabled, traffic redirected to the attacker is simply dropped and the victim's connection breaks, which would be immediately noticeable.
3. Ran arpspoof in both directions simultaneously, poisoning the victim's ARP cache to associate the target's IP with the attacker's MAC address, and poisoning the target's ARP cache to associate the victim's IP with the attacker's MAC address.
4. Verified the poisoning succeeded by comparing the victim's ARP table before and after the attack.
5. Captured traffic on the attacker's interface with Wireshark to confirm that traffic between the victim and target was now passing through the attacker's machine.
6. Stopped the attack and disabled IP forwarding, allowing arpspoof's built-in cleanup behavior to restore the correct ARP mappings on both hosts.

## Tests Performed

1. Confirmed the victim's ARP table held the correct, legitimate MAC address for the target prior to the attack.
2. Confirmed IP forwarding was active on the attacker host during the attack window.
3. Confirmed both arpspoof processes ran concurrently, one for each direction of the conversation.
4. Confirmed the victim's ARP table was successfully poisoned, now resolving the target's IP to the attacker's MAC address.
5. Confirmed via Wireshark that traffic between the victim and target was visible on the attacker's interface during the attack.
6. Confirmed IP forwarding was disabled and ARP tables returned to their legitimate state after the attack concluded.

## Key Learning

- ARP has no built-in authentication. Any host on a local segment can claim ownership of any IP address, and other hosts will accept that claim without verification.
- Because ARP is not routed, ARP spoofing is inherently a local-segment attack. It cannot be performed against a remote network across a router without the attacker already having a foothold on that specific segment.
- IP forwarding is the operational detail that keeps a man-in-the-middle attack covert. Without it, the attack degrades into a denial-of-service condition that is immediately visible to the victim.
- Real ARP spoofing tools send corrective ARP packets on exit to restore legitimate mappings. Interrupting the tool improperly can leave a network segment in a broken state even after the attack has stopped, which is itself a detectable and disruptive artifact.
- The defensive countermeasures for this attack class include Dynamic ARP Inspection on managed switches and static ARP entries for critical infrastructure hosts, both of which are relevant to the hardening work planned for Level 5 of this roadmap.

## Note on Scope

All activity in this lab was performed exclusively within an isolated, self-hosted lab environment built specifically for this training roadmap. ARP spoofing and man-in-the-middle techniques against any network without explicit authorization are illegal in most jurisdictions.

## Screenshots

See the `screenshots/` folder:

- `01-arp-table-before-attack.png` - Victim's legitimate ARP entry for the target prior to the attack
- `02-ip-forwarding-enabled.png` - Confirmation that IP forwarding was enabled on the attacker host
- `03-arpspoof-running-both-directions.png` - Both arpspoof processes running concurrently
- `04-arp-table-after-attack.png` - Victim's ARP entry now resolving to the attacker's MAC address
- `05-wireshark-mitm-capture.png` - Traffic between victim and target visible on the attacker's interface
- `06-cleanup-forwarding-disabled.png` - IP forwarding disabled and ARP state restored after the attack
