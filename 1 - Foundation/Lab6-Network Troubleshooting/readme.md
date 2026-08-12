# Lab 06 — Network Troubleshooting

## 🎯 Objective
Master systematic bottom-up OSI troubleshooting.
Diagnose and fix 7 common network failure scenarios.

## 🔧 Scenarios Practised

| # | Fault Injected | Symptom | Fix |
|---|---------------|---------|-----|
| 1 | Wrong IP | Cannot reach gateway | Correct IP address |
| 2 | Missing no shutdown | Interface down | no shutdown |
| 3 | Missing static route | Remote network unreachable | Add ip route |
| 4 | Wrong subnet mask | Intermittent local failures | Correct mask |
| 5 | Wrong gateway | Cross-network fails | Set correct gateway |
| 6 | Duplicate IP | Random intermittent failures | Remove duplicate |
| 7 | Full verification | — | All commands pass |

## 🛠️ Key Commands
show ip interface brief → interface status
show ip route          → routing table
show mac address-table → Layer 2 MAC learning
show arp               → IP to MAC mappings
tracert [ip]           → path tracing