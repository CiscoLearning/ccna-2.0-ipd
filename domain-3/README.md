# CCNA 2.0 — Domain 3.0 (IP Routing) — CML Demo

A single Cisco Modeling Labs (**CML‑Free compatible**) topology that demonstrates the
hands‑on portions of **Domain 3.0 (IP Routing)** of the CCNA 2.0 (200‑301 v2.0) blueprint.
Built for the Cisco Networking Academy **Instructor Professional Development** week.

Everything in this repository has been **built and validated on live IOL‑XE nodes** — the
`show`‑command output quoted throughout `docs/` is real captured output from this topology,
not hand‑written examples.

| Exam topic | What this lab demonstrates | Where |
|---|---|---|
| **3.1** Interpret the components of a routing table | R1's table is deliberately "clogged" with overlapping prefixes learned via **eBGP / EIGRP / IS‑IS / RIP / OSPF**, so you must use **Administrative Distance** and **longest‑prefix match** to predict the installed route | [docs/02](docs/02-routing-table-interpretation.md) |
| **3.2** Configure and verify IPv4 & IPv6 static routing | Network, host, default and **floating** static routes (IPv4 + IPv6). The IPv4 floating static contains a **deliberate error** | [docs/03](docs/03-static-and-floating-routes.md) |
| **3.3** Configure and verify single‑area OSPFv2 **and** OSPFv3 | Full‑mesh reachability, **point‑to‑point** links **and** a **broadcast** segment with a forced DR/BDR/DROTHER election, RIDs `0.0.0.x` | [docs/01](docs/01-ospf-validation.md) |
| **3.4** Describe the purpose / verify operation of first‑hop redundancy | **HSRPv2** — active/standby, virtual IP, priority, preempt, and failover forcing functions | [docs/04](docs/04-hsrp-fhrp.md) |
| (Design) Out‑of‑band management | A **MGMT VRF** keeps the global table clean; all devices reach an external workstation via `bridge0` | [docs/05](docs/05-mgmt-vrf-and-external.md) |

---

## Block diagram

```
                 EIGRP / RIP / IS-IS / eBGP injection
                 (phantom prefixes -> topic 3.1 table)
                              |
   +----------+  10.0.15.0/24 +----------+  10.0.12.0/24 +----------+
   |    R5    |===============|    R1    |===============|    R2    |
   | 0.0.0.5  |   OSPF p2p    | 0.0.0.1  |   OSPF p2p    | 0.0.0.2  |
   |injector  | Et0/1   Et0/2 | (viewer) | Et0/1   Et0/1 |core/DR   |
   +----------+               +----------+               +----+-----+
                                                          Et0/2 |
                                                                | 10.0.234.0/24
                                                                | OSPF broadcast
                                                                | (DR = R2, pri 255)
                                                           [ SW-BCAST ]
                                                            /          \
                                                     Et0/1 /            \ Et0/1
                                                +----------+        +----------+
                                                |    R3    |        |    R4    |
                                                | 0.0.0.3  |        | 0.0.0.4  |
                                                | OSPF BDR |        | DROTHER  |
                                                | HSRP act |        | HSRP sby |
                                                +----+-----+        +-----+----+
                                                Et0/2 |                   | Et0/2
                                                       \  10.0.34.0/24   /
                                                        [   SW-HSRP    ]
                                                     HSRP group 34, VIP 10.0.34.254
                                                     R3 active (110/preempt) · R4 standby (100)

   Out-of-band management plane  —  VRF "MGMT", 198.18.128.0/18:
   R1..R5 Et0/0 --> [ SW-MGMT ] --> ext-bridge0 (System Bridge / bridge0) --> workstation 198.18.133.252
```

`SW-BCAST`, `SW-HSRP`, `SW-MGMT` are **unmanaged switches** and `ext-bridge0` is an
**external connector** — none of these count toward the CML‑Free 5‑node limit. The five
active nodes are the IOL‑XE routers **R1–R5**.

---

## Addressing plan

Addressing is intentionally human‑readable:

* **Loopbacks** are carved from a single subnet — last octet = router number.
* **Every link uses a /24** (even point‑to‑point). The **3rd octet names the peers**
  (`.12` = R1↔R2, `.15` = R1↔R5, `.234` = R2/R3/R4, `.34` = R3/R4).
* **IPv6** mirrors IPv4: `2001:db8:<pair>::/64`, host id = router number.
* **Router IDs** are `0.0.0.<router#>` for **both** OSPFv2 and OSPFv3.

### Loopbacks & Router IDs

| Router | Loopback0 (IPv4) | Loopback0 (IPv6) | OSPF RID |
|:------:|:-----------------|:-----------------|:--------:|
| R1 | `10.0.0.1/32` | `2001:db8::1/128` | `0.0.0.1` |
| R2 | `10.0.0.2/32` | `2001:db8::2/128` | `0.0.0.2` |
| R3 | `10.0.0.3/32` | `2001:db8::3/128` | `0.0.0.3` |
| R4 | `10.0.0.4/32` | `2001:db8::4/128` | `0.0.0.4` |
| R5 | `10.0.0.5/32` | `2001:db8::5/128` | `0.0.0.5` |

### Links & segments

