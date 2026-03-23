# Lab 05 — Static vs Dynamic Routing

## 🎯 Objective
Connect two office buildings using static routing.
Understand routing tables, default routes, and
why static routing fails when links go down.

## 🖧 Topology
Building A: 192.168.1.0/24 — Router-A (Gi0/0: .1)
Building B: 192.168.2.0/24 — Router-B (Gi0/0: .1)
WAN Link:   10.0.0.0/30 — Router-A (.1) ↔ Router-B (.2)

## 📋 Static Routes Configured
Router-A: ip route 192.168.2.0 255.255.255.0 10.0.0.2
Router-B: ip route 192.168.1.0 255.255.255.0 10.0.0.1
Default:  ip route 0.0.0.0 0.0.0.0 10.0.0.2

## 🔑 Key Learning
Static routing breaks silently when links fail.
Dynamic routing (OSPF — Lab 10) solves this by
automatically detecting failures and rerouting.