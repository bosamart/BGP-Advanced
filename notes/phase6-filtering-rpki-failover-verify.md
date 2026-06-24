# Phase 6 — Filtering, max-prefix, RPKI, failover: Verification Log

**Date:**
**Objective:** Lock down the edge (allow-list inbound, max-prefix, optional RPKI) and PROVE multihoming with a failover test.
**Result:** ⬜ Not yet run

---

## Filtering + max-prefix

```
show bgp neighbor 100.64.1.2          ! confirm policies + maximum-prefix applied
show bgp 172.16.0.0/16                 ! our own aggregate must NOT be accepted back from an ISP
```

Expected: only allow-listed prefixes (8.8.8.0/24, 203.0.113.0/24 from A; 8.8.8.0/24, 198.51.100.0/24 from B) accepted; our aggregate dropped inbound; max-prefix 1000/80% set.

## RPKI (optional — needs an external validator)

```
show bgp rpki server                  ! RTR session up?
show bgp rpki table                   ! ROAs loaded
show bgp 8.8.8.0/24                     ! validity flag
```

See ../docs/RPKI.md. Skip if no validator running.

## Failover test (the payoff)

```
ping 8.8.8.1 count 100000               ! continuous, from R4 or a host behind us
! drop preferred upstream — on R1:
interface GigabitEthernet0/0/0/0
 shutdown
! observe:
show bgp 8.8.8.0/24                     ! ISP-A path gone; ISP-B (LP100) now best
show route 8.8.8.0/24                   ! next-hop now toward R4/ISP-B
! restore:
interface GigabitEthernet0/0/0/0
 no shutdown                            ! reverts to ISP-A (LP200)
```

Record: convergence time, packets lost during the switch, whether it reverted cleanly.

## Captured output

```
(paste here)
```

## Can I explain it?
Before failover, which step chose ISP-A? → Local-preference (step 2). After failover only one path remained, so there was nothing to choose.
