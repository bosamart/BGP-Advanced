# BGP Communities

A community is just a **tag** attached to a route — a 32-bit value usually written `ASN:value`
(e.g. `65000:100`). BGP doesn't *do* anything with most communities by itself; they're labels that
*your* (and others') route-policies match on. They let you express policy once as "what kind of
route is this" instead of re-listing prefixes in every policy.

## Why bother?

Without communities, every policy re-matches prefix lists — brittle and repetitive. With them you
tag a route **once** at the edge ("this came from ISP-A", "this is a customer route", "don't export
this") and every downstream policy just matches the tag. This is how real networks stay maintainable.

## Well-known communities (RFC 1997) — memorize these

| Community | Meaning |
|-----------|---------|
| `no-export` | Don't advertise outside the AS (to eBGP peers). Stays internal + to confederation. |
| `no-advertise` | Don't advertise to **any** peer at all. |
| `local-AS` / `no-export-subconfed` | Don't advertise outside the local sub-AS (confederations). |
| `internet` | Match-all (everything). |

`no-export` is the everyday one: tag your more-specific internal prefixes with it so they don't
leak to upstreams (you advertise only the aggregate outward).

## How this lab uses them

`ISP-A-IN` and `ISP-B-IN` tag every accepted route with where it came from:

```
route-policy ISP-A-IN
  ...
  set community (65000:100) additive    ! "learned via ISP-A"
  ...
route-policy ISP-B-IN
  ...
  set community (65000:200) additive    ! "learned via ISP-B"
```

`additive` means **add** the tag without wiping existing communities (omit it and you replace them
all — a common mistake). Now you can act on the source anywhere:

```
show bgp community 65000:100        ! everything learned via ISP-A
show bgp 8.8.8.0/24 detail          ! see "Community: 65000:100" on the route
```

## A common production pattern: customer-signaled policy

ISPs publish communities customers can *set* to ask for behavior — e.g. "tag `65000:80` and I'll
prepend once toward my upstreams", "tag `65000:666` and I'll blackhole it". The customer signals
intent with a tag; the provider's inbound policy matches it and acts. Sketch:

```
community-set CUST-PREPEND-1X
  65000:80
end-set
!
route-policy CUSTOMER-IN
  if community matches-any CUST-PREPEND-1X then
    prepend as-path 65000 1
  endif
  pass
end-policy
```

## Extended & large communities (know they exist)

- **Extended communities** — 64-bit; used by MP-BGP L3VPN for Route Targets (`route-target 100:1`)
  — you saw these in the MPLS-TE lab.
- **Large communities** (RFC 8092) — `ASN:func:value`, three 32-bit parts, made for 4-byte ASNs
  where the classic 16-bit `ASN:value` no longer fits.

Start with standard communities here; the others are the same idea with more room.
