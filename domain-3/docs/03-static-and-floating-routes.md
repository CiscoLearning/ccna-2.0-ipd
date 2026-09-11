# Topic 3.2 — IPv4 & IPv6 static routing (incl. the floating static)

> **Exam objective 3.2** — Configure and verify IPv4 and IPv6 static routing, including
> default, network, host and **floating** static routes.

All of the static configuration for this topic lives on **R1**.

## What is configured

```
! --- IPv4 ---
ip route 192.168.50.0 255.255.255.0 10.0.12.2          ! network (standard) static
ip route 192.168.99.99 255.255.255.255 10.0.12.2       ! host static (/32)
ip route 10.0.34.0 255.255.255.0 10.0.0.2 200          ! FLOATING static (AD 200) -- has a bug

! --- IPv6 ---
ipv6 route 2001:DB8:CAFE::/64 2001:DB8:12::2           ! network static
ipv6 route 2001:DB8:BEEF::100/128 2001:DB8:12::2       ! host static (/128)
ipv6 route ::/0 2001:DB8:12::2                          ! default static

! (default route in the GLOBAL table is the OSPF-learned O*E2 0.0.0.0/0 from R2;
!  the MGMT VRF has its own static default -- see docs/05)
```

These appear in the tables as `S` entries — e.g. in `show ipv6 route` on R1:

```
S   ::/0 [1/0]                       via 2001:DB8:12::2
S   2001:DB8:CAFE::/64 [1/0]         via 2001:DB8:12::2
S   2001:DB8:BEEF::100/128 [1/0]     via 2001:DB8:12::2
```

---

## The floating static and its deliberate error

**Intent:** back up the HSRP LAN `10.0.34.0/24`. Under normal conditions OSPF (AD 110) is used;
if the OSPF path is lost, the AD‑200 static should take over.

**The bug:** the backup points at next‑hop **`10.0.0.2` — R2's loopback**, which R1 only ever
learns **through OSPF**. A static route whose next‑hop must itself be resolved by the very
protocol it is backing up **can never install when that protocol fails** (a *recursive
next‑hop* failure). So this "backup" fails in **both** required scenarios — when the OSPF
peering is shut **and** when the link is dropped.

### 1. Normal state — the floating static is dormant

The OSPF route wins on AD, so the static isn't installed (note it is **absent** from
`show ip route static`):

```
R1# show ip route 10.0.34.0
Routing entry for 10.0.34.0/24
  Known via "ospf 1", distance 110, metric 30, type intra area
  * 10.0.12.2, from 0.0.0.3, ... via Ethernet0/1

R1# show ip route static
Gateway of last resort is 10.0.12.2 to network 0.0.0.0
S     192.168.50.0/24 [1/0] via 10.0.12.2
      192.168.99.0/32 is subnetted, 1 subnets
S        192.168.99.99 [1/0] via 10.0.12.2          <-- only the two GOOD statics appear
```

The recursive next‑hop the backup depends on is itself OSPF‑learned — this is the crux:

```
R1# show ip route 10.0.0.2
Routing entry for 10.0.0.2/32
  Known via "ospf 1", distance 110, metric 11, type intra area
  * 10.0.12.2, from 0.0.0.2, ... via Ethernet0/1
```

### 2. Forcing function — drop the OSPF peering

Break the R1↔R2 adjacency. Either forcing function triggers the bug:

```
! (a) OSPF peering down, link stays up:
R1(config)# router ospf 1
R1(config-router)# passive-interface Ethernet0/1

! -- or (b) drop the link entirely:
R1(config)# interface Ethernet0/1
R1(config-if)# shutdown
```

After the adjacency times out:

```
R1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
0.0.0.5           0   FULL/  -        00:00:33    10.0.15.5       Ethernet0/2   (R2 is GONE)

R1# show ip route 10.0.0.2
% Subnet not in table                     <-- the backup's next-hop vanished

R1# show ip route 10.0.34.0
% Subnet not in table                     <-- destination is now UNREACHABLE

R1# show ip route static
Gateway of last resort is not set
S     192.168.50.0/24 [1/0] via 10.0.12.2
      192.168.99.0/32 is subnetted, 1 subnets
S        192.168.99.99 [1/0] via 10.0.12.2   <-- floating static STILL not installed

R1# ping 10.0.34.254
.....
Success rate is 0 percent (0/5)               <-- backup FAILED (the teaching point)
```

Notice the two *good* statics survive — they point at `10.0.12.2`, which is **directly
connected** and stays resolvable. The floating static does not, because `10.0.0.2` was only
reachable via the dead OSPF adjacency.

### 3. Prove the diagnosis — apply the fix (while OSPF is still down)

Repoint the backup at the **directly‑connected** next‑hop:

```
R1(config)# ip route 10.0.34.0 255.255.255.0 10.0.12.2 200
```

```
R1# show ip route 10.0.34.0
Routing entry for 10.0.34.0/24
  Known via "static", distance 200, metric 0
  * 10.0.12.2

R1# ping 10.0.34.254
!!!!!
Success rate is 100 percent (5/5)             <-- backup now works, even with OSPF down
```

### 4. Restore

```
R1(config)# no ip route 10.0.34.0 255.255.255.0 10.0.12.2 200   ! keep the intentional bug for the demo
R1(config)# router ospf 1
R1(config-router)# no passive-interface Ethernet0/1              ! (or 'no shutdown' the link)
```

```
R1# show ip ospf neighbor
0.0.0.2   0   FULL/  -   ...  Ethernet0/1        (adjacency restored)
0.0.0.5   0   FULL/  -   ...  Ethernet0/2
R1# show ip route 10.0.34.0
  Known via "ospf 1", distance 110, metric 30 ...
R1# ping 10.0.34.254
!!!!! Success rate is 100 percent (5/5)
```

---

## Why it matters / the nuance

| Forcing function | Directly‑connected next‑hop (`10.0.12.2`) | Recursive next‑hop (`10.0.0.2`, as shipped) |
|---|---|---|
| OSPF peering down, **link up** | backup **works** (10.0.12.2 still connected) | backup **fails** (10.0.0.2 was OSPF‑only) |
| Link **dropped** | backup fails (interface down) | backup fails |

The shipped design uses the recursive next‑hop precisely so the backup fails in **both**
cases — matching the specification. It teaches a subtle, real‑world lesson: **a floating
static is only as good as the resolvability of its next‑hop.**

## Suggested "play" activities

1. Reproduce the failure with **both** forcing functions (passive‑interface vs `shutdown`) and
   compare — the directly‑connected fix survives (a) but not (b). Why?
2. Change the floating static's AD from `200` to `105` and re‑examine — with an AD **below**
   OSPF's 110, the (still‑resolvable‑or‑not) static now competes with OSPF. What installs?
3. Add `ip route 10.0.34.0 255.255.255.0 10.0.12.2 200` as a *second* backup alongside the
   buggy one and watch which one CEF actually uses.
4. Use `show ip route 10.0.34.0` and `debug ip routing` while flapping the link to watch the
   RIB add/withdraw the backup in real time.
