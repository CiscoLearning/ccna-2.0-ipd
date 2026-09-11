# Management VRF & external reachability

> Design requirement (not a numbered exam topic, but part of the lab spec): keep the global
> routing table clean by putting management on its **own VRF**, reachable out‑of‑band through
> an unmanaged switch to an external connector, and verify every router can reach a workstation
> at **`198.18.133.252`**.

## Why a VRF?

Topic 3.1 asks students to *read* the global routing table. If management reachability
(`198.18.0.0`, a default route to the lab gateway, etc.) lived in the global table it would add
noise and a second default route. Placing Et0/0 in VRF **`MGMT`** gives management its own
routing table, so the global table shows only the topology you want to teach.

## Configuration (identical pattern on all five routers)

```
vrf definition MGMT
 address-family ipv4
 exit-address-family
!
interface Ethernet0/0                 ! the first usable interface
 vrf forwarding MGMT
 ip address 198.18.180.<router#> 255.255.192.0     ! /18
 no shutdown
!
ip route vrf MGMT 0.0.0.0 0.0.0.0 198.18.128.1     ! default within the VRF only
```

Physically, every Et0/0 lands on the **SW‑MGMT** unmanaged switch, which uplinks to the
**`ext-bridge0`** external connector (set to **System Bridge / `bridge0`**). That bridges the
management segment to the `198.18.128.0/18` network on the CML host, where the workstation
lives.

Addresses: `198.18.180.1‑5/18` (R1‑R5), gateway `198.18.128.1`, workstation `198.18.133.252`.
Note the `/18` mask means `198.18.128.0`–`198.18.191.255` are all one subnet, so `.180.x`,
`.128.1` and `.133.252` are directly on‑link.

## Validation — reachability from every router

`ping vrf MGMT 198.18.133.252` succeeds from **all five** routers:

```
R1# ping vrf MGMT 198.18.133.252
.!!!!  Success rate is 80 percent (4/5), round-trip min/avg/max = 1/1/2 ms
R2# ping vrf MGMT 198.18.133.252
.!!!!  Success rate is 80 percent (4/5), round-trip min/avg/max = 1/2/3 ms
R3# ping vrf MGMT 198.18.133.252
.!!!!  Success rate is 80 percent (4/5), round-trip min/avg/max = 1/2/3 ms
R4# ping vrf MGMT 198.18.133.252
.!!!!  Success rate is 80 percent (4/5), round-trip min/avg/max = 1/1/1 ms
R5# ping vrf MGMT 198.18.133.252
.!!!!  Success rate is 80 percent (4/5), round-trip min/avg/max = 1/1/2 ms
```

**Result: the workstation is reachable from all five devices over the MGMT VRF.** The single
dropped packet in each run is the **first** echo, lost while the router resolves ARP for the
next‑hop — normal behavior, not a connectivity problem. Repeat the ping and it is 100 %.

You must specify `vrf MGMT` — a plain `ping 198.18.133.252` uses the **global** table (which
has no route there) and will fail. That contrast is itself a teaching point about VRFs.

## Confirm the separation

```
R1# show ip route vrf MGMT

Routing Table: MGMT
Gateway of last resort is 198.18.128.1 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 198.18.128.1
C     198.18.128.0/18 is directly connected, Ethernet0/0
      198.18.180.0/32 is subnetted, 1 subnets
L        198.18.180.1 is directly connected, Ethernet0/0
```

The VRF has its own default route and connected subnet; none of this appears in the global
`show ip route` used for topic 3.1.

## If the external ping fails in your environment

External reachability depends on the CML host: `bridge0` must be attached to a network where
`198.18.128.0/18` (and the `198.18.133.252` workstation) actually exist. **All other topics in
this lab — OSPF, the routing table, HSRP, the floating static — work with no external
connectivity at all.** If your pod has a different management network, change the four
management values (`198.18.180.x`, mask, gateway `198.18.128.1`, and the connector's bridge)
to match; nothing else in the topology needs to change.
