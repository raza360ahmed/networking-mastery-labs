# Lab 13 — Multi-Floor Enterprise Network

## 🎯 Objective
Design and build a complete 3-floor company network
for TechCorp Pakistan with 4 departments, centralised
servers, internet access, DHCP, and security policies.

## 🏢 Network Design
Floor 1: Data Centre — Server (DNS/DHCP), Core Switch, Router
Floor 2: HR (VLAN 10) + Sales (VLAN 20)
Floor 3: Management (VLAN 30) + IT (VLAN 40)

## 🔑 Technologies Used
Layer 3 switching (SVIs) · VLANs · Trunking · STP
DHCP relay (ip helper-address) · NAT/PAT · ACLs
Static routing (summary route) · PortFast

## 📋 VLAN Summary
VLAN 10 HR:    192.168.10.0/24 → GW 192.168.10.1
VLAN 20 Sales: 192.168.20.0/24 → GW 192.168.20.1
VLAN 30 Mgmt:  192.168.30.0/24 → GW 192.168.30.1
VLAN 40 IT:    192.168.40.0/24 → GW 192.168.40.1
VLAN 99 Srv:   192.168.99.0/24 → GW 192.168.99.1

## 🧪 Verification
All PCs get IPs via DHCP automatically
Cross-VLAN routing works via SW-Core SVIs
Internet access works via NAT on R1
HR/Sales cannot reach IT (ACL enforced)