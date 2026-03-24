# Lab 08 — Inter-VLAN Routing (Router-on-a-Stick)

## 🎯 Objective
Enable communication between VLANs 10, 20, 30 using
a single router with sub-interfaces (Router-on-a-Stick).

## ⚙️ Sub-interface Configuration

| Sub-interface | VLAN | IP (Gateway) |
|--------------|------|-------------|
| Gi0/0.10 | VLAN 10 HR | 192.168.10.1/24 |
| Gi0/0.20 | VLAN 20 Sales | 192.168.20.1/24 |
| Gi0/0.30 | VLAN 30 IT | 192.168.30.1/24 |

## 🔑 Critical Commands
encapsulation dot1Q [vlan-id]  → tag frames for this VLAN
switchport mode trunk           → carry all VLANs on one link
switchport trunk allowed vlan   → specify which VLANs allowed

## 🧪 Results
PC-HR1 ping PC-S1  → SUCCESS (routed via Gi0/0.10→.20)
PC-HR1 ping PC-IT1 → SUCCESS (routed via Gi0/0.10→.30)
tracert shows router as hop 1 confirming routing path