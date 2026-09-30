# Runbook 04 — Routing Problem

The local subnet is fine but a remote subnet is unreachable. Traceroute dies
at a router. The link is up — the router simply doesn't know where to send the
traffic.

## Symptoms

- PC-A (`192.168.10.10`) can ping its gateway (`192.168.10.1`) and other local
  hosts.
- PC-A cannot ping PC-B (`192.168.20.10`) on the far side of the routers.
- `tracert` / `traceroute` to `192.168.20.10` shows hops up to R1 (`10.0.0.1`)
  and then timeouts — the trail goes cold at the router.
- Interface status on both routers shows up/up; this is not runbook 05.

## Diagnosis commands

**1. Confirm local connectivity is healthy (PC-A):**

```
ping 192.168.10.1
ping 192.168.10.11
```

Both reply — the local segment, VLAN, DHCP, and gateway are all fine.

**2. Confirm the remote failure and find where the trail dies:**

```
tracert 192.168.20.10
traceroute 192.168.20.10
```

Hops: PC-A → `192.168.10.1` (R1) → `* * *`. Dying at R1 means R1 has no route
(or a bad route) toward `192.168.20.0/24`.

**3. Inspect R1's routing table:**

```
show ip route
```

Look for `192.168.20.0/24`. Missing entirely, or present with the wrong
next-hop, is the fault.

**4. Check the longest match for the destination:**

```
show ip route 192.168.20.10
```

If the best match is the default route (or nothing), R1 is forwarding blind.

**5. Prove the WAN link itself is healthy (R1 CLI):**

```
ping 10.0.0.2
show ip interface brief
```

R2's WAN address replies and the interface is up/up — the physical and data
link layers are fine. The problem is strictly the routing table.

**6. Check the return path on R2 (don't forget it — routing is two-way):**

```
show ip route 192.168.10.10
```

R2 also needs a route back to `192.168.10.0/24` via `10.0.0.1`, or replies die
on the way home even after you fix R1.

## Root cause

R1's static route to the remote LAN is missing or points at the wrong
next-hop (e.g. `ip route 192.168.20.0 255.255.255.0 10.0.0.99` — a next-hop
that doesn't exist). Packets arrive at R1 and are dropped: no route, no
forwarding. The WAN link is up, so Layer 1/2 are innocent.

## Fix

On R1, remove the bad route and install the correct one:

```
configure terminal
no ip route 192.168.20.0 255.255.255.0 10.0.0.99
ip route 192.168.20.0 255.255.255.0 10.0.0.2
end
```

Verify the return path on R2 while you're in there:

```
show ip route 192.168.10.10
```

If R2's route back is also wrong, fix it symmetrically:

```
configure terminal
ip route 192.168.10.0 255.255.255.0 10.0.0.1
end
```

## Verification

1. `show ip route` on R1 — `S 192.168.20.0/24 [1/0] via 10.0.0.2` present.
2. `show ip route` on R2 — return route to `192.168.10.0/24` via `10.0.0.1`.
3. `traceroute 192.168.20.10` from PC-A — completes: `192.168.10.1`,
   `10.0.0.2`, `192.168.20.10`.
4. `ping 192.168.20.10` — replies (both directions; test PC-B → PC-A too).

## How to break this in the lab

On R1, point the remote route at a next-hop that doesn't exist:

```
configure terminal
no ip route 192.168.20.0 255.255.255.0 10.0.0.2
ip route 192.168.20.0 255.255.255.0 10.0.0.99
end
```

Traceroute from PC-A will now die at R1 while `ping 10.0.0.2` from R1 still
works — the signature of a routing-table fault, not a link fault.
