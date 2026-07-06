# Advanced BGP on Cisco IOS XRv9000 — EVE-NG Lab

![Platform](https://img.shields.io/badge/platform-Cisco%20IOS%20XRv9000-1BA0D7?logo=cisco&logoColor=white)
![IOS XR](https://img.shields.io/badge/IOS%20XR-24.3.1-005073)
![EVE-NG](https://img.shields.io/badge/EVE--NG-Community%20%7C%20Pro-orange)
![Protocol](https://img.shields.io/badge/protocol-BGP--4%20%2B%20RR%20%2B%20RPKI-success)
![Phases](https://img.shields.io/badge/phases-6-blueviolet)

A hands-on lab to actually **understand** BGP — not just paste a peering template. You build a
dual-homed AS (two upstream ISPs), scale iBGP with route reflectors, then bend traffic where
you want it using the BGP best-path algorithm, communities, and AS-path prepending — and finally
secure the edge with prefix filtering, max-prefix, and RPKI origin validation.

> **How to use this lab:** follow the phases in order. Paste the config, run the verify commands,
> confirm it works — *then* move on. The goal isn't a working config (that's the easy part);
> it's being able to **explain why each route wins**. Each phase ends with a "Can I explain it?"
> question — answer it before moving on.
>
> Theory first? → [`docs/CONCEPTS.md`](docs/CONCEPTS.md) ·
> Best-path order → [`docs/PATH-SELECTION.md`](docs/PATH-SELECTION.md) ·
> Communities → [`docs/COMMUNITIES.md`](docs/COMMUNITIES.md) ·
> RPKI → [`docs/RPKI.md`](docs/RPKI.md)

> **Same diamond as the MPLS-TE and SRv6 labs** — same routers, same core links, same loopbacks.
> Only the external ISP attachments and the BGP overlay are new. Reusing the topology means you
> learn *BGP*, not a new map.

---

## TL;DR — quick start

For those who already know IOS XR and just want to build it:

1. Build the 6-node topology in EVE-NG (R1–R4 + ISP-A + ISP-B — wire per the [link table](#addressing-plan)).
2. Paste each device's full config from [`configs/`](configs/). Configs are the **final state**
   (route-reflector overlay, all 6 phases). New XR interfaces may come up admin-down — `no shutdown` when pasting.
3. Verify end to end:

```
! on R1 — sessions up: 1 eBGP to ISP-A + 2 iBGP to the RRs
show bgp ipv4 unicast summary

! on R1 — the dual-homed prefix prefers ISP-A (local-pref 200)
show bgp 8.8.8.0/24

! on R4 — even R4 (attached to ISP-B) prefers reaching 8.8.8.0/24 via R1 (iBGP, LP 200)
show bgp 8.8.8.0/24

! on ISP-B — our aggregate arrives with a prepended AS-path (65000 65000 65000)
show bgp 172.16.0.0/16
```

Want to learn instead of speed-run? Skip this and follow the phases.

---

## Lab Environment

| Component | Detail |
|---|---|
| Emulator | EVE-NG (Community/Pro) |
| Node image | Cisco IOS XRv9000 (24.3.1) |
| Our AS (core) | 4 × XRv9000 — R1–R4, **AS 65000** |
| Upstream ISPs | 2 × XRv9000 — ISP-A **AS 65100**, ISP-B **AS 65200** |
| IGP (underlay) | IS-IS Level-2-only (core only — ISPs are eBGP, not in the IGP) |
| Overlay | iBGP (full mesh → route reflectors), eBGP to both ISPs |

---

## Topology

![Advanced BGP lab — dual-ISP diamond with redundant route reflectors](diagrams/topology.svg)

<details>
<summary>Same topology as ASCII (copy-paste friendly)</summary>

```
                 +-----------+                         +-----------+
                 |  ISP-A    |                         |  ISP-B    |
                 | AS 65100  |                         | AS 65200  |
                 +-----+-----+                         +-----+-----+
                       | eBGP                                | eBGP
                       |                                     |
                 +-----+-----+                         +-----+-----+
                 |    R1     |   <-- edge / RR client    |    R4     |
                 | 1.1.1.1   |                           | 4.4.4.4   |
                 +--+-----+--+                           +--+-----+--+
                    |     |                                 |     |
            R1-R2   |     |  R1-R3                   R2-R4   |     |  R3-R4
                    |     |                                 |     |
          +---------+-+ +-+---------+             +---------+-+ +-+---------+
          |    R2     |-|    R3     |  R2-R3 x-link|   (R2)    |-|   (R3)    |
          | 2.2.2.2   | | 3.3.3.3   |             |           | |           |
          |  RR       | |  RR       |             +-----------+ +-----------+
          +-----------+ +-----------+
              (R2 and R3 are the two redundant route reflectors)
```

</details>

> GitHub renders the SVG in light mode inside `<img>`; open `diagrams/topology.svg` directly for the
> dark-mode-adaptive version.

Logical roles: **R1** and **R4** are the AS edges (one eBGP upstream each) and are
route-reflector **clients**. **R2** and **R3** sit in the core and act as **redundant route
reflectors**. The diamond gives every client two paths to each RR, so losing one RR or one core
link never isolates a client.

See [`diagrams/topology.md`](diagrams/topology.md) for the BGP session mesh vs. the physical wiring.

---

## Addressing Plan

| Node | Role | AS | Loopback0 |
|------|------|----|-----------|
| R1 | Edge → ISP-A, RR client | 65000 | 1.1.1.1/32 |
| R2 | Route reflector | 65000 | 2.2.2.2/32 |
| R3 | Route reflector | 65000 | 3.3.3.3/32 |
| R4 | Edge → ISP-B, RR client | 65000 | 4.4.4.4/32 |
| ISP-A | Upstream | 65100 | 100.100.100.100/32 |
| ISP-B | Upstream | 65200 | 200.200.200.200/32 |

| Link | Subnet | A-end | B-end |
|------|--------|-------|-------|
| R1–R2 | 10.12.0.0/30 | R1 Gi0/0/0/1 .1 | R2 Gi0/0/0/0 .2 |
| R1–R3 | 10.13.0.0/30 | R1 Gi0/0/0/3 .1 | R3 Gi0/0/0/2 .2 |
| R2–R3 (cross) | 10.23.0.0/30 | R2 Gi0/0/0/1 .1 | R3 Gi0/0/0/0 .2 |
| R2–R4 | 10.24.0.0/30 | R2 Gi0/0/0/3 .1 | R4 Gi0/0/0/2 .2 |
| R3–R4 | 10.34.0.0/30 | R3 Gi0/0/0/4 .1 | R4 Gi0/0/0/3 .2 |
| R1–ISP-A | 100.64.1.0/30 | R1 Gi0/0/0/0 .1 | ISP-A Gi0/0/0/0 .2 |
| R4–ISP-B | 100.64.2.0/30 | R4 Gi0/0/0/1 .1 | ISP-B Gi0/0/0/0 .2 |

**Prefixes in play**

| Prefix | Originated by | Purpose |
|--------|---------------|---------|
| 172.16.0.0/16 | Us (R1 **and** R4) | Our aggregate — advertised to both ISPs |
| 8.8.8.0/24 | **Both** ISP-A and ISP-B | Dual-homed "internet" prefix — the best-path test case |
| 203.0.113.0/24 | ISP-A only | Single-homed via ISP-A |
| 198.51.100.0/24 | ISP-B only | Single-homed via ISP-B |

---

## Phases

| Phase | Topic | The "real engineer" question it answers |
|-------|-------|------------------------------------------|
| 1 | IS-IS underlay | Why does iBGP need an IGP under it? |
| 2 | iBGP full mesh + eBGP multihoming | Why doesn't iBGP re-advertise between peers? Why next-hop-self? |
| 3 | Route reflectors | How do you scale iBGP past a full mesh — safely? |
| 4 | Path selection (local-pref, weight, MED) | Which path wins, and *why*, step by step? |
| 5 | Communities + AS-path prepend | How do I steer **inbound** vs **outbound** traffic? |
| 6 | Filtering, max-prefix, RPKI, failover | How do I make the edge safe and prove failover works? |

---

## Results

- [ ] **Phase 1** — IS-IS adjacencies up, all four core loopbacks reachable
- [ ] **Phase 2** — eBGP to both ISPs up; iBGP full mesh up; 172.16/16 seen at both ISPs; 8.8.8.0/24 seen on all core routers
- [ ] **Phase 3** — full mesh migrated to RR; clients still learn every route; `originator-id`/`cluster-list` present
- [ ] **Phase 4** — 8.8.8.0/24 prefers ISP-A AS-wide (local-pref 200); best-path reason confirmed
- [ ] **Phase 5** — ISP-B sees our aggregate with a prepended AS-path; community tags visible on learned routes
- [ ] **Phase 6** — allow-list + max-prefix in place; shutting ISP-A fails 8.8.8.0/24 over to ISP-B

---

## How to build this — two paths

**Path A — speed-run (you already know XR):** paste each device's full file from [`configs/`](configs/)
(final state, all 6 phases) and skip to the [TL;DR verify](#tldr--quick-start).

**Path B — learn it phase by phase (recommended):** work down the phases below. Each phase has a
collapsible **► Configure** block holding *only the delta that phase adds*, per device. Paste that
block into the matching router, `commit`, run the verify commands, then move on. The configs are
**cumulative** — Phase 4 edits the same route-policy Phase 2 created, so by Phase 6 you've landed
exactly on the shipped `configs/` files.

**The IOS-XR candidate/commit workflow (every paste):**

```
conf t                 ! enter global config — you're now editing a CANDIDATE, nothing is live yet
<paste the phase block>
show configuration      ! (optional) review the candidate before it takes effect
commit                  ! atomically apply the whole candidate — this is what activates it
end
```

Nothing you paste is active until `commit`. If a paste looks wrong, `abort` throws the candidate away
before it ever touches the running config. After changing an inbound route-policy, tell BGP to
re-evaluate what a neighbor already sent you without bouncing the session:

```
clear bgp ipv4 unicast <neighbor> soft in    ! re-apply inbound policy (no session reset)
```

> **XR gotcha:** newly added interfaces can come up admin-down — add `no shutdown` under any
> interface that stays down after commit.

---

## Phase 1 — IS-IS underlay

**Objective:** every core loopback reachable before any BGP. iBGP peers sit on loopbacks, so
the loopbacks must be reachable first — that's the IGP's only job here.

**Why an IGP under BGP at all?** iBGP sessions are built between loopbacks that may be several
hops apart. BGP doesn't find those loopbacks itself — the IGP (IS-IS) does. The IGP carries
*infrastructure* routes (loopbacks, links); BGP carries *everything else* (customer/internet
prefixes). Keep the two jobs separate and the design stays sane.

<details>
<summary><b>► Configure Phase 1</b> — paste per device, then <code>commit</code> (R1–R4; ISPs are not in the IGP)</summary>

```
! ===================== R1 =====================
interface Loopback0
 ipv4 address 1.1.1.1 255.255.255.255
!
interface GigabitEthernet0/0/0/1
 ipv4 address 10.12.0.1 255.255.255.252          ! to R2
!
interface GigabitEthernet0/0/0/3
 ipv4 address 10.13.0.1 255.255.255.252          ! to R3
!
router isis CORE
 is-type level-2-only
 net 49.0001.0000.0000.0001.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
  !
 !
!
commit
```

```
! ===================== R2 =====================
interface Loopback0
 ipv4 address 2.2.2.2 255.255.255.255
!
interface GigabitEthernet0/0/0/0
 ipv4 address 10.12.0.2 255.255.255.252          ! to R1
!
interface GigabitEthernet0/0/0/1
 ipv4 address 10.23.0.1 255.255.255.252          ! to R3 (cross-link)
!
interface GigabitEthernet0/0/0/3
 ipv4 address 10.24.0.1 255.255.255.252          ! to R4
!
router isis CORE
 is-type level-2-only
 net 49.0001.0000.0000.0002.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
  !
 !
!
commit
```

```
! ===================== R3 =====================
interface Loopback0
 ipv4 address 3.3.3.3 255.255.255.255
!
interface GigabitEthernet0/0/0/0
 ipv4 address 10.23.0.2 255.255.255.252          ! to R2 (cross-link)
!
interface GigabitEthernet0/0/0/2
 ipv4 address 10.13.0.2 255.255.255.252          ! to R1
!
interface GigabitEthernet0/0/0/4
 ipv4 address 10.34.0.1 255.255.255.252          ! to R4
!
router isis CORE
 is-type level-2-only
 net 49.0001.0000.0000.0003.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/4
  point-to-point
  address-family ipv4 unicast
  !
 !
!
commit
```

```
! ===================== R4 =====================
interface Loopback0
 ipv4 address 4.4.4.4 255.255.255.255
!
interface GigabitEthernet0/0/0/2
 ipv4 address 10.24.0.2 255.255.255.252          ! to R2
!
interface GigabitEthernet0/0/0/3
 ipv4 address 10.34.0.2 255.255.255.252          ! to R3
!
router isis CORE
 is-type level-2-only
 net 49.0001.0000.0000.0004.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
  !
 !
!
commit
```

> **Only the system-id differs** (`...0001`→`...0004`), and each router lists *its own* core
> interfaces. The ISP-facing interfaces (`100.64.x`) are added in Phase 2 — they never enter IS-IS.

</details>

**Verify**

```
show isis neighbors                 ! adjacencies on every core link
show route isis                     ! all loopbacks learned
ping 4.4.4.4 source 1.1.1.1         ! R1 -> R4 loopback over IS-IS
```

Expected: full adjacencies; R1 reaches 2.2.2.2/3.3.3.3/4.4.4.4. The ISP links are **not** in
IS-IS — they only carry eBGP.

> **Can I explain it?** Why don't we just run BGP directly between connected interfaces instead
> of loopbacks? (Hint: loopbacks never go down when one link does — session survives a link failure.)

---

## Phase 2 — iBGP full mesh + eBGP multihoming

**Objective:** bring up eBGP to both ISPs and iBGP among all four core routers (full mesh first),
originate our aggregate, and learn the ISP prefixes everywhere.

**Two BGP rules that trip everyone up:**

1. **iBGP is not transitive.** A route learned from one iBGP peer is **not** re-advertised to
   another iBGP peer. That's the loop-prevention rule — and exactly why a full mesh (every iBGP
   speaker peered with every other) is required... until route reflectors (Phase 3) change the rule.
   Full mesh sessions grow as *n(n-1)/2*: 4 routers = 6 sessions; 20 routers = 190. It does not scale.

2. **Next-hop on iBGP.** When R1 learns 8.8.8.0/24 from ISP-A, the next-hop is the ISP's link
   address (100.64.1.2). If R1 hands that to its iBGP peers unchanged, they have no route to
   100.64.1.2 (it's not in the IGP). `next-hop-self` rewrites the next-hop to R1's loopback,
   which the IGP *does* know. Forget this and routes are in the table but unreachable.

<details>
<summary><b>► Configure Phase 2</b> — eBGP + iBGP full mesh + originate the aggregate (all 6 devices)</summary>

> The full mesh is deliberate here so you *feel* why it doesn't scale. Phase 3 tears the
> `R1↔R4` session down and moves to route reflectors. The `ISP-*-IN` policies start as a plain
> `pass` and get tightened in Phases 4–6.

```
! ===================== R1 (edge → ISP-A) =====================
interface GigabitEthernet0/0/0/0
 ipv4 address 100.64.1.1 255.255.255.252         ! to ISP-A — eBGP only, NOT in IS-IS
!
prefix-set OUR-AGGREGATE
  172.16.0.0/16
end-set
!
route-policy ISP-A-IN
  pass                                            ! Phase 2: accept everything for now
end-policy
!
route-policy ISP-A-OUT
  if destination in OUR-AGGREGATE then
    pass                                          ! advertise ONLY our aggregate (no transit)
  else
    drop
  endif
end-policy
!
router static
 address-family ipv4 unicast
  172.16.0.0/16 Null0                             ! RIB route so 'network' can originate it
 !
!
router bgp 65000
 bgp router-id 1.1.1.1
 address-family ipv4 unicast
  network 172.16.0.0/16
 !
 neighbor 100.64.1.2
  remote-as 65100
  description eBGP-to-ISP-A
  address-family ipv4 unicast
   route-policy ISP-A-IN in
   route-policy ISP-A-OUT out
  !
 !
 ! --- iBGP FULL MESH: peer R2, R3, AND R4 ---
 neighbor 2.2.2.2
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
   next-hop-self
  !
 !
 neighbor 3.3.3.3
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
   next-hop-self
  !
 !
 neighbor 4.4.4.4
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
   next-hop-self                                  ! Phase-2-only mesh session (removed in Phase 3)
  !
 !
!
commit
```

```
! ===================== R4 (edge → ISP-B) =====================
interface GigabitEthernet0/0/0/1
 ipv4 address 100.64.2.1 255.255.255.252         ! to ISP-B — eBGP only, NOT in IS-IS
!
prefix-set OUR-AGGREGATE
  172.16.0.0/16
end-set
!
route-policy ISP-B-IN
  pass
end-policy
!
route-policy ISP-B-OUT
  if destination in OUR-AGGREGATE then
    pass
  else
    drop
  endif
end-policy
!
router static
 address-family ipv4 unicast
  172.16.0.0/16 Null0
 !
!
router bgp 65000
 bgp router-id 4.4.4.4
 address-family ipv4 unicast
  network 172.16.0.0/16
 !
 neighbor 100.64.2.2
  remote-as 65200
  description eBGP-to-ISP-B
  address-family ipv4 unicast
   route-policy ISP-B-IN in
   route-policy ISP-B-OUT out
  !
 !
 neighbor 2.2.2.2
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
   next-hop-self
  !
 !
 neighbor 3.3.3.3
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
   next-hop-self
  !
 !
 neighbor 1.1.1.1
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
   next-hop-self                                  ! Phase-2-only mesh session (removed in Phase 3)
  !
 !
!
commit
```

```
! ===================== R2 (full-mesh member: peers R1, R3, R4) =====================
router bgp 65000
 bgp router-id 2.2.2.2
 address-family ipv4 unicast
 !
 neighbor 1.1.1.1
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
  !
 !
 neighbor 3.3.3.3
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
  !
 !
 neighbor 4.4.4.4
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
  !
 !
!
commit
```

```
! ===================== R3 (full-mesh member: peers R1, R2, R4) =====================
router bgp 65000
 bgp router-id 3.3.3.3
 address-family ipv4 unicast
 !
 neighbor 1.1.1.1
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
  !
 !
 neighbor 2.2.2.2
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
  !
 !
 neighbor 4.4.4.4
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
  !
 !
!
commit
```

```
! ===================== ISP-A (AS 65100) — full node, one shot =====================
route-policy PASS
  pass
end-policy
!
router static
 address-family ipv4 unicast
  8.8.8.0/24 Null0
  203.0.113.0/24 Null0
 !
!
interface Loopback0
 ipv4 address 100.100.100.100 255.255.255.255
!
interface GigabitEthernet0/0/0/0
 ipv4 address 100.64.1.2 255.255.255.252
!
router bgp 65100
 bgp router-id 100.100.100.100
 address-family ipv4 unicast
  network 8.8.8.0/24
  network 203.0.113.0/24
 !
 neighbor 100.64.1.1
  remote-as 65000
  description eBGP-to-R1-AS65000
  address-family ipv4 unicast
   route-policy PASS in
   route-policy PASS out
  !
 !
!
commit
```

```
! ===================== ISP-B (AS 65200) — full node, one shot =====================
route-policy PASS
  pass
end-policy
!
router static
 address-family ipv4 unicast
  8.8.8.0/24 Null0
  198.51.100.0/24 Null0
 !
!
interface Loopback0
 ipv4 address 200.200.200.200 255.255.255.255
!
interface GigabitEthernet0/0/0/0
 ipv4 address 100.64.2.2 255.255.255.252
!
router bgp 65200
 bgp router-id 200.200.200.200
 address-family ipv4 unicast
  network 8.8.8.0/24
  network 198.51.100.0/24
 !
 neighbor 100.64.2.1
  remote-as 65000
  description eBGP-to-R4-AS65000
  address-family ipv4 unicast
   route-policy PASS in
   route-policy PASS out
  !
 !
!
commit
```

</details>

**Verify**

```
show bgp ipv4 unicast summary               ! all sessions Established; State/PfxRcd numeric
show bgp 8.8.8.0/24                          ! learned via both ISPs
show route 8.8.8.0/24                        ! installed, next-hop resolvable
! on ISP-A and ISP-B:
show bgp 172.16.0.0/16                       ! our aggregate arrived
```

> **Can I explain it?** R2 learns 8.8.8.0/24 from R1 over iBGP. Will R2 pass it to R3 over iBGP?
> (No — iBGP non-transitivity. That's the whole reason for the full mesh / route reflectors.)

---

## Phase 3 — Route reflectors

**Objective:** kill the full mesh. Make R2 and R3 route reflectors; R1 and R4 become clients that
peer **only** to the two RRs. The RR is allowed to re-advertise (reflect) iBGP routes — breaking
the non-transitivity rule on purpose, with loop prevention built in.

**How an RR stays loop-free:** when an RR reflects a route it adds two attributes —
`originator-id` (the router-id of the route's origin inside the AS) and `cluster-list` (the
chain of RR cluster-ids it passed through). A router that sees its own originator-id, or an RR
that sees its own cluster-id in the cluster-list, drops the route. No counting to infinity.

**Reflection rules (what an RR sends where):**
- From a **client** → reflect to all clients **and** non-clients.
- From a **non-client** → reflect to clients **only**.
- From an **eBGP** peer → send to everyone (normal).

<details>
<summary><b>► Configure Phase 3</b> — promote R2/R3 to RRs, then drop the R1↔R4 mesh session</summary>

> Migrate safely: first add the client keywords on the RRs, confirm every route still arrives,
> *then* remove the direct `R1↔R4` session. Re-applying a `neighbor` block just adds the new
> sub-command; `no neighbor` removes a session.

```
! ===================== R2 (route reflector) =====================
router bgp 65000
 neighbor 1.1.1.1
  description RR-CLIENT-R1
  address-family ipv4 unicast
   route-reflector-client
  !
 !
 neighbor 4.4.4.4
  description RR-CLIENT-R4
  address-family ipv4 unicast
   route-reflector-client
  !
 !
 neighbor 3.3.3.3
  description iBGP-RR-peer-R3                      ! other RR — non-client, no keyword
 !
!
commit
```

```
! ===================== R3 (route reflector) =====================
router bgp 65000
 neighbor 1.1.1.1
  description RR-CLIENT-R1
  address-family ipv4 unicast
   route-reflector-client
  !
 !
 neighbor 4.4.4.4
  description RR-CLIENT-R4
  address-family ipv4 unicast
   route-reflector-client
  !
 !
 neighbor 2.2.2.2
  description iBGP-RR-peer-R2                      ! other RR — non-client, no keyword
 !
!
commit
```

```
! ===================== R1 (drop the full-mesh session to R4) =====================
router bgp 65000
 no neighbor 4.4.4.4
!
commit
```

```
! ===================== R4 (drop the full-mesh session to R1) =====================
router bgp 65000
 no neighbor 1.1.1.1
!
commit
```

> After this, R1 and R4 each hold exactly **two** iBGP sessions (to R2 and R3). Their descriptions
> in the shipped configs read `iBGP-to-RR-R2` / `iBGP-to-RR-R3` — cosmetic, add them if you like.

</details>

**Redundancy detail:** R1 and R4 each peer **both** RRs, and each RR keeps its **own cluster-id**
(the default = its router-id). Two cluster-ids means full path diversity and the cluster-list
still prevents loops. Migrate by adding the RR sessions, confirming routes still arrive, then
removing the old R1↔R4 mesh session.

**Verify**

```
show bgp 8.8.8.0/24                  ! still present on every router after removing the mesh
show bgp 8.8.8.0/24 detail           ! note originator-id and cluster-list attributes
show bgp ipv4 unicast summary        ! R1/R4 now have 2 iBGP sessions (to RRs), not 3
```

> **Can I explain it?** If both RRs used the *same* cluster-id, what would you lose? (Path
> diversity — an RR ignores a reflected route carrying its own cluster-id, so identical
> cluster-ids can hide a valid backup path.)

---

## Phase 4 — Path selection (the heart of BGP)

**Objective:** make the whole AS prefer **ISP-A** to reach the dual-homed 8.8.8.0/24, using
**local-preference** — and understand exactly where that sits in the best-path algorithm.

**The best-path algorithm, in order** (full version in [`docs/PATH-SELECTION.md`](docs/PATH-SELECTION.md)):

```
1. Highest WEIGHT            (Cisco-only, local to one router)
2. Highest LOCAL-PREFERENCE  (AS-wide, the main OUTBOUND lever)   <-- we use this
3. Locally originated
4. Shortest AS-PATH
5. Lowest ORIGIN (IGP < EGP < incomplete)
6. Lowest MED                (hint to a neighbor AS)
7. eBGP over iBGP
8. Lowest IGP metric to next-hop
9. Oldest / lowest router-id ... (tie-breakers)
```

8.8.8.0/24 arrives with an equal 1-hop AS-path from both ISPs, so AS-path can't decide it —
**local-preference** does. `ISP-A-IN` sets local-preference **200** on routes from ISP-A; ISP-B's
stay at the default **100**. Local-pref is carried across iBGP, so *every* router in AS 65000 —
including R4, which is physically attached to ISP-B — prefers the exit via R1/ISP-A.

<details>
<summary><b>► Configure Phase 4</b> — set local-preference on the two edges (R1, R4)</summary>

> You're **replacing** the `pass`-only policies from Phase 2. Re-pasting a `route-policy` with the
> same name overwrites it. After commit, run the `clear ... soft in` so BGP re-scores routes the
> ISP already sent.

```
! ===================== R1 (ISP-A → local-pref 200) =====================
route-policy ISP-A-IN
  if destination in OUR-AGGREGATE then
    drop                                          ! never accept our own space back
  else
    set local-preference 200                      ! higher wins -> ISP-A preferred AS-wide
    pass
  endif
end-policy
!
commit
!
clear bgp ipv4 unicast 100.64.1.2 soft in
```

```
! ===================== R4 (ISP-B → explicit local-pref 100) =====================
route-policy ISP-B-IN
  if destination in OUR-AGGREGATE then
    drop
  else
    set local-preference 100                      ! default value, set explicitly -> ISP-A (200) wins
    pass
  endif
end-policy
!
commit
!
clear bgp ipv4 unicast 100.64.2.2 soft in
```

</details>

**Verify**

```
! on R1:
show bgp 8.8.8.0/24                 ! best path = via ISP-A; "best" flag set
! on R4 (attached to ISP-B!):
show bgp 8.8.8.0/24                 ! best path = via 1.1.1.1 (iBGP), localpref 200
show bgp 8.8.8.0/24 detail          ! read the "best #1, ... reason" line out loud
```

**Contrast — weight vs local-pref:** weight (step 1) is *local to one router* and not advertised;
local-pref (step 2) is *AS-wide*. If you only set weight on R1, R4 would still send 8.8.8.0/24
traffic out ISP-B. Try it: set `weight 500` on R1's ISP-A neighbor instead of local-pref and watch
R4 disagree.

> **Can I explain it?** Why is local-preference the right tool to make the *whole AS* exit one way,
> but weight is not?

---

## Phase 5 — Communities + AS-path prepend (steering inbound)

**Objective:** influence **inbound** traffic (the internet → us) with AS-path prepending, and tag
routes with **communities** so policy is driven by labels, not by re-matching prefixes everywhere.

**Outbound vs inbound — the asymmetry that confuses everyone:**
- **Outbound** (us → internet): *we* decide, with local-pref/weight (Phase 4). Easy, deterministic.
- **Inbound** (internet → us): the *other* networks decide. We can only *influence* them — most
  commonly by making one path look worse via **AS-path prepend** (a longer AS-path loses at step 4).

We advertise our aggregate 172.16.0.0/16 out both edges. On R4 (→ ISP-B) we prepend our AS twice,
so the wider internet sees `65000 65000 65000` via ISP-B vs `65000` via ISP-A → traffic comes in via ISP-A.

<details>
<summary><b>► Configure Phase 5</b> — tag inbound with communities (R1, R4) + prepend outbound to ISP-B (R4)</summary>

```
! ===================== R1 (add community tag to ISP-A routes) =====================
route-policy ISP-A-IN
  if destination in OUR-AGGREGATE then
    drop
  else
    set local-preference 200
    set community (65000:100) additive            ! Phase 5: mark "learned via ISP-A"
    pass
  endif
end-policy
!
commit
!
clear bgp ipv4 unicast 100.64.1.2 soft in
```

```
! ===================== R4 (tag ISP-B routes + prepend our aggregate to ISP-B) =====================
route-policy ISP-B-IN
  if destination in OUR-AGGREGATE then
    drop
  else
    set local-preference 100
    set community (65000:200) additive            ! Phase 5: mark "learned via ISP-B"
    pass
  endif
end-policy
!
route-policy ISP-B-OUT
  if destination in OUR-AGGREGATE then
    prepend as-path 65000 2                        ! add our ASN 2 extra times -> longer path
    pass                                           ! so the internet reaches us via ISP-A
  else
    drop
  endif
end-policy
!
commit
!
clear bgp ipv4 unicast 100.64.2.2 soft in         ! re-score inbound
clear bgp ipv4 unicast 100.64.2.2 soft out        ! re-advertise the prepended aggregate
```

</details>

**Communities** ([`docs/COMMUNITIES.md`](docs/COMMUNITIES.md)): we tag inbound ISP routes —
`65000:100` from ISP-A, `65000:200` from ISP-B — so any later policy can act on "where did this
come from?" without re-listing prefixes. Well-known communities matter too: `no-export` keeps a
route inside the AS (don't leak it to eBGP peers); `no-advertise` stops it being sent to *any* peer.

**Verify**

```
! on ISP-B — our aggregate now has a longer AS-path:
show bgp 172.16.0.0/16              ! AS-path shows 65000 65000 65000
! on ISP-A — still the short path:
show bgp 172.16.0.0/16              ! AS-path shows 65000
! on R1 — community tag applied to learned routes:
show bgp 8.8.8.0/24 detail          ! Community: 65000:100
show bgp community 65000:100        ! list everything tagged "from ISP-A"
```

> **Can I explain it?** Why can't I just set a high local-pref to pull inbound traffic in via
> ISP-A? (Local-pref is *ours* — it never leaves our AS. Inbound is decided by *other* ASes, who
> only see attributes we send them, like AS-path length.)

---

## Phase 6 — Filtering, max-prefix, RPKI, and failover

**Objective:** make the edge safe, then *prove* multihoming works by failing a provider.

**Inbound allow-list.** `ISP-A-IN`/`ISP-B-IN` accept only expected prefixes and explicitly **drop
our own aggregate if it comes back** (a classic loop/leak symptom). In production this is where
you'd also filter bogons and martians. Without an inbound policy, IOS-XR drops everything anyway —
but a deliberate allow-list is the engineer's version, not the accident.

**Max-prefix.** Each eBGP neighbor has `maximum-prefix 1000 80` — tear the session down (or warn)
if a neighbor suddenly sends more prefixes than expected. This is your seatbelt against a peer
fat-fingering a full-table leak into your edge.

**RPKI origin validation** ([`docs/RPKI.md`](docs/RPKI.md)). RPKI lets you check that the AS
originating a prefix is *authorized* to (via signed ROAs), and drop **Invalid** routes — stopping
basic prefix hijacks. It needs an external validator (Routinator / rpki-client), so the config is
included **commented** in `R1.txt`/`R4.txt`; enable it when you have a validator reachable.

```
! sketch — see docs/RPKI.md for the full setup:
router bgp 65000
 rpki server 192.0.2.250
  transport tcp port 3323
 address-family ipv4 unicast
  bgp bestpath origin-as use validity
```

<details>
<summary><b>► Configure Phase 6</b> — tighten inbound to an allow-list + add max-prefix (R1, R4)</summary>

> This replaces the open `else pass` from Phase 5 with an explicit **allow-list** and lands the
> policies exactly on the shipped `configs/`. RPKI stays commented until you have a validator.

```
! ===================== R1 (allow-list + max-prefix on ISP-A) =====================
prefix-set ISP-A-ALLOWED-IN
  8.8.8.0/24,
  203.0.113.0/24
end-set
!
route-policy ISP-A-IN
  if destination in OUR-AGGREGATE then
    drop                                          ! never accept our own space back
  elseif destination in ISP-A-ALLOWED-IN then
    set local-preference 200
    set community (65000:100) additive
    pass
  else
    drop                                          ! Phase 6: explicit allow-list, drop the rest
  endif
end-policy
!
router bgp 65000
 neighbor 100.64.1.2
  address-family ipv4 unicast
   maximum-prefix 1000 80                         ! seatbelt vs a full-table leak
  !
 !
!
commit
!
clear bgp ipv4 unicast 100.64.1.2 soft in
```

```
! ===================== R4 (allow-list + max-prefix on ISP-B) =====================
prefix-set ISP-B-ALLOWED-IN
  8.8.8.0/24,
  198.51.100.0/24
end-set
!
route-policy ISP-B-IN
  if destination in OUR-AGGREGATE then
    drop
  elseif destination in ISP-B-ALLOWED-IN then
    set local-preference 100
    set community (65000:200) additive
    pass
  else
    drop
  endif
end-policy
!
router bgp 65000
 neighbor 100.64.2.2
  address-family ipv4 unicast
   maximum-prefix 1000 80
  !
 !
!
commit
!
clear bgp ipv4 unicast 100.64.2.2 soft in
```

> **RPKI (optional):** the `configs/R1.txt` and `R4.txt` carry the `rpki server` block **commented
> out**. Uncomment it once a validator (Routinator / rpki-client) is reachable, then drop `Invalid`
> routes with a policy matching `validation-state is invalid`. See [`docs/RPKI.md`](docs/RPKI.md).

</details>

**Failover test (the payoff).**

```
! Start a continuous test toward the dual-homed prefix (from R4 or a host behind us):
ping 8.8.8.1 count 100000

! Now drop the preferred upstream — shut R1's eBGP link to ISP-A:
! (on R1)  interface GigabitEthernet0/0/0/0 -> shutdown

! Watch convergence:
show bgp 8.8.8.0/24            ! ISP-A path gone; only ISP-B (LP 100) remains -> best
show route 8.8.8.0/24         ! next-hop now points toward R4/ISP-B
```

Expected: the ISP-A path withdraws, the ISP-B path becomes best AS-wide, traffic re-routes out R4.
Un-shut the link and it reverts to ISP-A (local-pref 200 wins again). **That** is multihoming —
and you just proved it instead of hoping.

> **Can I explain it?** During the failover, *which step of the best-path algorithm* picked the
> ISP-B path once ISP-A's was gone? (With ISP-A withdrawn there's only one path left — but trace
> why it wasn't chosen *before*: local-preference, step 2.)

---

## What this lab builds toward

| You practiced... | At work this is... |
|------------------|--------------------|
| local-pref / weight | Choosing transit vs peering for outbound traffic, by cost |
| AS-path prepend / communities | Inbound traffic engineering; signaling policy to upstreams |
| Route reflectors | Scaling iBGP in a real SP/large-enterprise core |
| Filtering / max-prefix / RPKI | Edge security; passing peering audits; not being the next BGP-leak headline |
| Failover testing | Proving redundancy before an outage does it for you |

Next: automate the verification — see [`../../Automation/python/check_bgp.py`](../../Automation/python/check_bgp.py),
which checks every session in this lab is Established. Then read the
[Learning Roadmap](../../LEARNING-ROADMAP.md) for where to go deeper.
