# Lab 10 — OSPF Dynamic Routing

## 🎯 Objective
Replace static routing with OSPF on a 3-router network.
Prove automatic route discovery and failover recovery.

## 🖧 Topology
Router-A (1.1.1.1) ── Router-B (2.2.2.2)
     \                      /
      ────── Router-C (3.3.3.3)

LANs: 192.168.1.0/24, 192.168.2.0/24, 192.168.3.0/24
WANs: 10.0.0.0/30, 10.0.1.0/30, 10.0.2.0/30

## ⚙️ OSPF Configuration
router ospf 1
router-id [x.x.x.x]
network [network] [wildcard] area 0

## 🔑 Key Commands
show ip ospf neighbor   → verify adjacencies (must show FULL)
show ip route           → see O routes learned via OSPF
show ip ospf database   → full link state database

## 🧪 Failover Test
Shut down Router-A → Router-B direct link.
OSPF reroutes via Router-C in under 40 seconds.
No static routes. No manual intervention. Automatic.

## 🔑 Key Learning
OSPF = self-healing network. Links fail, OSPF recovers.
This is why every enterprise uses dynamic routing.