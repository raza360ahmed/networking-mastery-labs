# Lab 15 — Wireless Network (Secure WiFi)

## 🎯 Objective
Add enterprise wireless to TechCorp network with
3 separate SSIDs mapped to isolated VLANs.
Implement Guest isolation using ACLs.

## 📡 Wireless Design

| SSID | VLAN | Subnet | Security | Access |
|------|------|--------|----------|--------|
| TechCorp-Corp | 10-40 | 192.168.x.0/24 | WPA2-PSK | Full LAN |
| TechCorp-Guest | 50 | 192.168.50.0/24 | WPA2-PSK | Internet only |
| TechCorp-IOT | 60 | 192.168.60.0/24 | WPA2-PSK | Internet only |

## 🔐 Security Implementation
GUEST-ISOLATION ACL → blocks Guest from all corp VLANs
IOT-ISOLATION ACL   → blocks IoT from all other VLANs
Applied inbound on respective VLAN SVIs on SW-Core

## 🧪 Verification
Corporate laptop → ping internal server ✅
Guest laptop → ping corporate PC ❌ (blocked)
Guest laptop → ping 8.8.8.8 ✅ (internet allowed)

## 🔑 Key Learning
WiFi security = encryption + isolation + authentication
WPA2-Enterprise with 802.1X = gold standard for corporate
Guest VLAN + ACL = guests get internet, not your data