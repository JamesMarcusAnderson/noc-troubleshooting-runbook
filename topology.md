# Lab Topology

One topology that supports all five runbooks with no rewiring. Build it in
GNS3, EVE-NG, or Cisco Packet Tracer (use IOS/IOSvL2 images where the platform
needs them; Packet Tracer's simulated devices work as-is).

## Diagram

```
                    10.0.0.0/30 (WAN)
   PC-A ---- SW1 ---- R1 ================= R2 ---- SW2 ---- PC-B
  .10       |  .1     .1                   .2     |  .1       .10
            |                                    |
       VLAN 10: 192.168.10.0/24            VLAN 10: 192.168.20.0/24
       VLAN 20: voice (unused)             (single data VLAN)
```

## Device inventory

| Device | Role | Interfaces | IP addressing |
|--------|------|------------|---------------|
| PC-A | Client (DHCP) | NIC → SW1 Fa0/5 | DHCP from R1 (`192.168.10.10`) |
| SW1 | Access switch | Fa0/5 → PC-A; Gi0/1 trunk → R1 | VLAN 10 data, VLAN 20 voice |
| R1 | Router + DHCP server | Gi0/0 → SW1; Gi0/1 → R2 WAN | `192.168.10.1` (LAN), `10.0.0.1/30` (WAN) |
| R2 | Router | Gi0/0 → R1 WAN; Gi0/1 → SW2 | `10.0.0.2/30` (WAN), `192.168.20.1` (LAN) |
| SW2 | Access switch | Gi0/1 trunk → R2; Fa0/5 → PC-B | VLAN 10 only |
| PC-B | Client (DHCP) | NIC → SW2 Fa0/5 | DHCP from R2 (`192.168.20.10`) |

## Baseline configuration notes

- **R1** runs the DHCP pool `VLAN10` (`192.168.10.0/24`, gateway
  `192.168.10.1`, DNS `8.8.8.8`); **R2** runs pool `VLAN10-R2` for
  `192.168.20.0/24`.
- **SW1**: `Fa0/5` = access VLAN 10; `Gi0/1` = trunk allowing VLANs 10, 20.
  **SW2**: `Fa0/5` = access VLAN 10; `Gi0/1` = trunk.
- **Static routes:** R1: `ip route 192.168.20.0 255.255.255.0 10.0.0.2`;
  R2: `ip route 192.168.10.0 255.255.255.0 10.0.0.1`.
- Verify the baseline before breaking anything: PC-A → PC-B ping works,
  both PCs hold DHCP leases, `traceroute` shows the full path.

## How to break each scenario (quick reference)

| Runbook | Break it | Device |
|---------|----------|--------|
| 01 Bad VLAN | `interface fa0/5` → `switchport access vlan 20` | SW1 |
| 02 DHCP failure | `no service dhcp` (or delete the pool's `network` statement) | R1 |
| 03 DNS outage | DHCP `dns-server 192.0.2.53` (TEST-NET-1, unroutable) | R1 |
| 04 Routing problem | `ip route 192.168.20.0 255.255.255.0 10.0.0.99` | R1 |
| 05 Down interface | `interface gi0/1` → `shutdown` | R1 (WAN) or SW1 (access) |

Full break/fix/verify procedures live in each runbook under `runbooks/`.

## Restoring the lab

After each scenario, reverse the break step (each runbook's Fix section does
this), renew DHCP on both PCs, and re-run the baseline ping test before
starting the next scenario. Never stack two breaks — one fault at a time is
how a NOC works a ticket.
