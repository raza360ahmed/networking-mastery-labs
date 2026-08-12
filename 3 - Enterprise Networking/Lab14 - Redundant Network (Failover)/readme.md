# Lab 14 — Redundant Network (Failover)

## 🎯 Objective
Eliminate single points of failure using HSRP gateway
redundancy and EtherChannel link redundancy. Network
must survive any single device or link failure.

## 🔧 Technologies Implemented
HSRP — gateway redundancy between R1 and R2
EtherChannel — link bundling between switches
Interface tracking — smart failover on WAN failure

## ⚙️ HSRP Configuration
Virtual IP: 192.168.99.1 (shared between R1 and R2)
R1: priority 110, preempt, track Gi0/1 decrement 30
R2: priority 100, preempt (standby)

## 🧪 Failover Test
1. Confirm ping works through R1 (Active)
2. Shut down R1 Gi0/0
3. Wait 10 seconds — R2 becomes Active
4. Ping still works — zero manual intervention
5. Re-enable R1 — reclaims Active via preempt

## 📸 Screenshots
show standby brief (R1 Active, R2 Standby)
show standby brief (after failover — R2 Active)
ping test during failover

## 🔑 Key Learning
Single points of failure are unacceptable in production.
HSRP + EtherChannel + dual WAN = network that
self-heals from any single failure automatically.