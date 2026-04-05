# Lab 16 — Load Balancing

## Objective
Simulate enterprise load balancing across 3 web servers
using DNS Round Robin, Static NAT, and ECMP routing,
with health monitoring via Cisco IP SLA.

## Topology
Internet Users
↓
ISP Router
↓
R1 (Active) ←HSRP→ R2 (Standby)
↓
SW-Core
/    |    
Srv1  Srv2  Srv3
.10   .11   .12
(VLAN 99 — Server Subnet)

## Server Farm Configuration

| Server | IP | Subnet Mask | Gateway |
|--------|-----|-------------|---------|
| Web-Srv1 | 192.168.99.10 | 255.255.255.0 | 192.168.99.100 |
| Web-Srv2 | 192.168.99.11 | 255.255.255.0 | 192.168.99.100 |
| Web-Srv3 | 192.168.99.12 | 255.255.255.0 | 192.168.99.100 |

## Load Balancing Methods Implemented

| Method | Mechanism | Limitation |
|--------|-----------|------------|
| DNS Round Robin | 3 A records for www.techcorp.com | PT resolves to first record only |
| Static NAT | Each server mapped to public IP | Manual distribution |
| ECMP | Equal cost routes per server | Router-level only |
| IP SLA | ICMP probe every 10s | Not supported in PT |

## Key Configurations

### DNS Round Robin (Server0)
www.techcorp.com → 192.168.99.10
www.techcorp.com → 192.168.99.11
www.techcorp.com → 192.168.99.12

### Static NAT Load Balancing (R1)
ip nat inside source static tcp 192.168.99.10 80 203.0.113.10 80
ip nat inside source static tcp 192.168.99.11 80 203.0.113.11 80
ip nat inside source static tcp 192.168.99.12 80 203.0.113.12 80

### ECMP Routes (R1)
ip route 192.168.99.10 255.255.255.255 192.168.99.100
ip route 192.168.99.11 255.255.255.255 192.168.99.100
ip route 192.168.99.12 255.255.255.255 192.168.99.100

### IP SLA Health Monitoring (Real IOS Only)
ip sla 1
icmp-echo 192.168.99.10 source-interface GigabitEthernet0/0
frequency 10
ip sla schedule 1 life forever start-time now
ip sla 2
icmp-echo 192.168.99.11 source-interface GigabitEthernet0/0
frequency 10
ip sla schedule 2 life forever start-time now
> Note: IP SLA is not supported in Packet Tracer.
> Config provided for real IOS reference only.

## Failover Test Results

| Test | Result |
|------|--------|
| ping 192.168.99.10 | ✅ |
| ping 192.168.99.11 | ✅ |
| ping 192.168.99.12 | ✅ |
| Web-Srv2 powered off → ping .11 | ❌ Timeout |
| Remaining servers still serving | ✅ |

## PT Limitation Note
Packet Tracer does not support:
- True DNS round robin rotation
- IP SLA monitoring
- Layer 7 load balancing

In production, dedicated load balancers (F5, HAProxy,
AWS ALB) provide health checks, instant failover,
session persistence, and detailed analytics.

## Key Concepts Learned
- DNS Round Robin = simple but no health awareness
- Real load balancers = health checks + instant failover  
- Layer 4 = IP/port based (fast but dumb)
- Layer 7 = HTTP header aware (intelligent routing)
- Session persistence = critical for stateful apps