# Lab 09 — Trunking & STP

## 🎯 Objective
Build a redundant 3-switch topology. Understand
broadcast storms, STP loop prevention, and
automatic failover when links fail.

## 🖧 Topology
R1 → SW1 (Root Bridge) → SW2 + SW3
SW2 ↔ SW3 (redundant cross-link, managed by STP)

## ⚙️ Key Configuration
spanning-tree vlan 10 root primary  → force SW1 as root
spanning-tree portfast              → fast port for PCs
spanning-tree bpduguard enable      → security on PC ports
switchport mode trunk               → carry all VLANs

## 🔑 STP Port States
Blocking → Listening → Learning → Forwarding
PortFast: skips straight to Forwarding (PCs only)

## 🧪 Failover Test
Normal path: PC-A1 → SW2 → SW1 → SW3 → PC-B1
After SW1-SW2 link fails → STP unblocks SW2-SW3
Recovery time: 30-50s (STP) or 1-2s (RSTP)

## 🔑 Key Learning
Redundancy without STP = broadcast storm = network down.
STP gives you redundancy safely by blocking loop ports
and automatically recovering when primary links fail.