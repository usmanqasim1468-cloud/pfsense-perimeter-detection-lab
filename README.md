# 🧱 pfSense Perimeter Defence Home Lab

### `Firewall Deployment • Syslog Ingestion • Custom Decoder • Correlation Rules • MITRE ATT&CK`

<p align="center">
  <img src="https://img.shields.io/badge/pfSense-2.7.2-212121?style=for-the-badge&logo=pfsense&logoColor=white" />
  <img src="https://img.shields.io/badge/Wazuh-4.14.7-005792?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Ubuntu-26.04%20LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" />
  <img src="https://img.shields.io/badge/Kali%20Linux-2026.2-000000?style=for-the-badge&logo=kalilinux&logoColor=white" />
  <img src="https://img.shields.io/badge/VirtualBox-7.2-183A61?style=for-the-badge&logo=virtualbox&logoColor=white" />
  <img src="https://img.shields.io/badge/MITRE%20ATT%26CK-T1595-6E2C8B?style=for-the-badge" />
</p>

<p align="center">
  <b>A firewall put in the traffic path, its logs forwarded into a SIEM that could not read a single
  one of them, and the decoder written to fix that — so a dropped packet becomes a classified,
  queryable security event.</b>
</p>

---

## ⚡ Overview

An earlier lab in this series built a Wazuh server watching a single endpoint. It ended at a
boundary I could see clearly: a port scan run against the server produced no alert at all, because a
host-based platform reads logs that machines write and a stealth scan never causes a service to
write one. Closing that gap means putting a device in the traffic path and feeding what it records
into the platform.

This project does that. A pfSense firewall becomes the perimeter, the monitored endpoint moves
behind it, traffic is blocked at the boundary, and the firewall's logs are forwarded into Wazuh.

**Blocked packet → syslog → decoder → rule → correlation → alert → dashboard**

The interesting part turned out not to be the firewall. **Wazuh ships a decoder for pfSense, and it
could not match a single line of what arrived — with nothing misconfigured on either side.**
Diagnosing that, and then writing a decoder that identifies the message by its structure rather than
by the field that was missing, is the core of the work.

---

## 🎯 What This Lab Demonstrates

| Area | Implementation |
| --- | --- |
| 🧱 Perimeter Design | Two-interface firewall bridging a shared segment and an isolated internal one |
| 🌐 Network Segmentation | Endpoint moved behind the firewall with no path around it |
| 🚦 Traffic Filtering | Host aliases, rule placement by ingress interface, verified with controls |
| 🌍 Geographic Filtering | Address-to-country lists applied to named countries rather than whole regions |
| 📡 Log Transport | Firewall logs forwarded over the system logging protocol, alongside agent traffic |
| 🔍 Failure Diagnosis | A shipped decoder that silently matches nothing, traced to an absent syslog field |
| 🧩 Detection Engineering | Custom decoder anchored on message structure, not on program name |
| 🔗 Correlation | Repetition from one source collapsed into a single higher-severity alert |
| 🎯 Threat Framework | Correlation rule mapped to MITRE ATT&CK technique T1595, Reconnaissance |
| 🧪 Offline Testing | Full pipeline validated with `wazuh-logtest` before generating any traffic |
| 📊 Presentation | Saved search, two visualisations and a combined dashboard |
| 🧯 Failure Analysis | Eight build failures documented with symptom, cause, fix and lesson |
| ✅ Honest Verification | Every claim tested and tabulated, including the two that did not hold |

---

# 🏗️ Lab Architecture

```text
                        Windows host  ·  Oracle VirtualBox 7.2

  ══════════════════════════════════════════════════════════════════════════
   NAT Network  "NatNetwork"  ·  10.0.2.0/24  ·  gateway 10.0.2.1
  ══════════════════════════════════════════════════════════════════════════
       │                                                 │
  ┌────┴─────────────────────┐                 ┌─────────┴─────────────────┐
  │  ubuntu   ·  10.0.2.20   │                 │  pfsense  ·  2048 MB      │
  │  6144 MB                 │◀────── 514 ─────│  em0  10.0.2.30      WAN  │
  │                          │     syslog      │  em1  192.168.10.1   LAN  │
  │  Wazuh Manager           │                 └─────────┬─────────────────┘
  │  Wazuh Indexer           │                           │
  │  Wazuh Dashboard         │                           │
  └──────────────────────────┘                           │
                                                         │
  ══════════════════════════════════════════════════════════════════════════
   Internal Network  "LabLAN"  ·  192.168.10.0/24
  ══════════════════════════════════════════════════════════════════════════
                                                         │
                                               ┌─────────┴─────────────────┐
                                               │  kali  ·  192.168.10.100  │
                                               │  4096 MB                  │
                                               │                           │
                                               │  Wazuh Agent 4.14.7       │
                                               │  Traffic generator        │
                                               └───────────────────────────┘
```

