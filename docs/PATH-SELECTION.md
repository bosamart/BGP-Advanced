# BGP Best-Path Selection — read until it's automatic

When BGP has more than one path to a prefix, it walks this list **top to bottom** and stops at the
first step that breaks the tie. Memorize the order; everything you do to steer traffic is just
choosing *which step* to win at.

## The algorithm (Cisco IOS-XR)

| # | Rule | Prefer | Notes |
|---|------|--------|-------|
| 0 | Next-hop reachable? | (else path is invalid) | If the next-hop isn't resolvable, the path can't win at all |
| 1 | **Weight** | Highest | Cisco-only, **local to one router**, never advertised |
| 2 | **Local-preference** | Highest | **AS-wide** (carried over iBGP). The main outbound lever |
| 3 | Locally originated | Local `network`/aggregate/redistribute | Prefer routes we originated |
| 4 | **AS-path length** | Shortest | The main **inbound** lever (via prepend) |
| 5 | Origin code | IGP < EGP < Incomplete | `network` = IGP; redistributed = incomplete |
| 6 | **MED** | Lowest | Only compared between paths from the **same** neighbor AS (by default) |
| 7 | eBGP vs iBGP | eBGP > iBGP | Prefer a path learned from outside |
| 8 | IGP metric to next-hop | Lowest | "Hot-potato" — exit via the closest egress |
| 9 | Oldest eBGP path | Oldest | Stability tie-break |
| 10 | Lowest router-id | Lowest | Final deterministic tie-break |
| 11 | Lowest neighbor address | Lowest | Last resort |

> A handy mnemonic for 1–8: **W**eigh **L**ocal **A**S **A**bsurdly **L**ong **M**ovies, **E**ven **I**ntermissions
> (Weight, Local-pref, AS-path, ... actually just memorize the table — but the first two are what you'll use daily).

## How this lab exercises it

**8.8.8.0/24 is dual-homed** (originated by both ISP-A and ISP-B). Walk the algorithm:

- Step 0: both next-hops resolvable (after `next-hop-self`). Tie.
- Step 1: weight default on both. Tie.
- Step 2: **local-preference** — ISP-A path = 200, ISP-B path = 100. **ISP-A wins here.** Done.

Because we decided it at step 2 (an AS-wide attribute), *every* router in AS 65000 agrees — even
R4, which is directly attached to ISP-B, prefers reaching 8.8.8.0/24 via R1.

Now imagine we'd used **weight** on R1 instead (step 1, local-only):
- On R1, weight wins → R1 prefers ISP-A. Good.
- On R4, R1's weight was never advertised → R4 still sees LP 100 both ways → step 4/7 decide →
  R4 prefers its own eBGP path out ISP-B. **The AS disagrees with itself.** That's why weight is
  for one-router tweaks, not AS-wide policy.

**Inbound (our 172.16.0.0/16):** other networks run *their own* copy of this algorithm on the
paths *we* advertise. The only attribute we send that they'll honor cheaply is **AS-path length**
(step 4) — so R4 prepends, making the ISP-B path to us longer, and inbound arrives via ISP-A.

## Reading `show bgp <prefix> detail`

Look for the line that names the winner and the reason, e.g.:

```
  Path #1: Received by speaker 0 ... Best
    65100
    100.64.1.2 from 1.1.1.1 ...
      Local Preference: 200, ...
      ... best path, reason: Local Preference
```

Train yourself to predict the "reason" *before* you read it. When you can call it every time,
you understand BGP path selection.
