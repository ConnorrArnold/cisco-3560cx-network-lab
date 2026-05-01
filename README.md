# Cisco 3560-CX Network Segmentation & Hardening Lab

> Homelab network hardening on a Cisco Catalyst 3560-CX. Configured VLAN segmentation, sticky MAC port security, BPDUGuard, and ACLs to isolate a vulnerability research zone from trusted devices.

---

## Overview

This lab documents the configuration of a Cisco Catalyst 3560-CX Series switch from a factory-wiped baseline to a hardened, segmented network environment. Every decision mirrors enterprise-grade network security practices used in production SOC and IT security environments.

The switch serves as the physical network backbone for a personal cybersecurity homelab, with isolated zones for trusted devices and intentionally vulnerable lab machines (DVWA, future VMs).

---

## Environment

| Item | Value |
|---|---|
| Hardware | Cisco Catalyst 3560-CX Series |
| IOS Mode | Rapid-PVST |
| Starting State | Factory wiped |
| Management IP | 192.168.99.1 |
| Console Access | PuTTY via COM3, 9600 baud |

---

## Network Design

### VLAN Architecture

| VLAN ID | Name | Subnet | Purpose |
|---|---|---|---|
| 10 | SECURE | 192.168.10.0/24 | Trusted devices, admin workstation |
| 20 | VULN-LAB | 192.168.20.0/24 | Isolated zone for DVWA and vulnerable VMs |
| 99 | MGMT | 192.168.99.0/24 | Switch management only |

**Design rationale:** VULN-LAB is intentionally isolated from SECURE so that a compromised lab machine cannot pivot to the trusted network. VLAN 1 (the Cisco default) is left empty and unused to eliminate VLAN hopping exposure.

### Port Assignment

| Ports | VLAN | Violation Action |
|---|---|---|
| Gi0/1 - Gi0/4 | SECURE (10) | Shutdown |
| Gi0/5 - Gi0/8 | VULN-LAB (20) | Restrict |
| Gi0/9 | MGMT (99) | Management access only |
| Gi0/10 - Gi0/12 | MGMT (99) + Shutdown | Unused, administratively disabled |

---

## Configuration

### 1. VLAN Segmentation

```
vlan 10
 name SECURE
vlan 20
 name VULN-LAB
vlan 99
 name MGMT
```

**Security reasoning:** VLAN 1 is the default on all Cisco switches and a known target for VLAN hopping attacks. Moving all traffic to named VLANs and leaving VLAN 1 empty eliminates default-tagged traffic that could be exploited.

![VLAN Brief](screenshots/03-vlan-brief.png)

---

### 2. Port Assignment

```
interface range Gi0/1 - 4
 switchport mode access
 switchport access vlan 10

interface range Gi0/5 - 8
 switchport mode access
 switchport access vlan 20

interface Gi0/9
 switchport mode access
 switchport access vlan 99

interface range Gi0/10 - 12
 switchport mode access
 switchport access vlan 99
 shutdown
```

**Security reasoning:** Unused ports are assigned to MGMT then administratively shut down. An attacker with physical access cannot plug into an active port and get uncontrolled network access.

![Ports Assigned](screenshots/04-ports-assigned.png)

---

### 3. Management SVI

```
interface vlan 99
 ip address 192.168.99.1 255.255.255.0
 no shutdown

ip default-gateway 192.168.99.254
```

**Security reasoning:** Management traffic is isolated to its own VLAN. Neither SECURE nor VULN-LAB devices can reach the management plane without explicit ACL permission. This is out-of-band management and is standard enterprise hardening practice.

![Management SVI](screenshots/05-mgmt-svi.png)

---

### 4. Port Security

**SECURE zone — Gi0/1-4 (max 1 MAC, Shutdown on violation):**
```
interface range Gi0/1 - 4
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
```

**VULN-LAB zone — Gi0/5-8 (max 2 MACs, Restrict on violation):**
```
interface range Gi0/5 - 8
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict
```

**Violation mode comparison:**

| Mode | Traffic Dropped | Port Shut Down | Violation Logged |
|---|---|---|---|
| Protect | Yes | No | No |
| Restrict | Yes | No | Yes |
| Shutdown | Yes | Yes | Yes |

**Security reasoning:** SECURE ports use Shutdown because any unauthorized device on a trusted port is treated as a critical incident. VULN-LAB ports use Restrict to accommodate a VM host plus guest VM (maximum 2 MACs) while still logging violations. Sticky MAC automatically learns and locks the first MAC address that connects, persisting to running config without manual entry.

![Port Security](screenshots/07-port-security.png)

---

### 5. STP Hardening

```
spanning-tree portfast default
spanning-tree portfast bpduguard default

interface range Gi0/1 - 8
 spanning-tree guard root
```

