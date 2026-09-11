# Topic 3.1 — Interpreting the routing table

> **Exam objective 3.1** — Interpret the components of a routing table: route source /
> protocol code, administrative distance, metric, gateway of last resort, next hop, etc.

R1 is the "viewer." It peers with R5 over the `10.0.15.0/24` link using **four routing
protocols at once** — eBGP, EIGRP, IS‑IS and RIP — while also running OSPF to the rest of the
topology. R5 advertises a set of **phantom prefixes** through selected protocols so that R1's
table contains **overlapping prefixes** that can only be resolved with a solid grasp of
**Administrative Distance (AD)** and **longest‑prefix match (LPM)**.

These prefixes do **not** need to be reachable (per the spec) — they exist to be *read*.
They are **not** redistributed into OSPF, so the OSPF core stays clean.

## How each prefix is injected (the answer key)

| Prefix | Carried by (protocol : AD) | **Winner in R1's table** | Why |
|---|---|---|---|
| `10.30.0.0/16` | eBGP : 20 | **`B` (20)** | only BGP carries the /16 |
| `10.30.0.0/24` | EIGRP : 90 | **`D` (90)** | only EIGRP carries the /24 |
| `10.31.0.0/16` | IS‑IS : 115 | **`i L2` (115)** | only IS‑IS carries it |
| `10.32.0.0/16` | RIP : 120 | **`R` (120)** | only RIP carries it |
| `10.50.50.0/24` | eBGP 20 · EIGRP 90 · IS‑IS 115 · RIP 120 | **`B` (20)** | lowest AD wins (4‑way tie‑break) |
| `10.60.60.0/24` | EIGRP 90 · IS‑IS 115 · RIP 120 | **`D` (90)** | lowest AD wins (3‑way) |
| `10.61.0.0/16` | IS‑IS 115 · RIP 120 | **`i L2` (115)** | IS‑IS beats RIP |

Administrative Distance reference (Cisco defaults used here):

| Source | AD | Code |
|---|---|---|
| Connected | 0 | `C` |
| Static | 1 | `S` |
| eBGP | 20 | `B` |
| EIGRP (internal) | 90 | `D` |
| OSPF | 110 | `O` |
| IS‑IS | 115 | `i` |
| RIP | 120 | `R` |

> **Design note:** the phantom prefixes are advertised from R5 **loopbacks** with the target
> protocol enabled *on the interface* (not via redistribution). That keeps them **internal**
> to each protocol, so they arrive at R1 with the true textbook ADs (e.g. EIGRP **90**, not
> the 170 you would see for a redistributed/external EIGRP route).

---

## The routing table — `show ip route` on R1

```
R1# show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       ...
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ...

Gateway of last resort is 10.0.12.2 to network 0.0.0.0

O*E2  0.0.0.0/0 [110/1] via 10.0.12.2, 00:02:16, Ethernet0/1
      10.0.0.0/8 is variably subnetted, 18 subnets, 3 masks
C        10.0.0.1/32 is directly connected, Loopback0
O        10.0.0.2/32 [110/11] via 10.0.12.2, 00:02:16, Ethernet0/1
O        10.0.0.3/32 [110/21] via 10.0.12.2, 00:01:38, Ethernet0/1
O        10.0.0.4/32 [110/21] via 10.0.12.2, 00:01:38, Ethernet0/1
O        10.0.0.5/32 [110/11] via 10.0.15.5, 00:02:12, Ethernet0/2
C        10.0.12.0/24 is directly connected, Ethernet0/1
L        10.0.12.1/32 is directly connected, Ethernet0/1
C        10.0.15.0/24 is directly connected, Ethernet0/2
L        10.0.15.1/32 is directly connected, Ethernet0/2
O        10.0.34.0/24 [110/30] via 10.0.12.2, 00:01:38, Ethernet0/1
O        10.0.234.0/24 [110/20] via 10.0.12.2, 00:02:16, Ethernet0/1
B        10.30.0.0/16 [20/0] via 10.0.15.5, 00:01:22
D        10.30.0.0/24 [90/409600] via 10.0.15.5, 00:02:21, Ethernet0/2
i L2     10.31.0.0/16 [115/20] via 10.0.15.5, 00:02:18, Ethernet0/2
R        10.32.0.0/16 [120/1] via 10.0.15.5, 00:00:00, Ethernet0/2
B        10.50.50.0/24 [20/0] via 10.0.15.5, 00:01:22
D        10.60.60.0/24 [90/409600] via 10.0.15.5, 00:02:21, Ethernet0/2
i L2     10.61.0.0/16 [115/20] via 10.0.15.5, 00:02:18, Ethernet0/2
S     192.168.50.0/24 [1/0] via 10.0.12.2
      192.168.99.0/32 is subnetted, 1 subnets
S        192.168.99.99 [1/0] via 10.0.12.2
```

