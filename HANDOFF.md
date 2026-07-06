# HANDOFF — BGP-Advanced

Quick-resume notes for the next session. The lab's own `README.md` is the source of truth.

## Status

🟡 **Built, not yet tested in EVE-NG.** Configs are written as cumulative final state
(`configs/R1–R4.txt`, `ISP-A.txt`, `ISP-B.txt`). README is now fully phase-by-phase.

## Latest change (2026-07-06)

Made the README **easier to configure phase by phase** + added the topology SVG:

- Added a **"How to build this — two paths"** section (speed-run vs phase-by-phase) plus the
  IOS-XR **candidate/commit** workflow and the `clear bgp ... soft in` re-evaluation note.
- Each phase now has a collapsible **► Configure** block containing *only that phase's per-device
  delta* + `commit`. The deltas were derived line-for-line from `configs/` and build up to the exact
  shipped final state (Phase 2 policies start as `pass`; Phases 4–6 tighten the same route-policies).
- Created `diagrams/topology.svg` — **dark-mode adaptive** (`@media prefers-color-scheme: dark`),
  showing the dual-ISP diamond, RR overlay (dashed iBGP), and addressing — and embedded it in the
  README Topology section (ASCII kept in a `<details>`).

## To push to GitHub

Repo **is** set up: `github.com/bosamart/BGP-Advanced` (branch `main`). Only the initial commit is
pushed so far. Edits made in a Cowork session land on disk but are **not** auto-committed — commit
and push from a local terminal:

```
git add -A
git commit -m "…"
git push
```

**Do not `git clone` into the project folder** from the sandbox (per CLAUDE.md — phantom entries,
no sync). Recreate/edit files with the Write/Edit tools, then commit locally.

## Gotchas baked into the lab

- **IOS-XR needs an in/out route-policy on every eBGP neighbor** or routes are silently dropped.
  Phase 2 uses a minimal `pass`; later phases replace it by name (re-pasting overwrites).
- **next-hop-self** on R1/R4 iBGP sessions — without it the ISP link next-hop is unresolvable.
- **Full mesh in Phase 2 is intentional**; Phase 3 removes the `R1↔R4` session (`no neighbor …`).
- Both RRs keep their **own cluster-id** (default = router-id) for path diversity.

## TODO

- [ ] Build the 6-node topology in EVE-NG and walk the phases; capture real `show` output into `notes/`.
- [ ] Tick the Results checklist in the README as each phase verifies.
- [ ] Confirm `docs/` referenced files exist: CONCEPTS, PATH-SELECTION, COMMUNITIES, RPKI.
- [ ] Once verified, publish to GitHub and add the repo link to CLAUDE.md.