> **Why two networks and not one?**
> A firewall can only act on traffic that passes through it. While the endpoint shares a segment
> with the firewall's outer interface, its packets go straight to the hypervisor's router and never
> reach pfSense at all. Every rule would read correctly, the counters would stay at zero, and the
> connections being blocked would succeed. The internal network is guest-to-guest only, so the
> single route out is through the firewall — and that is the property everything else depends on.

---

# 🔬 The Log Pipeline

```text
   192.168.10.100  →  packet dropped at the boundary
             │
             ▼
  ┌──────────────────────┐
  │  pfSense filterlog   │  one comma separated line per decision
  │  remote logging      │  BSD (RFC 3164) · System + Firewall categories only
  └──────────┬───────────┘
             │  system logging protocol · UDP 514 · allowed-ips pinned
             ▼
  ┌──────────────────────┐
  │  PRE-DECODER         │  ⚠ NO HOSTNAME FIELD  →  program name displaced
  │                      │     shipped pfSense decoder cannot match
  └──────────┬───────────┘
             ▼
  ┌──────────────────────┐
  │  CUSTOM DECODER      │  pfsense-filterlog   anchored on  ",match,"
  │                      │  extracts → action · direction · protocol
  │                      │             srcip · dstip · srcport · dstport
  └──────────┬───────────┘
             ▼
  ┌──────────────────────┐
  │  RULES               │  100900  level 5   every blocked packet
  │                      │  100901  level 3   permitted traffic
  │                      │  100902  level 10  repetition  ·  T1595
  └──────────┬───────────┘
             ▼
  ┌──────────────────────┐
  │  INDEXER             │  wazuh-alerts-*
  └──────────┬───────────┘
             ▼
  ┌──────────────────────┐
  │  DASHBOARD           │  saved search → 2 visualisations → dashboard
  └──────────────────────┘
```

**A decoder answers what the log says, field by field. A rule answers whether that matters.** Both
are required, and they have to agree exactly about field names. A fault in any single stage above
produces the identical symptom at the dashboard: nothing appears.

---

# 🧰 Technology Stack

**Perimeter**

* **pfSense CE 2.7.2** — firewall and router, two interfaces, FreeBSD based
* **pfBlockerNG** — address-to-country lists and domain filtering
* **BSD syslog (RFC 3164)** — transport from firewall to monitoring platform

**Monitoring platform**

* **Wazuh 4.14.7** — manager, indexer and dashboard on a single host
* **OpenSearch** — indexing and search layer beneath the dashboard
* **MITRE ATT&CK** — classification vocabulary for the correlation rule

**Endpoint and activity generation**

* **Kali Linux 2026.2** — monitored endpoint behind the firewall
* `ping`, `curl`, `nslookup` — activity generators and controls

**Infrastructure**

* **Oracle VirtualBox 7.2** — hypervisor
* **Ubuntu 26.04 LTS** — server operating system
* **tcpdump** — proving the transport before debugging the decoder

---

# 🧪 Lab Modules

## 01 — Building the Perimeter

A third virtual machine with **two** adapters: the first on the shared NAT Network becoming the
outer interface, the second on the internal network `LabLAN` becoming the inner one. Both must exist
before the installer runs, because the interface assignment step can only assign interfaces that are
present.

Two traps in the console:

* At the local interface prompt the text reads `(em1 a or nothing if finished)`. Pressing Enter to
  accept what looks like a default is read as **nothing**, and the console answers that you have
  chosen to remove the local interface. The blank Enter belongs at the *Optional* prompt, one step
  later.
* **An address is not a route.** Setting the outer address is not the same as selecting the upstream
  gateway on it. With the gateway unselected the firewall's own console reaches everything, and
  nothing behind the firewall reaches anything — because the console test only exercises a path to a
  directly connected neighbour and proves nothing about the path that is broken.

## 02 — Moving the Endpoint Behind the Firewall

The only structural change this project makes to the earlier environment. Everything else is
additive, so it is worth treating as a **regression check** rather than a configuration step: the
question is not whether the endpoint gets an address, it is whether everything built earlier still
works from its new position.

