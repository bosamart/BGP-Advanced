# Phase 5 — Communities + AS-path prepend: Verification Log

**Date:**
**Objective:** Steer INBOUND traffic to enter via ISP-A by prepending our AS on R4→ISP-B. Tag learned routes with communities (65000:100 from ISP-A, 65000:200 from ISP-B).
**Result:** ⬜ Not yet run

---

## Commands run

```
! ISP-B — our aggregate should now have a longer AS-path:
show bgp 172.16.0.0/16              ! AS-path 65000 65000 65000
! ISP-A — still short:
show bgp 172.16.0.0/16              ! AS-path 65000
! R1 — community tag on learned routes:
show bgp 8.8.8.0/24 detail          ! Community: 65000:100
show bgp community 65000:100        ! everything "from ISP-A"
show bgp community 65000:200        ! everything "from ISP-B"
```

## Expected

| Where | What |
|-------|------|
| ISP-B view of 172.16/16 | AS-path = 65000 65000 65000 (2x prepend) |
| ISP-A view of 172.16/16 | AS-path = 65000 |
| R1 routes from ISP-A | tagged 65000:100 |
| R4 routes from ISP-B | tagged 65000:200 |

Result: the wider internet reaches 172.16/16 via ISP-A (shorter AS-path). Inbound TE achieved.

## Captured output

```
(paste here)
```

## Can I explain it?
Why prepend (not local-pref) for inbound? → Local-pref never leaves our AS; other networks only honor what we send them, e.g. AS-path length.