| Segment | IPv4 | IPv6 | OSPF network type | Addresses |
|---|---|---|---|---|
| R1 ↔ R2 (core) | `10.0.12.0/24` | `2001:db8:12::/64` | point‑to‑point | R1 `.1`, R2 `.2` |
| R1 ↔ R5 (edge) | `10.0.15.0/24` | `2001:db8:15::/64` | point‑to‑point | R1 `.1`, R5 `.5` |
| R2/R3/R4 (broadcast) | `10.0.234.0/24` | `2001:db8:234::/64` | broadcast | R2 `.2` **DR** (pri 255), R3 `.3` **BDR** (pri 100), R4 `.4` **DROTHER** (pri 0) |
| R3/R4 (HSRP LAN) | `10.0.34.0/24` | `2001:db8:34::/64` | passive (advertised only) | R3 `.3`, R4 `.4`, **VIP `.254`** |

### Management (VRF `MGMT`, out of band)

| Router | MGMT IP (Et0/0) | Default gateway | External target |
|:------:|:----------------|:---------------:|:----------------|
| R1 | `198.18.180.1/18` | `198.18.128.1` | workstation `198.18.133.252` |
| R2 | `198.18.180.2/18` | `198.18.128.1` | |
| R3 | `198.18.180.3/18` | `198.18.128.1` | |
| R4 | `198.18.180.4/18` | `198.18.128.1` | |
| R5 | `198.18.180.5/18` | `198.18.128.1` | |

The management interface is the **first usable interface (Et0/0)** on every router and lives
in VRF `MGMT`, so management traffic never pollutes the global routing table you interpret in
topic 3.1.

---

## Getting started

### Option A — import the ready‑made topology (recommended)

1. In the CML UI: **Import → Topology**, choose [`ccna-2.0-domain-3.0.yaml`](ccna-2.0-domain-3.0.yaml).
2. **Start** the lab and wait for convergence (~1 minute; the OSPF broadcast segment needs
   the ~40 s DR wait timer to complete).
3. Jump to any `docs/` page and follow the validation steps.

> The external connector is set to **System Bridge (`bridge0`)**. External MGMT reachability
> (topic in [docs/05](docs/05-mgmt-vrf-and-external.md)) depends on `bridge0` being connected
> to the `198.18.128.0/18` network on your CML host. OSPF/HSRP/routing‑table topics all work
> with **no external connectivity at all**.

### Option B — build by hand

The per‑device configurations live in [`config/`](config/) (`R1.cfg` … `R5.cfg`). Wire the
five routers per the block diagram, then paste each file into the matching router.

### Files in this repo

```
├── README.md                     ← you are here
├── ccna-2.0-domain-3.0.yaml       ← importable CML topology (all configs embedded)
├── config/                        ← per-device IOS configs (R1..R5)
├── docs/
│   ├── 01-ospf-validation.md          (topic 3.3 — OSPFv2 + OSPFv3)
│   ├── 02-routing-table-interpretation.md (topic 3.1 — AD & longest-prefix match)
│   ├── 03-static-and-floating-routes.md   (topic 3.2 — static + the broken floating static)
│   ├── 04-hsrp-fhrp.md                    (topic 3.4 — HSRP status & forcing functions)
│   └── 05-mgmt-vrf-and-external.md         (MGMT VRF + external reachability)
└── PLAN.md                        ← original functional specification
```

---

## Validation summary (as tested)

| Check | Result |
|---|---|
| OSPFv2 adjacencies (p2p R1‑R2, R1‑R5) | `FULL` |
| OSPFv2 broadcast segment | R2 = **DR**, R3 = **BDR**, R4 = **DROTHER** ✔ |
| OSPFv3 (IPv6) adjacencies & DR/BDR | identical to IPv4 ✔ |
| Full IPv4 **and** IPv6 reachability (loopback‑to‑loopback, any pair) | 100 % |
| R1 routing table shows overlapping prefixes at ADs **20/90/115/120** + a /16‑vs‑/24 LPM pair | ✔ |
| HSRP group 34 | R3 **Active** (pri 110, preempt), R4 **Standby** (pri 100), VIP `10.0.34.254` reachable ✔ |
| Floating static under normal state | correctly **dormant** (OSPF AD 110 < static AD 200) ✔ |
| Floating static after OSPF peering drop | **fails** (recursive next‑hop unresolvable) — the intended teaching point ✔ |
| MGMT VRF → workstation `198.18.133.252` from **all 5 routers** | reachable (80 % — first packet lost to ARP) ✔ |

See each `docs/` page for the full captured `show`‑command output and the step‑by‑step
forcing functions instructors can run live.

---

## Design guardrails honored

* **CML‑Free**: 5 active IOL‑XE nodes; unmanaged switches / external connector are free.
* **/24 on every link** (even p2p), **/64 IPv6**, loopbacks from one subnet, RIDs `0.0.0.x`.
* **Mix of OSPF p2p and broadcast**; DR/BDR forced via **priority** only.
* **No OSPF hello/dead‑timer changes and no authentication** (out of scope for CCNA).
* The multi‑protocol prefixes (3.1) are **injected only toward R1** and are **not**
  redistributed into OSPF, so core reachability stays clean and predictable.
