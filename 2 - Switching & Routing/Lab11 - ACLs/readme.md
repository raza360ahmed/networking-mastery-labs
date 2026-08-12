# Lab 11 — Access Control Lists (ACLs)

## 🎯 Objective
Implement network security using Standard, Extended,
and Named ACLs. Control traffic between departments
based on real company security policies.

## 🔐 Scenarios Implemented

| Scenario | Type | Rule | Result |
|----------|------|------|--------|
| Block guest PC | Standard | deny host 192.168.1.10 | Single PC blocked |
| Block HR→IT | Standard | deny 192.168.1.0/24 | Dept isolation |
| Block SSH only | Extended | deny tcp HR→IT eq 22 | Port-level control |
| Mgmt access | Named Extended | Only IT can SSH routers | Role-based access |

## ⚙️ Key Syntax
access-list [#] [permit/deny] [protocol] [src] [dst] [port]
ip access-group [name/#] [in/out]

## 🔑 Rules to memorise
1. First match wins — order matters
2. Implicit deny at bottom — always add permit ip any any
3. Standard ACL → place near destination
4. Extended ACL → place near source
5. Named ACLs → always use in production

## 🛠️ Verification
show access-lists          → rules + match counts
show ip interface Gi0/0    → which ACL applied where