```bash
ip -brief address show          # address, now 192.168.10.100
ip route                        # default via 192.168.10.1 — the firewall
cat /etc/resolv.conf            # resolver, also the firewall
ping -c 3 10.0.2.20             # manager still reachable, through the firewall
```

The agent reports active with **no configuration change at all**, and that is the point rather than
a happy accident. Every mechanism in the earlier build runs outward from the agent to the manager.
Outbound mechanisms survive a change of network position; inbound ones do not. Had the manager been
the party reaching out, this move would have broken the entire earlier build.

## 03 — Blocking by Alias, and Where Rules Actually Live

Addresses are named as aliases first, so the rule reads as intent rather than as a number and there
is one place to change it when the address moves.

> ⚠️ **pfSense evaluates rules on the interface where a packet ENTERS the firewall.** Traffic from
> the endpoint towards the internet enters on the **inner** interface, so that is where a denying
> rule has to sit. A rule on the outer interface is evaluated against traffic arriving from outside
> and will never match outbound traffic from within. The rule reads exactly as intended, the alias
> resolves, and the traffic passes anyway.

Rules must also sit **above** the default permit rule, because evaluation stops at the first match.

**Verification needs a control.** A rule that blocks everything also blocks the two target
addresses, so the test has to prove both halves at once:

```bash
ping -c 4 142.251.153.4          # blocked   → 100% loss
ping -c 4 157.240.227.174        # blocked   → 100% loss
ping -c 3 8.8.8.8                # control   → answers normally
curl -I --connect-timeout 5 http://example.com   # control → HTTP/1.1 200 OK
```

**Reading the counters.** The states column shows `0/672 B`. The leading zero is permanent and is
not a fault — a blocked packet never establishes a connection state, so there is nothing to count.
The byte figure is what says the rule is matching. `0/0 B` is the real failure signal, and it is
exactly what these rules show when placed on the wrong interface.

## 04 — Geographic and Domain Filtering

Address filtering blocks a number; geographic filtering applies the same idea to tens of thousands
of them at once. **Named countries, not whole regions** — this laboratory sits in Pakistan, and
denying Asia as a region would have denied the country the laboratory itself runs in, with the first
symptom being total loss of connectivity and the cause several menus away from where anyone would
start looking. China, India, France and Germany were selected instead, producing alias tables of
13,251 and 29,878 entries.

**Domain filtering did not work, and is reported as unresolved rather than quietly omitted.** Name
lookups from the endpoint continued to return the real public addresses for both listed domains,
while an unrelated control domain behaved normally in the same session. Two known causes produce
exactly that symptom and neither could be confirmed or excluded — a sinkhole address missing from
the loopback interface, or the safe-search option, which works by resolving a domain to a *filtered*
address and therefore guarantees the name is answered.

There is a deeper limit that applies even when this is configured perfectly: a browser can resolve
names over its own encrypted connection to a public resolver, bypassing the firewall's resolver
completely. **Domain filtering at a perimeter is a control that works on cooperative clients rather
than a boundary that holds.**

## 05 — Forwarding Firewall Logs

A second listener is added to the manager **after** the existing agent listener, never in place of
it — replacing it disconnects every enrolled endpoint, and the symptom would be the endpoint falling
silent at the same moment firewall logging was switched on, which invites entirely the wrong
diagnosis.

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>10.0.2.30</allowed-ips>
  <local_ip>10.0.2.20</local_ip>
</remote>
```

Two settings on the firewall decide whether any of this works:

| Setting | Value | Consequence of getting it wrong |
| --- | --- | --- |
| Message format | **BSD (RFC 3164)** | The newer structured format is not what the pre-decoding stage expects. Messages arrive and are then discarded in silence — the most common cause of a transport that provably works and produces no alerts |
| Categories | **System + Firewall only** | Every other category dilutes the event stream, buries what matters, and makes the dashboard harder to read rather than richer |

**Prove the messages physically arrive before touching a decoder.** The alternative is debugging a
decoder against messages that were never delivered.

```bash
sudo ss -ulnp | grep :514                       # is the listener even bound?
sudo tcpdump -n -i enp0s3 -A udp port 514       # are messages arriving?
```

## 06 — A Message the Platform Could Not Read

**This is the centrepiece.** Wazuh ships a pfSense decoder. The device is common enough that support
is built in, and the reasonable expectation was that these messages would be decoded and classified
with no further work.

They were not decoded at all.

```text
**Phase 1: Completed pre-decoding.
        timestamp: 'Aug 29 12:22:52'
        hostname:  'filterlog[84779]:'        ← the PROGRAM NAME, in the hostname field

