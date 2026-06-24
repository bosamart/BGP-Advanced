# Concepts — Advanced BGP

The theory behind the lab. Read this once before Phase 2, then again after Phase 6 — it reads
differently when you've seen it work.

## What BGP is (and isn't)

BGP is a **path-vector** protocol. Unlike IS-IS/OSPF (which compute shortest paths from a full
map of the topology), BGP routers exchange *reachability* plus a *list of attributes* per prefix,
and apply **policy** to pick the best path and decide what to re-advertise. BGP is not trying to
find the shortest path — it's trying to enforce *business and operational policy* between networks.

- **IGP (IS-IS/OSPF):** "how do I get across *my* network?" — fast, metric-based, inside one AS.
- **BGP:** "how do I reach *other* networks, and whose rules apply?" — policy-based, between ASes.

## eBGP vs iBGP

| | eBGP | iBGP |
|---|------|------|
| Between | Different ASes | Same AS |
| TTL | 1 by default (directly connected) | Multihop (loopback to loopback) |
| AS-path | Prepends local AS on advertise | Unchanged |
| Re-advertises to... | Everyone | **Not** to other iBGP peers (loop prevention) |
| Next-hop on advertise | Self (the eBGP link) | **Unchanged** unless `next-hop-self` |

The two rules in bold are the source of most "my route is missing" confusion:

1. **iBGP non-transitivity** → you need a full mesh, or route reflectors (Phase 3).
2. **iBGP next-hop unchanged** → eBGP-learned next-hops aren't in the IGP → use `next-hop-self`
   on the edge routers (Phase 2).

## Attributes you'll actually touch

| Attribute | Scope | Direction it controls | Set with |
|-----------|-------|------------------------|----------|
| Weight | Local to one router | Outbound (that router) | neighbor `weight` / route-policy |
| Local-preference | Whole AS (iBGP) | **Outbound** (AS-wide) | route-policy `set local-preference` |
| AS-path (prepend) | Leaves the AS | **Inbound** | route-policy `prepend as-path` |
| MED | To one neighbor AS | Inbound (weak hint) | route-policy `set med` |
| Community | A label, carried with the route | Drives other policy | route-policy `set community` |
| Origin | IGP/EGP/incomplete | Tie-break | rarely set by hand |

**Mental model for traffic steering:**
- Control **outbound** (your traffic leaving) with **local-preference** — it's yours, it's AS-wide, it's deterministic.
- *Influence* **inbound** (traffic coming to you) with **AS-path prepend** (and sometimes MED/communities) — you're asking *other* networks nicely, because the decision is theirs.

## Route reflectors (Phase 3)

A full iBGP mesh needs *n(n-1)/2* sessions — fine for 4 routers, impossible for 200. A **route
reflector** is an iBGP speaker allowed to *re-advertise* (reflect) iBGP routes, so clients only
need a session to the RR(s), not to each other.

- **Clients** peer the RR(s) only. **Non-client** iBGP peers (e.g. RR↔RR) behave normally.
- Loop prevention uses two attributes added by the RR: **originator-id** (who first injected the
  route into the AS) and **cluster-list** (which RR clusters it passed through). See them with
  `show bgp <prefix> detail`.
- **Redundancy:** give clients two RRs. Keeping a distinct cluster-id per RR preserves path diversity.

## Origin validation / RPKI (Phase 6)

By default BGP trusts whoever announces a prefix — the root cause of prefix hijacks and fat-finger
leaks. **RPKI** adds cryptographic proof of *which AS is allowed to originate which prefix* (ROAs),
so you can mark routes Valid / Invalid / NotFound and drop Invalids. Details in [`RPKI.md`](RPKI.md).

## Why this lab uses the diamond

Two equal paths from each edge to each RR (R1→R2 / R1→R3, etc.) plus a cross-link means: redundant
RR reachability, a real second path for failover, and a topology identical to the MPLS-TE/SRv6
labs so you compare *technologies*, not maps.
