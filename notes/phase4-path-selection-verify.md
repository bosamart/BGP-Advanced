# Phase 4 — Path Selection (local-pref vs weight): Verification Log

**Date:**
**Objective:** Make AS 65000 prefer ISP-A to reach 8.8.8.0/24 using local-preference 200. Confirm even R4 (attached to ISP-B) prefers the path via R1.
**Result:** ⬜ Not yet run

---

## Commands run

```
! R1:
show bgp 8.8.8.0/24                 ! best via ISP-A (LP 200)
! R4 (attached to ISP-B!):
show bgp 8.8.8.0/24                 ! best via 1.1.1.1 iBGP, LP 200
show bgp 8.8.8.0/24 detail          ! read the "best path, reason: Local Preference" line
```

## Best-path walk for 8.8.8.0/24 (predict before you look)

| Step | ISP-A path | ISP-B path | Winner |
|------|-----------|-----------|--------|
| Weight | 0 | 0 | tie |
| **Local-pref** | **200** | 100 | **ISP-A** ✋ decided here |

## Experiment: swap to weight, see the AS disagree

- Replace local-pref with `weight 500` on R1's ISP-A neighbor.
- R1 still prefers ISP-A, but **R4 reverts to its own ISP-B exit** (weight isn't advertised).
- Restore local-pref. This is the whole point of weight (local) vs local-pref (AS-wide).

## Captured output

```
(paste here)
```

## Can I explain it?
Why local-pref for AS-wide outbound, not weight? → Local-pref is carried over iBGP to every router; weight never leaves the box.
