# Topic 3.3 — Single‑area OSPFv2 and OSPFv3

> **Exam objective 3.3** — Configure and verify single‑area OSPFv2 (and, in v2.0, OSPFv3),
> including neighbor adjacencies, point‑to‑point and broadcast network types, the DR/BDR
> election, and the router ID.

This lab runs **OSPFv2 (process 1) for IPv4** and **OSPFv3 (process 1) for IPv6**
simultaneously, in a **single area 0**, on the same interfaces. Both processes use the
router ID `0.0.0.<router#>`.

## How the two network types are configured

Point‑to‑point links (R1↔R2, R1↔R5) and the loopbacks use `network point-to-point`
(no DR/BDR). The R2/R3/R4 segment is left as `network broadcast` and the election is
**forced with priority** — no timers or authentication are touched.

```
! point-to-point link (R1 Et0/1, both address families)
 ip   ospf 1 area 0
 ip   ospf network point-to-point
 ipv6 ospf 1 area 0
 ipv6 ospf network point-to-point

! broadcast segment (R2 Et0/2 — highest priority => DR)
 ip   ospf network broadcast
 ip   ospf priority 255
 ipv6 ospf network broadcast
 ipv6 ospf priority 255
```

Priorities on the broadcast segment: **R2 = 255 → DR**, **R3 = 100 → BDR**,
**R4 = 0 → DROTHER** (priority 0 = never eligible). The HSRP LAN (R3/R4 Et0/2) carries
`ip/ipv6 ospf 1 area 0` but is set **`passive-interface`**, so the 10.0.34.0/24 and
`2001:db8:34::/64` prefixes are advertised into OSPF without forming an adjacency over the
FHRP LAN.

---

## Validation

### 1. OSPFv2 neighbors and DR/BDR — on R2 (the core / DR)

```
R2# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
0.0.0.3         100   FULL/BDR        00:00:34    10.0.234.3      Ethernet0/2
0.0.0.4           0   FULL/DROTHER    00:00:34    10.0.234.4      Ethernet0/2
0.0.0.1           0   FULL/  -        00:00:38    10.0.12.1       Ethernet0/1
```

Read it as:

* `0.0.0.3 … FULL/BDR` — R3 is the **Backup DR** (priority 100).
* `0.0.0.4 … FULL/DROTHER` — R4 is neither DR nor BDR (priority 0).
* `0.0.0.1 … FULL/  -` — the dash means **no DR/BDR role** on that neighbor, i.e. the
  R1↔R2 link is **point‑to‑point** (DR/BDR do not exist on p2p links). This one line lets
  students distinguish the two network types at a glance.

### 2. Confirm the network types and the local DR role

```
R2# show ip ospf interface brief

Interface    PID   Area            IP Address/Mask    Cost  State Nbrs F/C
Lo0          1     0               10.0.0.2/32        1     LOOP  0/0
Et0/2        1     0               10.0.234.2/24      10    DR    2/2      <- broadcast, R2 is DR
Et0/1        1     0               10.0.12.2/24       10    P2P   1/1      <- point-to-point
```

`State DR` on Et0/2 confirms R2 won the election; `State P2P` on Et0/1 confirms the
point‑to‑point type. `2/2` = two neighbors, both full.

> **Live talking point — the WAIT timer.** Immediately after boot the broadcast interface
> shows `State WAIT` and the neighbors sit at `2WAY/DROTHER` for up to ~40 s (the dead
> interval) while OSPF waits to see if a DR already exists. This is normal; the election
> completes on its own. You can force a re‑election any time with `clear ip ospf process`.

### 3. OSPFv3 (IPv6) — same election, separate process

```
R2# show ipv6 ospf neighbor

            OSPFv3 Router with ID (0.0.0.2) (Process ID 1)

Neighbor ID     Pri   State           Dead Time   Interface ID    Interface
0.0.0.3         100   FULL/BDR        00:00:33    2               Ethernet0/2
0.0.0.4           0   FULL/DROTHER    00:00:33    2               Ethernet0/2
0.0.0.1           0   FULL/  -        00:00:37    2               Ethernet0/1
```

Identical roles to IPv4 — a good chance to point out that **OSPFv3 is a separate protocol
instance** that happens to share the RID scheme, and that OSPFv3 neighbors are identified by
RID while adjacency runs over IPv6 **link‑local** addresses.

### 4. The point‑to‑point router (R1) sees only p2p neighbors

```
R1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
0.0.0.5           0   FULL/  -        00:00:31    10.0.15.5       Ethernet0/2
0.0.0.2           0   FULL/  -        00:00:36    10.0.12.2       Ethernet0/1
```

Both R1 adjacencies show the `-` role → both of R1's OSPF links are point‑to‑point.

### 5. Full reachability (IPv4 and IPv6)

Every loopback is reachable from every router, in both address families. Example, from R1:

```
R1# ping 10.0.0.3            !! R3 loopback via R2
!!!!! Success rate is 100 percent (5/5)
R1# ping 10.0.0.5            !! R5 loopback via the p2p link
!!!!! Success rate is 100 percent (5/5)
R1# ping 2001:DB8::3         !! R3 loopback, IPv6
!!!!! Success rate is 100 percent (5/5)
```

…and across the topology from a leaf (R3 → R5, proving the far corners reach each other):

```
R3# ping 10.0.0.5
!!!!! Success rate is 100 percent (5/5)
R3# ping 2001:DB8::5
!!!!! Success rate is 100 percent (5/5)
```

### 6. IPv6 OSPF routes use link‑local next‑hops

In `show ipv6 route` on R1, OSPFv3 routes point at `FE80::…` addresses — a nice contrast to
IPv4:

```
O   2001:DB8::2/128 [110/10]
     via FE80::A8BB:CCFF:FE00:610, Ethernet0/1
O   2001:DB8:234::/64 [110/20]
     via FE80::A8BB:CCFF:FE00:610, Ethernet0/1
```

---

## Instructor checklist

- [ ] `show ip ospf neighbor` — all adjacencies `FULL`; identify DR/BDR/DROTHER vs the `-` (p2p).
- [ ] `show ip ospf interface brief` — `DR` on R2 Et0/2, `P2P` on the point‑to‑point links.
- [ ] `show ipv6 ospf neighbor` — OSPFv3 mirrors OSPFv2.
- [ ] `show ip ospf` / `show ipv6 ospf` — confirm the RID is `0.0.0.x`.
- [ ] Ping loopback‑to‑loopback in both families — 100 %.
- [ ] Optional: `clear ip ospf process` on R2 and re‑watch the election complete.
