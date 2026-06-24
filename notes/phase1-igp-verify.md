# Phase 1 — IS-IS Underlay: Verification Log

**Date:**
**Objective:** All four core loopbacks reachable via IS-IS L2 before any BGP. (ISP links are NOT in IS-IS.)
**Result:** ⬜ Not yet run

---

## Commands run (per core router)

```
show isis neighbors
show isis database
show route isis
ping 4.4.4.4 source 1.1.1.1
```

## Expected adjacency map

| Router | Neighbors (interface) | Count |
|--------|------------------------|-------|
| R1 | R2 (Gi0/0/0/1), R3 (Gi0/0/0/3) | 2 |
| R2 | R1 (Gi0/0/0/0), R3 (Gi0/0/0/1), R4 (Gi0/0/0/3) | 3 |
| R3 | R1 (Gi0/0/0/2), R2 (Gi0/0/0/0), R4 (Gi0/0/0/4) | 3 |
| R4 | R2 (Gi0/0/0/2), R3 (Gi0/0/0/3) | 2 |

Expected: R1 reaches 2.2.2.2 / 3.3.3.3 / 4.4.4.4 (ECMP to 4.4.4.4 via R2 and R3). ISP links absent from `show route isis`.

## Captured output

```
(paste here)
```

## What I learned / gotchas

-
