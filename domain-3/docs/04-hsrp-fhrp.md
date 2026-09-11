# Topic 3.4 — First‑Hop Redundancy Protocol (HSRP)

> **Exam objective 3.4** — Describe the purpose, functions and operation of first‑hop
> redundancy protocols. This is associate‑level: we configure the **bare minimum** and focus
> on reading **status** with `show` commands.

**R3** and **R4** share the `10.0.34.0/24` LAN and run **HSRPv2 group 34**.

## Minimal configuration

```
! R3 (intended ACTIVE)                    ! R4 (intended STANDBY)
interface Ethernet0/2                      interface Ethernet0/2
 standby version 2                          standby version 2
 standby 34 ip 10.0.34.254                  standby 34 ip 10.0.34.254
 standby 34 priority 110                    standby 34 priority 100
 standby 34 preempt                         !  (no preempt)
```

Only four ingredients: the **group number** (34), the **virtual IP** (`10.0.34.254`), a
**priority** to break the tie (higher wins → R3), and **preempt** so the higher‑priority
router reclaims the active role after a recovery.

---

## 1. Normal state

```
R3# show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Et0/2       34   110 P Active  local           10.0.34.4       10.0.34.254
```

```
R4# show standby brief
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Et0/2       34   100   Standby 10.0.34.3       local           10.0.34.254
```

R3 = **Active** (priority 110, `P` = preempt configured), R4 = **Standby** (priority 100).
The full view shows the virtual MAC and timers:

```
R3# show standby Ethernet0/2 34
Ethernet0/2 - Group 34 (version 2)
  State is Active
    2 state changes, last state change 00:02:12
  Virtual IP address is 10.0.34.254
  Active virtual MAC address is 0000.0c9f.f022 (MAC In Use)
    Local virtual MAC address is 0000.0c9f.f022 (v2 default)
  Hello time 3 sec, hold time 10 sec
  Preemption enabled
  Active router is local
  Standby router is 10.0.34.4, priority 100 (expires in 9.760 sec)
  Priority 110 (configured 110)
  Group name is "hsrp-Et0/2-34" (default)
```

Things to point out:

* **Virtual MAC `0000.0c9f.f022`** — the HSRPv2 IPv4 format `0000.0C9F.Fxxx` where `xxx` is
  the group in hex (`0x022` = **34**). Hosts ARP for the VIP and get this MAC regardless of
  which router is active.
* **`Hello 3 / hold 10`** — the *defaults*. Per the spec we did **not** tune HSRP timers.
* Any host on the LAN uses `10.0.34.254` as its default gateway and never has to know which
  physical router is forwarding.

---

## 2. Forcing function — fail the active router *(validated live)*

Shut the active router's LAN interface:

```
R3(config)# interface Ethernet0/2
R3(config-if)# shutdown
```

Within the 10‑second hold time, R4 promotes itself:

```
R4# show standby brief
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Et0/2       34   100   Active  local           unknown         10.0.34.254
```

R4 is now **Active** (Standby shows `unknown` because R3 is gone). Forwarding through the VIP
is uninterrupted — pinged straight through the failover from R1:

```
R1# ping 10.0.34.254
!!!!!
Success rate is 100 percent (5/5)
```

## 3. Validate preempt — recover the active router *(validated live)*

```
R3(config)# interface Ethernet0/2
R3(config-if)# no shutdown
```

Because R3 has **preempt** and the higher priority (110), it takes the active role back:

```
R3# show standby brief
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Et0/2       34   110 P Active  local           10.0.34.4       10.0.34.254

R4# show standby brief
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Et0/2       34   100   Standby 10.0.34.3       local           10.0.34.254
```

Back to the steady state. (Had R3 **not** been configured with `preempt`, R4 would have
*stayed* active after R3 recovered — a great "why do we need preempt?" moment.)

---

## Other forcing functions to try

* **Change priority instead of shutting a port.** On R4: `standby 34 priority 130` — with
  preempt it would take over; without preempt it waits for the next election. Contrast the two.
* **Watch the transitions:** `debug standby terse` on both routers while you flap R3 Et0/2 —
  students see `Active → Speak → Standby → Listen` state changes and the coup/resign hellos.
* **Track the MAC:** from a host (or another router) `show ip arp 10.0.34.254` before and
  after failover — the MAC (`0000.0c9f.f022`) does **not** change, which is the whole point.

## Instructor checklist

- [ ] `show standby brief` on R3/R4 — identify Active/Standby, priority, preempt flag, VIP.
- [ ] `show standby Ethernet0/2 34` — virtual MAC, default timers, group name.
- [ ] Shut the active interface → confirm the standby becomes active and the VIP still pings.
- [ ] Restore → confirm preempt returns the active role to R3.
