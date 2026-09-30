# Runbook 03 — DNS Outage

Everything works by IP address, but nothing resolves by name. This is the
single most misdiagnosed ticket in a NOC — always prove it is DNS before you
touch anything else.

## Symptoms

- `ping 8.8.8.8` works; `ping google.com` fails with "could not find host" /
  "temporary failure in name resolution."
- Web browsers show DNS errors while IP-based connectivity is fine.
- Affects every client using the same DNS server; clients pointed at a
  different resolver are fine.

## Diagnosis commands

**1. Prove it is DNS and not connectivity (the golden test):**

```
ping 8.8.8.8
ping google.com
```

IP works, name fails → DNS. If the IP ping also fails, this is not a DNS
ticket — work runbooks 04/05 instead.

**2. See the failure at the resolver level:**

```
nslookup google.com
```

Expect a timeout or "server failed." Note which server it queried.

**3. Check what DNS servers the client is configured to use:**

```
ipconfig /all
cat /etc/resolv.conf
```

Confirm the client is pointed at the expected DNS server (in this lab,
the DHCP-advertised resolver, `8.8.8.8`).

**4. Test the same query against a known-good public resolver:**

```
nslookup google.com 8.8.8.8
dig @8.8.8.8 google.com
```

If this resolves, the client's configured DNS server is the problem — not the
client, not the path.

**5. Query the suspect server directly:**

```
dig @192.0.2.53 google.com
nslookup google.com 192.0.2.53
```

(`192.0.2.53` is the dead TEST-NET-1 resolver from the lab break below —
substitute whatever suspect IP step 3 showed you.) Timeout here isolates the
fault to that server (or the path to it).

**6. Check whether something in the path is blocking DNS (router CLI):**

```
show access-lists
show running-config | include access-group
```

Look for an ACL denying UDP/TCP port 53 applied inbound on the client's
interface. DNS needs both UDP 53 (queries) and TCP 53 (large/zone transfers).

**7. If the DNS server is self-hosted in the lab, check the service:**

```
systemctl status named
systemctl status bind9
```

Dead service = dead resolver.

## Root cause

One of three: (a) clients are pointed at a DNS server IP that is down or
wrong (stale DHCP `dns-server` option); (b) an ACL on the path is blocking
UDP/TCP port 53 to the resolver; (c) the DNS service itself is stopped. In
each case name resolution fails while raw IP connectivity is untouched — which
is exactly why the IP-vs-name test in step 1 is the whole diagnosis.

## Fix

Match the fix to the cause you proved:

- **Wrong/dead resolver:** fix the DHCP pool and renew clients:
  ```
  configure terminal
  ip dhcp pool VLAN10
   dns-server 8.8.8.8 8.8.4.4
  end
  ```
  then `ipconfig /release` + `ipconfig /renew` on clients.
- **ACL blocking port 53:** remove or correct the access-list entry, e.g.:
  ```
  configure terminal
  ip access-list extended CLIENTS-IN
   no deny udp any any eq 53
   permit udp any any eq 53
  end
  ```
- **Dead service:** `systemctl restart named` (or `bind9`).

## Verification

1. `nslookup google.com` — resolves, using the correct server.
2. `ping google.com` — replies.
3. `dig google.com` — returns A records with sane TTLs.
4. Repeat from a second client to confirm it wasn't a one-host cache.

## How to break this in the lab

Point the lab DNS at a dead IP via DHCP:

```
configure terminal
ip dhcp pool VLAN10
 dns-server 192.0.2.53
end
```

(`192.0.2.0/24` is TEST-NET-1 — guaranteed unroutable.) Renew clients, then
`ping 8.8.8.8` works while `ping google.com` fails.
