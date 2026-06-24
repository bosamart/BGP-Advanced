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

## Phase 1 — IS-IS underlay

**Objective:** every core loopback reachable before any BGP. iBGP peers sit on loopbacks, so
the loopbacks must be reachable first — that's the IGP's only job here.

**Why an IGP under BGP at all?** iBGP sessions are built between loopbacks that may be several
hops apart. BGP doesn't find those loopbacks itself — the IGP (IS-IS) does. The IGP carries
*infrastructure* routes (loopbacks, links); BGP carries *everything else* (customer/internet
prefixes). Keep the two jobs separate and the design stays sane.

```
router isis CORE
 is-type level-2-only
 net 49.0001.0000.0000.0001.00
 address-family ipv4 unicast
  metric-style wide
 ...
```

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

```
! eBGP (R1 -> ISP-A) — IOS-XR REQUIRES a route-policy or routes are silently dropped
 neighbor 100.64.1.2
  remote-as 65100
  address-family ipv4 unicast
   route-policy ISP-A-IN in
   route-policy ISP-A-OUT out
!
! iBGP full mesh (Phase 2): R1 peers R2, R3, AND R4 on loopbacks
 neighbor 4.4.4.4
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
   next-hop-self
```

> The committed [`configs/`](configs/) show the **Phase 3** end-state (RR overlay), where R1/R4
> peer only the RRs. For Phase 2, temporarily also peer R1↔R4 directly to experience the full mesh.

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

```
! On R2 (and mirror on R3):
 neighbor 1.1.1.1
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
   route-reflector-client          ! R1 is a client
 !
 neighbor 4.4.4.4
  ... route-reflector-client       ! R4 is a client
 !
 neighbor 3.3.3.3
  ... (no client keyword)          ! the other RR — a non-client peer
```

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

```
route-policy ISP-A-IN
  ...
  set local-preference 200       ! higher wins -> ISP-A preferred AS-wide
  ...
```

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

```
route-policy ISP-B-OUT
  if destination in OUR-AGGREGATE then
    prepend as-path 65000 2        ! add our ASN 2 extra times
    pass
  endif
```

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
