# Phase 3 — Route Reflectors: Verification Log

**Date:**
**Objective:** Migrate the iBGP full mesh to RR. R2 & R3 reflect; R1 & R4 are clients of both. After removing the R1↔R4 mesh session, every router still learns every route.
**Result:** ⬜ Not yet run

---

## Migration order (do it safely, no outage)

1. On R2 & R3: add `route-reflector-client` for R1 and R4; add RR↔RR peer (R2↔R3).
2. Confirm R1 & R4 still learn 8.8.8.0/24 and each other's routes **via the RRs**.
3. Only then remove the old direct R1↔R4 iBGP session.

## Commands run

```
show bgp ipv4 unicast summary          ! R1/R4 now have 2 iBGP sessions (to RRs)
show bgp 8.8.8.0/24                      ! still present after mesh removed
show bgp 8.8.8.0/24 detail               ! look for Originator-id and Cluster-list
```

## Expected

- R1/R4: 2 iBGP sessions (to 2.2.2.2 and 3.3.3.3), no longer peering each other.
- Reflected routes carry **Originator-id** (e.g. 1.1.1.1) and **Cluster-list** (the RR's id).
- No route loss vs Phase 2.

## Captured output

```
(paste here)
```

## Can I explain it?
If both RRs shared one cluster-id, what's lost? → **Path diversity** (an RR drops a reflected route carrying its own cluster-id).
