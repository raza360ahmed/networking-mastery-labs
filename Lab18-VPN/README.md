# Lab 18 - Site-to-Site VPN (IPsec)

## Objective

Connect two separate office networks (HQ and Branch) securely across an untrusted transit network using a site-to-site IPsec VPN, so that traffic between the two private LANs is encrypted end-to-end.

## Topology

```
[HQ-PC1] [HQ-PC2]--[SW-HQ]--[R1]====[R-ISP]====[R2]--[SW-Branch]--[Branch-PC1] [Branch-PC2]
         192.168.2.0/24   Gi0/0                Gi0/0    192.168.1.0/24
                        203.0.113.1          198.51.100.1
```

| Device | Role | Interface | IP Address |
|---|---|---|---|
| R1 | HQ router | Fa0/0 (WAN) | 203.0.113.1 |
| R1 | HQ router | Fa0/1 (LAN) | 192.168.2.1 |
| R2 | Branch router | Fa0/0 (WAN) | 198.51.100.1 |
| R2 | Branch router | Fa0/1 (LAN) | 192.168.1.1 |
| R-ISP | Simulated internet transit | Fa0/0 / Fa0/1 | 203.0.113.2 / 198.51.100.2 |

R-ISP performs no VPN function. It only routes between the two public subnets, simulating the untrusted internet path between HQ and Branch.

## Technologies Configured

| Technology | Purpose |
|---|---|
| ISAKMP / IKE Phase 1 | Negotiates a secure channel and authenticates both peers |
| IPsec / IKE Phase 2 | Encrypts and verifies the actual data traffic |
| Extended ACL | Defines which traffic is sent through the tunnel |
| Crypto Map | Binds the peer, transform-set, and ACL to a physical interface |

## Configuration Summary

### Phase 1 (ISAKMP policy)

```
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 3600

crypto isakmp key <shared-key> address <peer-public-ip>
```

### Phase 2 (IPsec transform set)

```
crypto ipsec transform-set TECHCORP-TS esp-aes 256 esp-sha256-hmac
crypto ipsec security-association lifetime seconds 3600
```

### Interesting Traffic (ACL)

Defined from each router's own perspective (local subnet first, remote subnet second):

```
R1: access-list 100 permit ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
R2: access-list 100 permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255
```

### Crypto Map

```
crypto map TECHCORP-VPN 10 ipsec-isakmp
 set peer <remote-public-ip>
 set transform-set TECHCORP-TS
 match address 100
```

Applied to the WAN-facing interface on each router.

## Verification Commands

| Command | What it confirms |
|---|---|
| `show crypto isakmp sa` | Phase 1 status. State `QM_IDLE` means the secure negotiation channel succeeded. |
| `show crypto ipsec sa` | Phase 2 status. Non-zero `#pkts encaps` / `#pkts decaps` confirms real packets are being encrypted and decrypted. |
| `show crypto map` | Confirms the crypto map exists and is applied to the correct interface. |
| `show running-config \| section crypto` | Full crypto configuration for review. |

## Tests Performed

1. Verified basic IP connectivity between R1, R2, and R-ISP prior to enabling encryption.
2. Confirmed each site could reach its own default gateway but not the remote LAN prior to VPN configuration.
3. Brought up the IPsec tunnel and confirmed Phase 1 reached state `QM_IDLE` on both routers.
4. Confirmed Phase 2 packet counters incrementing on both routers during a ping test.
5. Successful ping from HQ-PC1 (192.168.2.11) to Branch-PC1 (192.168.1.1) and the reverse direction.
6. Packet capture on the R1-to-R-ISP transit link confirmed ESP-encapsulated packets rather than plaintext ICMP, verifying the traffic is genuinely encrypted in transit.

## Issues Encountered and Resolved

| Issue | Root Cause | Resolution |
|---|---|---|
| ISAKMP SA table empty, tunnel never triggered | Crypto ACL had source and destination subnets reversed on both routers relative to each router's actual LAN | Rebuilt ACL 100 on each router so the local subnet is listed first and the remote subnet second |
| `No pre-shared key with <peer-ip>` in ISAKMP debug | Typo in `crypto isakmp key` peer address (192.51.100.1 instead of 198.51.100.1) | Corrected the peer IP in the pre-shared key statement to match the actual remote WAN interface |

## Key Concepts

- **Phase 1 vs Phase 2**: Phase 1 (ISAKMP) establishes a secure channel used purely for negotiation. Phase 2 (IPsec) uses that channel to agree on how the actual data traffic will be encrypted and verified.
- **Interesting traffic**: The ACL bound to the crypto map determines which packets are encrypted. Traffic not matching the ACL is routed normally and is not protected by the VPN.
- **Crypto map**: The binding point that ties together the peer address, the transform-set, and the ACL, then applies all three to a physical interface.
- **ACL directionality**: Each router's crypto ACL must be written from that router's own point of view (its own subnet as source, the remote subnet as destination). Reversing this is one of the most common site-to-site VPN misconfigurations in practice.

## Screenshots

See the `screenshots/` folder for supporting evidence:

- `01-isakmp-sa.png` - Phase 1 SA state on R1 and R2
- `02-ipsec-sa.png` - Phase 2 SA with packet counters
- `03-ping-hq-to-branch.png` - Successful ping HQ to Branch
- `04-ping-branch-to-hq.png` - Successful ping Branch to HQ
- `05-wireshark-esp.png` - Packet capture showing ESP-encrypted traffic on the transit link
- `06-crypto-map.png` - Crypto map binding and applied interface
- `07-running-config-crypto.png` - Final crypto configuration on both routers
