# Lab 12 — NAT (Network Address Translation)

## 🎯 Objective
Configure PAT (NAT Overload) to allow all internal
devices to share one public IP for internet access.
Configure Static NAT for an internal web server.

## 🖧 Topology
LAN: 192.168.1.0/24 → Router-A → Internet (203.0.113.x)
Inside: Gi0/0 (192.168.1.1)
Outside: Gi0/1 (203.0.113.2)

## ⚙️ NAT Configuration
! Mark interfaces
ip nat inside  → LAN interface
ip nat outside → WAN interface

! PAT (all hosts share one public IP)
access-list 1 permit 192.168.1.0 0.0.0.255
ip nat inside source list 1 interface Gi0/1 overload

! Static NAT (web server)
ip nat inside source static 192.168.1.10 203.0.113.10

## 🔑 NAT Translation Table
Inside local   → real private IP
Inside global  → public IP seen by internet
show ip nat translations → verify all entries
show ip nat statistics  → hits and misses

## 🧪 Results
PC1 ping 8.8.8.8 → SUCCESS (translated via PAT)
3 PCs share 203.0.113.2 differentiated by port number
Web server reachable at fixed public IP 203.0.113.10

## 🔑 Key Learning
PAT = entire network shares ONE public IP using ports.
This is how every home and office connects to internet.
NAT also provides basic security by hiding internal IPs.