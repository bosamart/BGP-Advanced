# RPKI Origin Validation

## The problem

By default BGP trusts whatever AS announces a prefix. If someone announces `8.8.8.0/24` from the
wrong AS — by mistake (a leak) or on purpose (a hijack) — neighbors may believe it and traffic gets
misrouted or blackholed. This has caused real, large internet outages.

## What RPKI does

**RPKI (Resource Public Key Infrastructure)** lets the legitimate holder of a prefix publish a
signed **ROA (Route Origin Authorization)**: "AS *X* is authorized to originate prefix *P* up to
length *L*." Your router checks received routes against ROAs and labels each path:

| State | Meaning | Typical action |
|-------|---------|----------------|
| **Valid** | A ROA covers it and the origin AS + max-length match | Accept, maybe prefer |
| **Invalid** | A ROA exists but origin AS or prefix length is wrong | **Drop** (or strongly depref) |
| **NotFound** | No ROA covers it | Accept (most of the table is still unsigned) |

Origin validation only checks **the origin AS of a prefix** — it does *not* validate the whole
AS-path. Path validation is a separate, newer effort (ASPA/BGPsec).

## Why it's commented out in this lab

The router doesn't compute validation itself — it talks **RTR (RPKI-to-Router protocol)** to an
external **validator** that has downloaded and cryptographically checked the global ROA set.
Common validators: **Routinator** (NLnet Labs), **rpki-client**, FORT. You need one reachable from
the lab before the config does anything, so `R1.txt`/`R4.txt` ship it commented.

## Enabling it (IOS-XR)

1. Run a validator (e.g. Routinator in a container) reachable from the routers, serving RTR
   (default-ish port 3323).
2. Point BGP at it and let validity influence best-path:

```
router bgp 65000
 rpki server 192.0.2.250
  transport tcp port 3323
 !
 address-family ipv4 unicast
  bgp bestpath origin-as use validity      ! factor validity into best-path
 !
```

3. To **hard-drop Invalids** at ingress, match validation state in your inbound policy:

```
route-policy ISP-A-IN
  if validation-state is invalid then
    drop
  endif
  ... (rest of the existing policy) ...
end-policy
```

> Exact keywords vary by IOS-XR release — verify against your image's command reference. The
> concept (server → states → drop Invalid) is stable; the syntax drifts.

## Verify

```
show bgp rpki server                 ! RTR session to the validator up?
show bgp rpki table                  ! ROAs learned
show bgp 8.8.8.0/24                   ! origin-AS validity flag on the path
```

## Why a real engineer cares

Dropping RPKI-Invalid routes is now table stakes at the edge — peering partners and audits expect
it, and it's one of the few BGP security wins that's cheap and effective. Knowing *what it does and
doesn't cover* (origin only, not full path) is exactly the depth that separates an engineer from a
template-paster.
