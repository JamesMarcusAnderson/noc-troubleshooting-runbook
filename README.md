# NOC Troubleshooting Runbook

A tier-1 NOC technician triages the same five failures on repeat: a host on the
wrong VLAN, DHCP handing out nothing, DNS resolving nothing, a route pointing
nowhere, and an interface that is simply down. This repo is the playbook for all
five — symptoms, the exact diagnosis commands, root cause, fix, and verification
— written the way a NOC actually works a ticket: methodically, from the edge in.

Each scenario is designed to be built and broken on purpose in a virtual lab
(GNS3, EVE-NG, or Cisco Packet Tracer), so you can practice the full
detect → diagnose → fix → verify loop without touching anything real.

## The five scenarios

| # | Runbook | One-line symptom |
|---|---------|------------------|
| 1 | [Bad VLAN assignment](runbooks/01-bad-vlan-assignment.md) | Host gets an APIPA address; everyone else on the switch is fine |
| 2 | [DHCP failure](runbooks/02-dhcp-failure.md) | Every client on the segment gets 169.254.x.x; static-IP hosts fine |
| 3 | [DNS outage](runbooks/03-dns-outage.md) | `ping 8.8.8.8` works, `ping google.com` fails |
| 4 | [Routing problem](runbooks/04-routing-problem.md) | Local subnet reachable, remote subnet dead; traceroute dies at the router |
| 5 | [Down interface](runbooks/05-down-interface.md) | Total loss to a segment; link light off or interface administratively down |

## How to use this with a virtual lab

1. Build the lab in [topology.md](topology.md) — two routers, two switches,
   two PCs. It supports all five scenarios with no rewiring.
2. Pick a scenario and break it on purpose using the "how to break it" notes
   in `topology.md`.
3. Work the ticket: open [templates/incident-ticket.md](templates/incident-ticket.md),
   follow the matching runbook's diagnosis commands in order, and don't peek at
   the root cause until your commands have proven it.
4. Fix it, verify it, close the ticket.

Every runbook follows the same structure:

**Symptoms → Diagnosis commands → Root cause → Fix → Verification**

## Scope

All scenarios are simulated in a virtual lab for training purposes. No
production systems are involved, and no real incident data is used. The
commands target Cisco IOS-style CLIs (GNS3/EVE-NG with IOS images, or Packet
Tracer's simulated IOS) and standard Windows/Linux host tooling.

## Keywords

network troubleshooting, incident response, NOC runbook, VLAN, DHCP, DNS,
routing, static routes, GNS3, EVE-NG, Packet Tracer, tier-1 NOC, network
operations, show commands, ping, traceroute, nslookup, dig
