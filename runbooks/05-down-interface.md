# Runbook 05 — Down Interface

Total loss to a segment. No ping, no DHCP, nothing. The interface is down at
Layer 1 or administratively shut — check the port before you check anything
else.

## Symptoms

- Every host past the interface is unreachable; `ping` fails with timeouts
  (not "destination unreachable" — there is simply no path).
- Monitoring (if present) shows the interface down; the link light is off or
  the port shows `administratively down`.
- Unlike runbook 04, the router on the near side can't even ping the far side
  of the link — the failure is at or below the interface, not in the routing
  table.

## Diagnosis commands

**1. Check interface status summary (router/switch CLI):**

```
show ip interface brief
```

Read the Status and Protocol columns:
- `administratively down / down` → someone (or something) shut the port.
- `down / down` → physical problem: cable, SFP, or the far end is down.
- `up / down` → Layer 1 good, Layer 2 problem (encapsulation/keepalive).

**2. Get detail on the suspect interface:**

```
show interfaces gi0/1
```

Check: `line protocol`, input/output error counters, and the tell-tale
`administratively down` string. A climbing error count points at cabling.

**3. On a switch, check port state at a glance:**

```
show interfaces status
```

Look for `disabled`, `err-disabled`, or `notconnect` on the port in question.

**4. Check the logs for when it went down:**

```
show logging | include UPDOWN
```

`%LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to down` with a
timestamp tells you when — correlate with change windows and tickets.

**5. Rule out errdisable (switch ports):**

```
show interfaces status err-disabled
```

A port in `err-disabled` was shut by the switch itself — usually port-security
violation, BPDU guard, or a duplex/speed storm. The cause must be cleared
before the port will stay up.

**6. Physical check (real gear; in the lab, check the virtual link):**

- Cable seated at both ends? SFP seated? Far-end device powered?
- In GNS3/EVE-NG: is the link actually drawn between the nodes, and are both
  node interfaces started?

## Root cause

Either (a) the interface was administratively shut down (`shutdown` in the
config — the most common lab/accidental cause), (b) a physical failure
(unplugged cable, bad SFP, far-end device down), or (c) the switch err-disabled
the port (port-security/BPDU guard). `show ip interface brief` distinguishes
(a) from (b) in one glance.

## Fix

Match the fix to the cause:

- **Administratively down:**
  ```
  configure terminal
  interface gi0/1
   no shutdown
  end
  ```
- **Err-disabled:** fix the underlying cause first (e.g. remove the offending
  device, correct the port-security MAC), then bounce the port:
  ```
  configure terminal
  interface fa0/5
   shutdown
   no shutdown
  end
  ```
- **Physical:** reseat/replace cable or SFP; power on the far-end device. In
  the lab, verify the virtual link exists and both nodes are running.

## Verification

1. `show ip interface brief` — `up / up` on the interface.
2. `show interfaces gi0/1` — `line protocol is up`, no rising error counters.
3. `show logging | include UPDOWN` — `changed state to up` logged.
4. End-to-end `ping` across the link, then the full path (PC-A to PC-B).
5. Confirm it stays up for a few minutes — a flapping port (up/down cycling
   in the logs) means the physical fault isn't actually fixed.

## How to break this in the lab

```
configure terminal
interface gi0/1
 shutdown
end
```

Pick the R1–R2 WAN link to kill the whole remote side, or a switch access
port to isolate one PC. `show ip interface brief` will show
`administratively down` — the signature to look for.