| Feature | Attack Prevented | Mechanism |
|---|---|---|
| PortFast | STP convergence delays on access ports | Skips listening and learning states |
| BPDUGuard | Rogue switch injection | Shuts port immediately on any BPDU received |
| Root Guard | Root bridge hijacking | Ignores superior BPDU announcements |

**Security reasoning:** An attacker who plugs a rogue switch into an access port can win the STP root bridge election and become the center of the network, enabling traffic interception. BPDUGuard shuts the port the moment any STP packet arrives, eliminating that attack vector entirely.

![STP Hardening](screenshots/08-stp-hardening.png)

---

### 6. ACLs — Inter-VLAN Traffic Control

```
ip access-list extended VULN-LAB-OUT
 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny ip 192.168.20.0 0.0.0.255 192.168.99.0 0.0.0.255
 permit ip 192.168.20.0 0.0.0.255 any

interface vlan 20
 ip access-group VULN-LAB-OUT in

ip routing
```

| Rule | Action | Source | Destination | Reason |
|---|---|---|---|---|
| 10 | Deny | VULN-LAB | SECURE | Compromised lab machine cannot reach trusted devices |
| 20 | Deny | VULN-LAB | MGMT | Lab machines cannot access switch management |
| 30 | Permit | VULN-LAB | Any | Internet access for updates and lab connectivity |

**Security reasoning:** The ACL is applied inbound on VLAN 20, filtering traffic as it leaves the VULN-LAB zone before it can be routed anywhere. Cisco ACLs include an implicit deny all at the bottom, so Rule 30 must exist or all VULN-LAB internet traffic would be silently blocked.

![ACL Config](screenshots/10-acl-vuln-lab.png)

---

## Final Verification

```bash
show vlan brief
show port-security
show spanning-tree summary
show ip access-lists
show ip interface brief
```

![Final Verification](screenshots/11-final-verification.png)

---

## Security Controls Summary

| Control | Implementation | Threat Mitigated |
|---|---|---|
| VLAN Segmentation | 3 named VLANs, VLAN 1 empty | VLAN hopping, lateral movement |
| Port Assignment | All ports off VLAN 1 | Default VLAN exploitation |
| Unused Port Shutdown | Gi0/10-12 administratively down | Unauthorized physical access |
| Management SVI | Isolated on VLAN 99 | Management plane exposure |
| Port Security | Sticky MAC, Shutdown/Restrict | Rogue device connection, MAC flooding |
| STP Hardening | PortFast, BPDUGuard, Root Guard | Rogue switch, root bridge hijacking |
| ACLs | VULN-LAB blocked from SECURE and MGMT | Lateral movement from compromised machine |

---

## Screenshot Index

| File | Contents |
|---|---|
| `01-initial-prompt.png` | First console connection |
| `02-hostname-set.png` | Hostname configured |
| `03-vlan-brief.png` | VLANs created and verified |
| `04-ports-assigned.png` | All ports moved off VLAN 1 |
| `05-mgmt-svi.png` | Management IP configured |
| `06-write-memory.png` | Config saved to NVRAM |
| `07-port-security.png` | Port security rules verified |
| `08-stp-hardening.png` | STP features enabled |
| `09-final-save.png` | Final save confirmation |
| `10-acl-vuln-lab.png` | ACL rules verified |
| `11-final-verification.png` | Full end-to-end proof |

---

## Key Commands

```bash
show vlan brief                              # Verify VLANs and port assignments
show port-security                           # Verify port security and violations
show spanning-tree summary                   # Verify STP hardening features
show ip access-lists                         # Verify ACL rules
show ip interface brief                      # Verify interface IPs and states
show running-config                          # View full running config
write memory                                 # Save config to NVRAM

# Re-enable a port shut down by port security
interface Gi0/X
 shutdown
 no shutdown
```

---

## Lessons Learned

- **VLAN 1 should always be empty.** Any traffic on the default VLAN is a liability. Moving all ports to named VLANs is a foundational hardening step.
- **Sticky MAC is practical for small environments.** It avoids manual MAC entry while enforcing port security. In enterprise environments, 802.1X is preferred for identity-based access control.
- **ACL implicit deny requires careful planning.** Forgetting the final permit statement silently blocks all traffic with no error message.
- **Write memory after every change.** The switch rebooted mid-lab. All config survived because of consistent saves.
- **BPDUGuard and PortFast are a pair.** PortFast alone speeds up convergence but leaves the port open to rogue switch attacks. BPDUGuard is what provides the actual protection.

---

## Related Projects

- [DVWA Vulnerability Lab](https://github.com/ConnorrArnold/aws-ec2-dvwa-vuln-lab) — The VULN-LAB VLAN was built specifically to isolate this lab's attack environment from trusted devices
