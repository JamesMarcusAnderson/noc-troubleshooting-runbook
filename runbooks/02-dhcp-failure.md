# Runbook 02 — DHCP Failure

Every client on the segment gets an APIPA address at once. Hosts with static
IPs are fine, links are up, and the VLAN is correct — the DHCP service itself
is the problem.

## Symptoms

- All DHCP clients on VLAN 10 receive `169.254.x.x` (APIPA).
- Hosts with manually configured static IPs on the same VLAN work normally.
- `ipconfig /renew` times out: "unable to contact your DHCP server."
- The scope of the failure (whole segment, not one host) points at the server,
  not a client or a single port.

## Diagnosis commands

**1. Confirm the scope — multiple clients affected (any client):**

```
ipconfig
ip addr show
```

APIPA on more than one host rules out a single bad port (see runbook 01).

**2. Attempt a manual renew and watch it fail:**

```
ipconfig /release
ipconfig /renew
```

Timeout / "unable to contact your DHCP server" confirms no DHCPOFFER arrives.

**3. Check the DHCP bindings on the router acting as DHCP server:**

```
show ip dhcp binding
```

Empty or stale. On a healthy server this lists leased addresses.

**4. Check the pool configuration:**

```
show ip dhcp pool
```

Verify the pool exists and the `network` statement matches the segment
(`192.168.10.0 /24`).

**5. Inspect the running config for the pool and the service itself:**

```
show running-config | section ip dhcp
```

Look for: pool missing entirely, `network` statement wrong (wrong subnet),
the whole range excluded (`ip dhcp excluded-address`), or `no service dhcp`.

**6. Check DHCP server statistics and conflicts:**

```
show ip dhcp server statistics
show ip dhcp conflict
```

Zero discovers received suggests the requests never reach the server process;
conflicts suggest an overlapping static assignment.

**7. Verify the gateway (sub)interface for the segment is up:**

```
show ip interface brief
```

If the gateway subinterface (R1 `Gi0/0.10`, R2 `Gi0/1.10`) is down,
DHCP relay/broadcast never reaches the server. (If it is down, work
runbook 05 first.)

## Root cause

The DHCP pool for VLAN 10 is misconfigured or the DHCP service is disabled —
e.g. the `network` statement was changed to the wrong subnet, the entire range
was excluded, or `no service dhcp` is in the running config. Client DHCP
discovers go unanswered and every client falls back to APIPA. Static-IP hosts
are unaffected because they never ask for a lease.

## Fix

On the router acting as DHCP server, rebuild the pool and enable the service:

```
configure terminal
service dhcp
ip dhcp pool VLAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8 8.8.4.4
 lease 7
exit
no ip dhcp excluded-address 192.168.10.1 192.168.10.10
end
```

Then on each client:

```
ipconfig /release
ipconfig /renew
```

## Verification

1. `show ip dhcp binding` — leases appear for the clients.
2. `ipconfig` on clients — `192.168.10.x` addresses, correct gateway and DNS.
3. `ping 192.168.10.1` — replies.
4. `show ip dhcp server statistics` — discovers and offers incrementing.

## How to break this in the lab

Pick one (run it on the router whose segment you want to break —
R1 serves PC-A, R2 serves PC-B):

```
configure terminal
no service dhcp
end
```

or delete the pool's network statement:

```
configure terminal
ip dhcp pool VLAN10
 no network 192.168.10.0 255.255.255.0
end
```

Then renew on that router's PC and watch it land on APIPA. Note: R1 and R2
run independent DHCP pools, so a break on R1 only affects PC-A's segment —
PC-B (served by R2's pool) is unaffected, and vice versa.