**Phase 2: Completed decoding.
        No decoder matched.
```

A BSD syslog message is expected to carry, in order, a priority marker, a timestamp, a **hostname**,
a program name, and then the body. The pre-decoding stage reads those fields **positionally**.

pfSense sent no hostname. The field is simply absent, so the message runs straight from the
timestamp to the program name — and read positionally, the pre-decoder recorded `filterlog[84779]:`
as the *hostname* and found no program name at all.

**That single displacement is enough to defeat the decoder**, because the shipped decoder identifies
these messages **by program name**, and the program name is now sitting in a different field under a
different label.

> Nothing was broken. The firewall was sending in the format it was asked to send in. The manager
> was receiving. The decoder was present, loaded, and the right decoder for the device. Two products
> simply disagreed about whether an optional field is optional. Setting the hostname explicitly,
> re-saving the logging configuration and rebooting the firewall all left the field absent.

## 07 — Writing a Decoder That Matches on Structure

If a message cannot be identified by a field that is missing, identify it by something that is
present. Every filterlog line contains the literal `,match,` and nothing else the firewall sends
does, so that string is a fingerprint that survives the sender's choices about optional fields.

```xml
<decoder name="pfsense-filterlog">
  <prematch>,match,</prematch>
</decoder>

<decoder name="pfsense-filterlog-ports">
  <parent>pfsense-filterlog</parent>
  <regex>,(\S+),match,(\S+),(\S+),4,\S*,\S*,\S*,\S*,\S*,\S*,\S*,(\S*),\S*,(\S*),(\S*),(\S*),(\S*),</regex>
  <order>interface,action,direction,protocol,srcip,dstip,srcport,dstport</order>
</decoder>
```

Two child decoders are needed because not every message carries ports — TCP and UDP lines end with a
source and destination port, ICMP lines end with a type and a sequence number. The ports variant is
listed **first**, because decoding stops at the first child that matches.

**It took four attempts, and the three failures are more instructive than the result:**

| # | Approach | Outcome |
| --- | --- | --- |
| 1 | Anchor the pre-match at the start of the line, count fields from the beginning | No match at all — the pre-decoder has already consumed part of the line |
| 2 | Reduce the pre-match to the literal `,match,` | Parent matched, **no fields extracted** |
| 3 | Add an offset so child patterns begin after the parent match | No change whatsoever |
| 4 | Re-anchor the child patterns on `,(\S+),match,` | ✅ Fields extracted correctly |

> ⚠️ **The same pattern trap as the earlier lab, in a new place.** Wazuh's default expression engine
> is not PCRE. A plain full stop is a **literal dot**, `\.` means any character, and square bracket
> character classes are **not supported at all** and silently match nothing. Counting comma
> separated fields with `\S*,` and extracting with `(\S+)` sidesteps the question entirely, and it
> is the idiom the platform's own shipped decoders use.

## 08 — Detection Rules and Correlation

| ID | Level | Matches | Mapping |
| --- | --- | --- | --- |
| `100900` | 5 | Any decoded firewall message — source, destination, protocol, direction | — |
| `100901` | 3 | Permitted traffic, lower severity because it is context not incident | — |
| `100902` | 10 | Repeated matches of `100900` from the **same source** within 60 seconds | **T1595** Active Scanning |

Rule `100902` is the one that earns its place. One blocked packet is noise; repeated blocked packets
from one source inside a minute is a pattern, and describing the pattern once is more useful to an
analyst than describing each packet eight times.

```xml
<rule id="100902" level="10" frequency="8" timeframe="60">
  <if_matched_sid>100900</if_matched_sid>
  <same_source_ip />
  <description>pfSense: repeated blocked packets from $(srcip) within one minute.</description>
  <mitre>
    <id>T1595</id>
  </mitre>
  <group>firewall_block,recon,</group>
