# Lab 29 - Network Hardening

## Objective

Close the exposure created by earlier labs in this roadmap by hardening the network infrastructure itself. Where Labs 19 through 27 focused on detecting and responding to attacks, this lab focuses on removing the conditions that made those attacks possible in the first place, directly addressing several gaps identified in the Lab 28 incident response report.

## Topology

No new nodes. Hardening applied to existing infrastructure:

| Device | Role |
|---|---|
| R1 | HQ router |
| R2 | Branch router |
| SW-HQ / SW-Branch | Access switches (where IOS-capable) |

## Hardening Actions Applied

### 1. SSH-only remote management, Telnet removed

```
crypto key generate rsa modulus 2048
ip domain-name lab.local
username admin secret StrongPass2026!
line vty 0 4
 transport input ssh
 login local
```

Applied identically on R1 and R2. This replaces the earlier approach in Lab 24, which only logged and denied Telnet on a specific interface via ACL. Removing Telnet from the VTY transport list entirely means the protocol is refused outright, rather than relying on a firewall rule to catch it after the fact.

`secret` was used instead of `password` for the local account, since `secret` stores a hashed value in the configuration rather than a reversible plaintext or weakly-obfuscated one.

### 2. Password encryption for any remaining plaintext entries

```
service password-encryption
```

Applies Type 7 encoding to any legacy line or SNMP passwords still present in the configuration. This is a baseline improvement, not a complete fix: Type 7 encoding is well known to be trivially reversible and should not be relied upon as strong protection. It is included here as a minimum-floor hardening step while the plaintext services themselves are being removed.

### 3. SNMP migrated from v2c to v3

```
no snmp-server community TechCorp-RO ro
no snmp-server community TechCorp-RW rw
snmp-server group SECURE-GROUP v3 priv
snmp-server user secureadmin SECURE-GROUP v3 auth sha AuthPass2026! priv aes 128 PrivPass2026!
```

This directly resolves the SNMPv2c exposure first flagged as a known limitation back in Lab 17 and carried forward as an open item in the Lab 28 incident report. SNMPv2c community strings are sent unencrypted and function as plaintext passwords for device management; SNMPv3 adds real authentication (SHA) and encryption (AES) to every exchange.

### 4. Unused and unnecessary services disabled

```
no ip http server
no ip http secure-server
no cdp run
no ip source-route
no service pad
```

Each of these reduces attack surface on the router itself:

- HTTP/HTTPS management removed in favor of SSH-only access, consistent with item 1.
- CDP disabled globally, since it broadcasts device model, IOS version, and IP addressing information to anything on the local segment, which is useful for legitimate troubleshooting but equally useful for attacker reconnaissance.
- IP source routing disabled, removing a legacy mechanism that can be used to bypass expected routing and ACL behavior.
- The X.25 PAD service disabled as a standard baseline hardening item with no legitimate use in this environment.

### 5. Legal warning banner

```
banner motd #
UNAUTHORIZED ACCESS TO THIS DEVICE IS PROHIBITED.
All activity is monitored and logged. Disconnect immediately
if you are not an authorized administrator.
#
```

Beyond deterrence, a clearly stated access banner has documented legal relevance in several jurisdictions' unauthorized-access statutes, which often consider whether a system clearly communicated that access was restricted.

### 6. VTY access restricted by source address

```
access-list 10 permit 192.168.1.0 0.0.0.255
line vty 0 4
 access-class 10 in
```

Limits which source addresses can even attempt to open a management session on R1, independent of whether valid credentials are presented. Combined with SSH-only transport, this is a layered control: a single failure (for example, a leaked credential) does not by itself grant management access from an arbitrary source.

### 7. Port security on access switches

```
interface range fastEthernet0/1 - 24
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
```

Limits the number of MAC addresses permitted on a single access port. This is directly relevant to the ARP spoofing activity carried out in Lab 21, where an attacker's ability to inject traffic under multiple MAC identities on a single segment was central to the attack. Port security does not prevent ARP spoofing on its own, but restricts one of the conditions that makes it easier to sustain undetected at scale.

## Verification

```
show ip ssh
telnet 192.168.1.1
```
Telnet connections are now refused outright rather than accepted and separately logged as denied, as they were prior to this lab.

```
ssh admin@192.168.1.1
```
Confirms SSH access succeeds with the newly configured local credential.

```
show snmp user
```
Confirms the SNMPv3 user is active and no v2c community strings remain.

## Key Learning

- This lab closes the loop on the roadmap's attacker-side labs. The Telnet exposure demonstrated in Lab 20, the plaintext SNMP configuration from Lab 17, and the generally permissive reachability that made Labs 19 through 27 possible are directly remediated here rather than only detected after the fact.
- Hardening is layered by design. SSH-only transport, the VTY access list, and port security each address a different failure mode independently, so that a single control failing does not by itself expose the device.
- Hardening has real operational tradeoffs that are worth stating rather than glossing over. Disabling CDP removes a legitimate troubleshooting tool, and port security can lock out a legitimate device replacement if not planned for. A hardening exercise that only lists what was disabled, without acknowledging what that costs operationally, is incomplete.
- Type 7 password encoding is included here as a floor, not a solution. It was applied because it costs nothing and improves on cleartext, but it should not be presented or relied upon as strong protection.

## Screenshots

See the `screenshots/` folder:

- `01-ssh-keys-generated.png` - RSA key generation and SSH transport configuration
- `02-telnet-refused-after-hardening.png` - Telnet connection refused outright, contrasted against the Lab 24 deny-and-log behavior
- `03-ssh-login-successful.png` - Successful SSH login with the new local credential
- `04-snmpv3-configured.png` - SNMPv3 user and group configuration, v2c communities removed
- `05-unused-services-disabled.png` - HTTP server, CDP, source routing, and PAD service disabled
- `06-banner-displayed.png` - Legal warning banner shown at login
- `07-vty-acl-restricting-access.png` - Access list applied to VTY lines
- `08-port-security-configured.png` - Port security applied on access switch interfaces
