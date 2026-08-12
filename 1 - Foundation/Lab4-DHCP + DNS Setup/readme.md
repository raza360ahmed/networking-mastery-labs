# Lab 04 — DHCP + DNS Setup

## 🎯 Objective
Configure Router R1 as a DHCP server for 3 department
subnets. Deploy a DNS server so PCs resolve names
automatically. Eliminate all manual IP configuration.

## 🔄 DORA Process
Discover → Offer → Request → Acknowledge
PC broadcasts → Server offers IP → PC accepts →
Server confirms. Completes in under 1 second.

## ⚙️ DHCP Pools Configured

| Pool | Network | Excluded Range | DNS |
|------|---------|---------------|-----|
| HR-POOL | 192.168.1.0/26 | .1 – .10 | 192.168.1.131 |
| SALES-POOL | 192.168.1.64/26 | .65 – .70 | 192.168.1.131 |
| IT-POOL | 192.168.1.128/26 | .129 – .135 | 192.168.1.131 |

## 🧪 Verification Commands
- show ip dhcp binding   → see all assigned leases
- show ip dhcp pool      → pool usage statistics  
- ipconfig /all          → verify PC received config
- ping router.company.com → test DNS resolution