Every component of a routing‑table entry is visible here. Using `B  10.30.0.0/16 [20/0] via 10.0.15.5`:

* **`B`** — route source / protocol code (BGP).
* **`10.30.0.0/16`** — destination network and prefix length.
* **`[20/0]`** — **`[Administrative Distance / Metric]`**.
* **`via 10.0.15.5`** — next‑hop IP.
* **`Ethernet0/2`** (when shown) — outgoing interface.
* **`O*E2 0.0.0.0/0`** and the banner **"Gateway of last resort is 10.0.12.2"** — the default
  route learned from OSPF (R2 originates it with `default-information originate always`); the
  `*` flags it as the candidate default, `E2` = OSPF external type‑2.

### The two lessons, side by side

**Longest‑prefix match beats AD.** Both `B 10.30.0.0/16` and `D 10.30.0.0/24` are installed.
For a packet to `10.30.0.50`, the router does **not** compare AD — it picks the most specific
prefix first:

```
R1# show ip route 10.30.0.50
Routing entry for 10.30.0.0/24
  Known via "eigrp 100", distance 90, metric 409600, ... type internal
  * 10.0.15.5, from 10.0.15.5, via Ethernet0/2
```

→ the **/24 (EIGRP)** is chosen even though a BGP route with a *better* AD exists, because
`/24` is longer than `/16`. AD is only the tie‑breaker **when prefixes are identical**.

**AD decides among equal prefixes.** `10.50.50.0/24` is offered by four protocols; only the
eBGP copy (AD 20) is installed (`B`). `10.60.60.0/24` is offered by three; EIGRP (90) wins
(`D`). `10.61.0.0/16` is offered by two; IS‑IS (115) beats RIP (120) (`i L2`).

---

## Supporting evidence (optional, for the "prove it" crowd)

R1 really is running all five protocols:

```
R1# show ip protocols | include Routing Protocol
Routing Protocol is "ospf 1"
Routing Protocol is "isis"
Routing Protocol is "eigrp 100"
Routing Protocol is "rip"
Routing Protocol is "bgp 65001"
```

eBGP session up, 2 prefixes received:

```
R1# show bgp ipv4 unicast summary
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.15.5       4        65005       6       5        3    0    0 00:01:56        2
```

EIGRP neighbor up:

```
R1# show ip eigrp neighbors
H   Address                 Interface        Hold Uptime   SRTT   RTO  Q  Seq
0   10.0.15.5               Et0/2              12 00:02:02    1   100  0  3
```

---

## Student exercises

1. Before running any command, predict the installed route (protocol code + AD) for each of
   the seven `10.x` phantom prefixes using only the "answer key" table. Then verify.
2. Which entry proves that **longest‑prefix match outranks administrative distance**? What
   path does a packet to `10.30.0.50` take, and to `10.30.5.5`?
3. `10.50.50.0/24` is advertised by four protocols. Which one wins, and what would happen to
   the installed route if you shut BGP down? (Answer: it would fall to EIGRP → IS‑IS → RIP as
   each better source is removed.)
4. Find the gateway of last resort. Which router originates it, and what does the `*` and the
   `E2` tell you?
5. Contrast `show ip route` with `show ipv6 route` — why do the OSPFv3 entries list
   `FE80::…` next‑hops?
