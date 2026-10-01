# Segmented Enterprise Network with DMZ (Cisco Packet Tracer)

A three-zone enterprise network built from scratch in Cisco Packet Tracer: a public-facing DMZ, a trusted internal network split into three VLANs, and a Cisco ASA firewall enforcing trust boundaries between them.

![Topology overview](./screenshots/topology-overview.png)
<!-- Replace with your clean final topology screenshot -->

## Why this project

Most student Packet Tracer projects are a handful of PCs and a switch. The goal here was to build something closer to how a real small-to-medium business would actually segment and secure its network, and to document the reasoning behind every design decision, not just the finished diagram.

## Architecture

```
                         Internet
                              |
                    [Edge Router] (NAT, default route)
                              |
                    [ASA Firewall] — 3 zones:
                    Outside (0) | DMZ (50) | Inside (100)
                    /                              \
          [DMZ Switch]                      [Core L3 Switch]
           /        \                         /      |      \
     [Web Server] [Mail Server]          [VLAN 10]  [VLAN 20]  [VLAN 30]
                                           Finance      IT        Guest
```

| Zone | Security level | Contents |
|---|---|---|
| Outside | 0 | Internet-facing edge router link |
| DMZ | 50 | Public-facing web and mail servers |
| Inside | 100 | Finance, IT, and Guest VLANs |

Three security levels mean traffic can flow from high → low automatically (Inside can reach the DMZ), but low → high is denied by default unless explicitly permitted — the same trust model a real firewall deployment uses.

## IP addressing scheme

| Segment | Network | Notes |
|---|---|---|
| Router ↔ Firewall (outside) | 10.0.0.0/30 | Point-to-point |
| Firewall ↔ Core switch (inside) | 10.0.0.4/30 | Point-to-point, routed port on the L3 switch |
| DMZ | 172.16.10.0/24 | Web server .10, Mail server .20 |
| VLAN 10 — Finance | 192.168.10.0/24 | Gateway .1 |
| VLAN 20 — IT | 192.168.20.0/24 | Gateway .1 |
| VLAN 30 — Guest | 192.168.30.0/24 | Gateway .1 |

## What's implemented

- **Three-legged ASA firewall** with correctly assigned `nameif` and `security-level` per interface
- **Three internal VLANs** (Finance, IT, Guest), each on its own access switch, trunked to a Layer 3 core switch via 802.1Q
- **Inter-VLAN routing** via SVIs on the core switch, verified working between all three departments
- **DMZ hosting a web and mail server**, reachable from Inside, isolated from direct exposure
- **An ACL restricting the Guest VLAN** from reaching Finance or IT, while still allowing Guest to reach the DMZ and internet — tested in both directions (confirmed it blocks what it should, confirmed it doesn't break what it shouldn't)
- **NAT/PAT** so internal hosts can reach the internet without exposing private addressing, verified with a real translated ping and a correctly decremented TTL

## The hardest bug, and how it was found

Partway through, Inside-to-DMZ connectivity silently broke. Every standard check came back clean:

- PC's own gateway — reachable
- Trunk links and VLAN forwarding state on both switches — healthy
- ACL hit counters — incrementing, traffic was being permitted
- ARP table — fully resolved
- Spanning tree — nothing blocked
- Interface `nameif`/security-level config — correct in the running config

The actual cause: the firewall's DMZ-facing interface had silently reverted to an administratively shut-down state — invisible in the running config, and only exposed once a full device reload revealed a mismatch between the running and saved configuration. Re-enabling the interface and saving properly resolved it immediately.

The troubleshooting method (isolating hop by hop — PC → switch → firewall → DMZ — rather than guessing or rebuilding blind) mattered more than the specific fix.

## Verification performed

| Test | Result |
|---|---|
| Finance ↔ IT ↔ Guest (inter-VLAN routing) | ✅ All pairs successfully routed |
| Inside → DMZ web server | ✅ (after the interface bug was fixed) |
| Guest → Finance / IT | ✅ Correctly blocked by ACL |
| Guest → DMZ | ✅ Still allowed (ACL not overly broad) |
| Internal host → simulated internet (NAT/PAT) | ✅ Verified via ping with correct TTL decrement and translated source address |

## Tools

Cisco Packet Tracer (Cisco ASA 5506-X, 3560 L3 switch, 2960 access switches, 4321 ISR router)

## Next steps

This network is the target environment for a separate attack-and-detection exercise — see [`dvwa-sqli-attack`](../dvwa-sqli-attack) for the live exploitation and traffic analysis phase of this portfolio project.
