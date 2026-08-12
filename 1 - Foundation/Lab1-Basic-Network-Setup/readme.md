# Lab 01 — Basic Network Setup

## 🎯 Objective
Build a basic LAN with 3 PCs, 1 switch, and 1 router.
Understand how devices communicate within a subnet and 
how the router acts as a default gateway.

## 🛠️ Tools Used
- Cisco Packet Tracer 8.x
- Router: Cisco 2911
- Switch: Cisco 2960

## 🖧 Network Topology
![Topology](screenshots/topology.png)

## 📋 IP Addressing Table

| Device | Interface | IP Address | Subnet Mask | Default Gateway |
|--------|-----------|------------|-------------|-----------------|
| R1 | GigabitEthernet0/0 | 192.168.1.1 | 255.255.255.0 | — |
| PC1 | NIC | 192.168.1.10 | 255.255.255.0 | 192.168.1.1 |
| PC2 | NIC | 192.168.1.11 | 255.255.255.0 | 192.168.1.1 |
| PC3 | NIC | 192.168.1.12 | 255.255.255.0 | 192.168.1.1 |

## ⚙️ Configuration

### Router R1
\```
enable
configure terminal
hostname R1
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
 description LAN-Interface
exit
write memory
\```

## 🧪 Testing & Verification

### Ping Test — PC1 to PC2
![Ping Test](screenshots/ping-test.png)

### Interface Verification
![Interface Brief](screenshots/interface-brief.png)

## 📖 Key Concepts Learned
- A **subnet** groups devices that communicate directly 
  without a router
- Router interfaces are **off by default** — `no shutdown` 
  is always required
- The **default gateway** is the router's IP — it is the 
  exit door for traffic leaving the local network
- `write memory` saves config to NVRAM — without it, 
  reboot erases everything

## 🔗 Real World Application
This is the foundation of every office network. Every 
company has this exact setup — PCs on a switch, switch 
connected to a router, router connecting to the internet.