# Lab 07 — VLAN Creation

## 🎯 Objective
Create 3 VLANs on one physical switch to isolate
3 departments. Prove traffic isolation between VLANs.

## 🖧 VLAN Design

| VLAN | Name | Subnet | Ports |
|------|------|--------|-------|
| 10 | HR | 192.168.10.0/24 | Fa0/1-3 |
| 20 | Sales | 192.168.20.0/24 | Fa0/4-6 |
| 30 | IT | 192.168.30.0/24 | Fa0/7-9 |

## ⚙️ Key Commands
vlan 10 / name HR              → create and name VLAN
switchport mode access         → set port as access port
switchport access vlan 10      → assign port to VLAN
show vlan brief                → verify all assignments

## 🧪 Proof of Isolation
PC-HR1 ping PC-HR2 (same VLAN)    → SUCCESS ✅
PC-HR1 ping PC-S1  (diff VLAN)    → TIMEOUT ✅ (correct)

## 🔑 Key Learning
VLANs isolate at Layer 2. Cross-VLAN communication
requires a router — configured in Lab 08.