</rule>
```

> ⚠️ **A rule file that loads is not a rule that fires.** My first version tested the decoded
> `action` field explicitly. `action` is a **reserved static field** in the rule language, and the
> manager refuses to start outright:
>
> ```text
> wazuh-analysisd: ERROR: Failure to read rule 100900. Field 'action' is static.
> wazuh-analysisd: CRITICAL: (1220): Error loading the rules
> ```
>
> Always parse the ruleset before restarting. It is the difference between a five-second correction
> and a manager that will not start, taking agent log processing down with it for a reason buried in
> a log file rather than shown on screen:
>
> ```bash
> sudo /var/ossec/bin/wazuh-analysisd -t
> ```

**Where the alert says it came from.** These messages arrive with **no agent behind them**, so
alerts are recorded against the *manager*, not against the endpoint that generated the traffic.
Filtering a dashboard by the endpoint's agent name returns nothing at all, for events the endpoint
caused — and an analyst who starts from the machine they suspect will conclude the detection is
broken. Filter on `rule.groups` or `data.srcip` instead. **An alert belongs to whatever produced the
log, not to whatever caused the activity.**

## 09 — Visualising the Firewall Activity

Alerts in a table answer what happened. A dashboard answers whether the pattern is changing, which
is the question an analyst actually asks during a shift.

* **Saved search** — filter `rule.groups:pfsense` stored as a reusable object
* **Firewall Block Trend** — count over a date histogram. Answers *when*
* **Blocked Destinations by Volume** — count grouped by `data.dstip`. Answers *where*

---

# 📊 Validation Results

| # | Capability | How it was tested | Result |
| --- | --- | --- | --- |
| 1 | Endpoint behind the firewall, no path around it | Address, route and resolver read after the move | ✅ All three point at the firewall |
| 2 | Earlier monitoring build survives the move | Agent status on the manager, dashboard from the endpoint | ✅ Active, zero configuration change |
| 3 | Named addresses blocked from the endpoint | Both attempted, two unrelated destinations as controls | ✅ 100% loss, controls unaffected |
| 4 | Denials recorded against the causing rule | Firewall log inspected after the test | ✅ Rule description attached |
| 5 | Rules matching rather than merely present | Byte counters read on both rules | ✅ Counters advanced from zero |
| 6 | Firewall messages reach the manager | `tcpdump` on the manager's interface, port 514 | ✅ Arriving, one line each |
| 7 | Messages decoded into named fields | Captured line replayed through `wazuh-logtest` | ✅ All fields extracted |
| 8 | Blocked packets raise classified alerts | Offline validation, then live traffic | ✅ Rule 100900, level 5 |
| 9 | Repetition correlated into one alert | Burst of 16 requests from one source | ✅ 14 alerts + 2 correlation alerts |
| 10 | Geographic filtering denies selected countries | Alias tables and generated rules inspected | ⚠️ Lists built and rules generated; block not separately verified from the endpoint |
| 11 | Permitted traffic raises a distinguishable alert | Searched for rule 100901 after permitted traffic | ❌ Rule is dead — reserved field name, see below |
| 12 | Domain filtering sinkholes listed domains | Name lookups from the endpoint | ❌ Real addresses still returned |

**The three results that are not a tick mark are the most useful in the set**, and all three are
explained in full in the report rather than being dropped from it. A build described only by its
successes is not a description of a build.

---

# 🧯 Problems Encountered and How They Were Resolved

| # | Symptom | Cause | Fix |
| --- | --- | --- | --- |
| 1 | Endpoint receives no address after being moved | Stale fixed resolver configuration from earlier work | Cleared it and requested configuration again |
| 2 | Firewall console reaches everything, nothing behind it reaches anything | Address set, but no upstream gateway **selected**. A route is a separate object from an address | Selected the gateway on the outer interface |
| 3 | Rule reads correctly, traffic passes, counters stay at zero | Rule sits on the interface the traffic does not enter | Moved both rules to the ingress interface |
| 4 | Messages arrive at the manager, no alerts appear | Message format set to the newer structured standard | Set the format back to BSD (RFC 3164) |
| 5 | `No decoder matched` for a correctly formatted message | Firewall sends no hostname, displacing the program name the shipped decoder identifies on | Wrote a decoder anchored on message structure |
| 6 | Decoder matches but extracts no fields | Child patterns counted from the start of a string the pre-decoder had already consumed | Re-anchored the children on the identifying literal |
| 7 | Manager refuses to start after a rule change | `action` is a reserved static field, rejected as a general field test | Parsed the ruleset offline first, removed the condition |
| 8 | Validation passes, alerts are written, dashboard is empty | A filter left pinned on the search, and two machines five hours apart in their time zones | Cleared the filter, set both machines to the same time zone |

---

# 🧠 Key Takeaways

```text
A MISSING FIELD CAN DEFEAT A WORKING DECODER
        │
        ├── Nothing has to be misconfigured for an integration to fail completely
        ├── Two products can each be correct and still disagree about optional fields
        │
        ├── Identify a message by what is PRESENT, not by what SHOULD be
        ├── Rules are evaluated where packets ENTER — the byte counter is the diagnostic
        ├── An address is not a route, and the two failures look identical from inside
        ├── Validate offline before generating traffic — two seconds against one hour
        ├── An alert belongs to whatever produced the log, not what caused the activity
        ├── Correlation is what the SIEM adds: fifteen denials become one sentence
        ├── Name what you mean at a boundary — a region is not a country
        └── Report what did not work, or it is not a description of a build
