# Advanced BGP Lab Notes

## Lab Goals
- Stop pasting BGP templates — be able to **explain why each route wins**
- Build a dual-homed AS and steer traffic in (prepend/communities) and out (local-pref)
- Scale iBGP with route reflectors; secure the edge (filtering, max-prefix, RPKI)
- Prove multihoming with a real failover test

## Topics Covered
- [ ] Phase 1 — IS-IS underlay (loopback reachability for iBGP)
- [ ] Phase 2 — iBGP full mesh + eBGP multihoming
- [ ] Phase 3 — Route reflectors (migrate off the full mesh)
- [ ] Phase 4 — Path selection (local-pref vs weight)
- [ ] Phase 5 — Communities + AS-path prepend
- [ ] Phase 6 — Filtering, max-prefix, RPKI, failover

## Lab Topology
> Same diamond as the MPLS-TE/SRv6 labs. AS 65000 = R1–R4. R1↔ISP-A (AS65100),
> R4↔ISP-B (AS65200). R2 & R3 are redundant route reflectors; R1 & R4 are RR clients.
> See ../diagrams/topology.md

## How to use these notes
Each phase note is a **verification log** — fill in the Date, paste what you actually saw, and
record gotchas. The expected results and commands are pre-filled so you know what "good" looks
like. Don't mark a phase ✅ until the "Can I explain it?" question in the README is answered.

## Related
- Roadmap: ../../../LEARNING-ROADMAP.md
- Automation check: ../../../Automation/python/check_bgp.py (verifies every session is Established)
- MPLS-TE lab (same topology): ../../MPLS-TE/

## Session Log
| Date | Phase | Topic | Outcome |
|------|-------|-------|---------|
|  | Phase 1 | IS-IS underlay |  |
|  | Phase 2 | iBGP mesh + eBGP |  |
|  | Phase 3 | Route reflectors |  |
|  | Phase 4 | Path selection |  |
|  | Phase 5 | Communities + prepend |  |
|  | Phase 6 | Filtering / RPKI / failover |  |
