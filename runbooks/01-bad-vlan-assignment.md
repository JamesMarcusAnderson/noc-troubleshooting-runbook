# Runbook 01 — Bad VLAN Assignment

A single host can't get on the network while every other host on the same
switch works fine. The classic cause: the switchport is in the wrong VLAN.

## Symptoms

- One PC gets an APIPA address (`169.254.x.x`) instead of a DHCP lease.
- `ping <gateway>` fails from the affected PC only.
- Other PCs on the same switch get DHCP and reach the gateway normally.
- Link light is on; the cable is good. This is not a physical problem.

## Diagnosis commands

Work from the host toward the switch. Do not skip steps.

**1. Confirm the host has no valid address (Windows / Linux):**

```
ipconfig
ip addr show
```

Look for `169.254.x.x` (APIPA) — proof the DHCP discover never got an answer.

**2. Confirm local gateway is unreachable:**

```
ping 192.168.10.1
```

Request timed out. Expected — the host isn't really on VLAN 10's subnet.

**3. Check which VLAN the port is actually in (switch CLI):**

```
show vlan brief
```

Find the PC's port (e.g. `Fa0/5`). If it sits in VLAN 20 (voice) instead of
VLAN 10 (data), you have your answer.

**4. Confirm the port's operational switchport config:**

```
show interfaces fa0/5 switchport
```

Check `Operational Mode: static access` and `Access Mode VLAN: 20` — wrong.

**5. Check whether the port learned the PC's MAC in the wrong VLAN:**

```
show mac address-table interface fa0/5
```

A MAC learned in VLAN 20 confirms the port membership, not a trunking issue.

**6. Rule out a trunk problem (sanity check):**

```
show interfaces trunk
```

The uplink trunk should carry VLAN 10. If it does, the fault is the access
port, not the trunk.

## Root cause

The access port for the PC was assigned to VLAN 20 (the voice VLAN) instead of
VLAN 10 (the data VLAN). The DHCP pool only serves VLAN 10, so the client's
DHCP discovers are broadcast into a VLAN with no DHCP server and no gateway —
the client falls back to APIPA.

## Fix

On the switch, move the port back to the data VLAN:

```
configure terminal
interface fa0/5
 switchport access vlan 10
 shutdown
 no shutdown
end
```

On the PC, force a fresh DHCP attempt:

```
ipconfig /release
ipconfig /renew
```

(Linux: `dhclient -r && dhclient`, or unplug/replug the virtual NIC.)

## Verification

1. `show vlan brief` — `Fa0/5` now listed under VLAN 10.
2. `ipconfig` on the PC — address is `192.168.10.x`, not `169.254.x.x`.
3. `ping 192.168.10.1` — replies.
4. `ping 8.8.8.8` — replies (full path works).

## How to break this in the lab

```
configure terminal
interface fa0/5
 switchport access vlan 20
end
```

Then `ipconfig /release` + `ipconfig /renew` on PC-A and watch it land on APIPA.