```

---

# 🚧 Scope and Next Steps

**Deliberately out of scope**

* Packet inspection — the firewall reports its own *verdicts*, not what the traffic looks like
* Active response — every detection here reports; none of them act
* Distributed deployment — all three central components on one host, correct for a lab and wrong
  for production
* High availability — a single firewall with no failover partner

**Clearest next extensions**

* **Give rule 100901 a real discriminator.** It is dead because the obvious field name is reserved.
  Matching on decoded direction, or a tighter child decoder that only ever claims denials, separates
  the two cases properly
* **Measure the correlation threshold** with controlled bursts of known size, turning an honest gap
  into a stated fact
* **Fix the two decoder field offsets** — cosmetic today, and both would matter the moment somebody
  searched on the interface field
* **Add a network sensor** such as Suricata, so the design sees traffic patterns rather than only
  the firewall's verdicts

---

# 📂 Repository Structure

```text
pfsense-perimeter-detection-lab/
│
├── README.md
├── LICENSE
├── .gitignore
│
├── docs/
│   └── Perimeter-Defence-Home-Lab.pdf       # full 44-page report, 36 labelled figures
│
├── detections/
│   ├── pfsense_custom_decoders.xml          # parses filterlog by structure, not program name
│   └── pfsense_custom_rules.xml             # 100900 / 100901 / 100902, mapped to T1595
│
├── config/
│   ├── ossec-syslog-listener.conf           # the <remote> block added to the manager
│   └── pfsense-settings.md                  # interfaces, rules, remote logging, pfBlockerNG
│
└── samples/
    └── filterlog-samples.log                # real messages for wazuh-logtest, no firewall needed
```

---

# 🚀 Quick Start

Reproduce the detection half without building the firewall. The sample messages are real lines as
received, so the decoder and rules can be exercised end to end on a Wazuh manager alone.

```bash
# 1 — deploy the detection content
sudo cp detections/pfsense_custom_decoders.xml /var/ossec/etc/decoders/
sudo cp detections/pfsense_custom_rules.xml    /var/ossec/etc/rules/
sudo chown wazuh:wazuh /var/ossec/etc/decoders/pfsense_custom_decoders.xml
sudo chown wazuh:wazuh /var/ossec/etc/rules/pfsense_custom_rules.xml
sudo chmod 660 /var/ossec/etc/rules/pfsense_custom_rules.xml

# 2 — parse the ruleset BEFORE restarting anything
sudo /var/ossec/bin/wazuh-analysisd -t

# 3 — reload
sudo systemctl restart wazuh-manager

# 4 — paste a line from samples/filterlog-samples.log
sudo /var/ossec/bin/wazuh-logtest
```

Expect three phases to complete, the decoder `pfsense-filterlog` to extract fields, and rule
`100900` to fire at level 5.

A decoder file with the wrong ownership is ignored **in silence**, which from the validation tool is
indistinguishable from a decoder that does not match — so do not skip step 1's `chown`.

---

# 📚 Full Documentation

The complete report in **`docs/`** runs to 44 pages with 36 labelled figures across four parts:
design and build, filtering at the perimeter, teaching the platform to read the firewall, and
analysis. It contains every configuration value used, every command in the order it was run, the
reasoning behind each fix rather than only the final answer, a verification table covering every
claim including the ones that failed, and a command reference appendix so the build can be repeated
without rereading the document.

Credentials, device identifiers and machine identifiers are obscured in every figure.

---

# 👨‍💻 Author

### Muhammad Usman

**Cybersecurity Student | Security Operations | Blue Team | Detection Engineering**

---

<p align="center">

### 🧱 Segment → Filter → Forward → Decode → Correlate → Visualise

**⭐ Star this repository if the decoder that shipped with your SIEM ever matched nothing.**

</p>
