# pfSense settings used in this build

Everything below is set through the firewall's own web interface. Values are the ones used here;
substitute your own addresses.

---

## Interfaces

| Interface | Device | Role | Address |
| --- | --- | --- | --- |
| WAN | `em0` | Outer, faces the shared segment | `10.0.2.30/24`, gateway `10.0.2.1` |
| LAN | `em1` | Inner, faces the protected segment | `192.168.10.1/24` |

Automatic assignment is served on the inner interface for `192.168.10.100` to `192.168.10.200`.

**The outer address must be fixed.** The manager accepts system log messages only from an address
named explicitly in its configuration. If this address moves after a reboot, forwarding stops and
nothing announces it — the alerts simply cease, and the natural place to look is the decoder, which
is the wrong place entirely.

**An address is not a route.** Setting the address is not the same as selecting the upstream gateway
on the interface. With the gateway unselected, the firewall's own console reaches everything and
nothing behind the firewall reaches anything, because the console test only exercises a path to a
directly connected neighbour.

---

## Firewall rules

Rules live on the interface where a packet **enters** the firewall. Traffic from the endpoint towards
the internet enters on the inner interface, so that is where a rule denying it has to sit. A rule on
the outer interface is evaluated against traffic arriving from outside and will never match outbound
traffic from within.

Rules are evaluated top to bottom and stop at the first match, so block rules must sit **above** the
default rule that permits everything from the local network outward.

| Interface | Action | Source | Destination | Logging |
| --- | --- | --- | --- | --- |
| LAN | Block | `192.168.10.100` | `Blocked_IP_YouTube` (alias) | Enabled |
| LAN | Block | `192.168.10.100` | `Blocked_IP_Instagram` (alias) | Enabled |
| LAN | Block | `192.168.10.100` | any, protocol ICMP | Enabled |

**Logging must be enabled on every rule you want to detect on.** A rule that blocks without logging
protects the network and tells the monitoring platform nothing.

**Reading the counters.** The states column shows `0/<bytes>`. The leading zero is permanent and is
not a fault: a blocked packet never establishes a connection state, so there is nothing to count.
The byte figure is what says the rule is matching. `0/0 B` is the real failure signal and is exactly
what these rules show when placed on the wrong interface.

---

## Aliases

Created under Firewall → Aliases → IP, one host alias per service. Naming an address first makes the
rule read as intent rather than as a number, and when the address changes there is one place to
change it.

Both services here answer from large and constantly changing address pools, so resolve your own
rather than reusing these:

```bash
nslookup youtube.com
nslookup instagram.com
```

---

## Remote logging

Status → System Logs → Settings → Remote Logging.

| Setting | Value | Why |
| --- | --- | --- |
| Source address | WAN interface | The sending address must match `<allowed-ips>` on the manager |
| Message format | **BSD (RFC 3164)** | The newer structured format is not what the manager's pre-decoding stage expects; messages arrive and are then discarded in silence |
| Remote server | `10.0.2.20:514` | The manager's listener |
| Categories | **System Events** and **Firewall Events** only | The rules act on firewall messages; every other category dilutes the event stream and buries what matters |

Message format is the most common cause of a transport that is provably working and produces no
alerts at all.

---

## pfBlockerNG

Installed through System → Package Manager.

**Geographic filtering.** Select individual countries, not whole regions. This laboratory sits in
Pakistan; denying Asia as a region would have denied the country the laboratory itself runs in, and
the first symptom would have been the whole environment losing connectivity with the cause several
menus away. Countries selected here: China and India within Asia, France and Germany within Europe.
The resulting alias tables held 13,251 and 29,878 entries.

A geographic database account and licence key are needed. **Treat the key as a credential** — it is
obscured in every figure in the report for that reason.

**Settings take effect on a reload run, not on save.** A group can be saved correctly and do nothing
at all until a reload has completed, which makes it very easy to conclude that a correct
configuration is broken and start changing it.

**Domain filtering works**, and both blocked names resolve to the sinkhole address. A virtual address
of `10.10.10.1/32` was created on the loopback interface for the package's sinkhole web server, and a
block group created with the resolver-based action. Two things had to be cleared first, and both fail
without an error:

* **The reload.** Covered above. Until a reload run completes, the group does nothing.
* **The DNSBL whitelist.** It lives further down the DNSBL settings page, below the group
  configuration, and it is easy to miss entirely. It held `youtube.com` and `www.youtube.com` but not
  `instagram.com`, which is exactly why one name sinkholed and the other did not. **A whitelist entry
  overrides a block group with no error and no indication next to the group it contradicts.** Check it
  before you doubt the mechanism. The safe-search page states that enabling that feature
  wildcard-whitelists the sites it covers, which is one way entries arrive there.

Verify from the endpoint with a control in the same session, so a broken resolver cannot be mistaken
for a working block:

```bash
nslookup youtube.com        # expect 10.10.10.1
nslookup instagram.com      # expect 10.10.10.1
nslookup example.com        # expect real addresses
```

A limit remains even when this is configured correctly: a client can resolve names over its own
encrypted connection to a public resolver and never ask the firewall at all. The package can answer
*no such domain* for the published DNS-over-HTTPS endpoints, under DNSBL → SafeSearch, which narrows
that gap without closing it. It was left disabled in this build.
