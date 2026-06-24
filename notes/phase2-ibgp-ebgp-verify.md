# Phase 2 — iBGP full mesh + eBGP multihoming: Verification Log

**Date:**
**Objective:** eBGP to both ISPs up; iBGP full mesh up; 172.16/16 advertised to both ISPs; 8.8.8.0/24 learned on all core routers with a resolvable next-hop.
**Result:** ⬜ Not yet run

---

## Commands run

```
show bgp ipv4 unicast summary          ! sessions Established (State/PfxRcd numeric)
show bgp 8.8.8.0/24                      ! learned via both ISPs
show bgp 172.16.0.0/16
show route 8.8.8.0/24                    ! installed; next-hop resolvable?
! on ISP-A / ISP-B:
show bgp 172.16.0.0/16                   ! our aggregate arrived
```

## Expected sessions (Phase 2 = full mesh)

| Router | eBGP | iBGP peers (full mesh) |
|--------|------|------------------------|
| R1 | ISP-A (100.64.1.2) | R2, R3, R4 |
| R2 | — | R1, R3, R4 |
| R3 | — | R1, R2, R4 |
| R4 | ISP-B (100.64.2.2) | R1, R2, R3 |

## Gotchas to watch for (write what actually bit you)

- **Missing `next-hop-self`** on R1/R4 → 8.8.8.0/24 in `show bgp` but not installed (next-hop 100.64.x unreachable).
- **No route-policy on eBGP** → IOS-XR silently drops all routes in/out.
- **`network 172.16.0.0/16` with no matching RIB route** → not originated. (Static to Null0 fixes it.)

## Captured output

```
(paste here)
```

## Can I explain it?
Will R2 re-advertise an iBGP-learned route to R3? → **No** (iBGP non-transitivity — the reason for the full mesh).
