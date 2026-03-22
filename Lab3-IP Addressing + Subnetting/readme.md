# Lab 03 — IP Addressing + Subnetting

## 🎯 Objective
Divide 192.168.1.0/24 into 3 department subnets using /26.
Configure inter-subnet routing through Router R1.

## 🧮 Subnet Calculation
Starting network: 192.168.1.0/24
Required subnets: 3 departments
Chosen prefix: /26 (62 usable hosts per subnet, block size 64)

| Department | Network | Usable Range | Broadcast | Gateway |
|------------|---------|-------------|-----------|---------|
| HR | 192.168.1.0/26 | .1 – .62 | .63 | 192.168.1.1 |
| Sales | 192.168.1.64/26 | .65 – .126 | .127 | 192.168.1.65 |
| IT | 192.168.1.128/26 | .129 – .190 | .191 | 192.168.1.129 |

## 🧠 Key Formula
Block size = 256 - last octet of subnet mask
/26 → 256 - 192 = 64 → subnets increment by 64