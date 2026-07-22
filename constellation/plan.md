---
name: Puzzle Pieces
connected_repos:
  - name: puzzle
    path: ../puzzle
    description: >-
      The Puzzle framework — the .pzl compiler + runtime pieces are built for. Has its own
      constellation plan.
  - name: morph-engine
    path: ../morph-engine
    description: >-
      @magic-spells/morph-engine — the one published npm dep morph pieces (Select, Dialog,
      DatePicker) declare.
  - name: magic-spells-site
    path: ../magic-spells-site
    description: Astro marketing site — the future home of long-form public docs (DECISION-DOCS-DEMO-SPLIT).
connections:
  - DIAGRAM-TOPOLOGY
  - DOC-REGISTRY
  - DOC-DEMO-DOCS-SITE
  - FEATURE-ADD-CLI
  - RELEASE-V0-1-0
  - DECISION-COPY-IN-DISTRIBUTION
  - DECISION-NATIVE-REBUILD
  - DECISION-REGISTRY-SHAPED-REPO
  - DECISION-DOCS-DEMO-SPLIT
  - DECISION-CONFIG-FIRST-API
---

# Puzzle Pieces

A **copy-in** UI component registry for the [Puzzle framework](../puzzle):
41 Tailwind-styled, accessible, morph-aware `.pzl` pieces distributed as **source you
copy into a consumer app**, not packages you install. This card is the map; the
always-load rules, conventions, and hard-won gotchas live in `CLAUDE.md` (read it every
session) and are not duplicated here.

## The whole idea in one paragraph

`.pzl` single-file components can't ship via npm (the Puzzle compiler prunes
`node_modules` and `.pzl` isn't resolvable there; Tailwind only scans `app/`), so pieces
land in the consumer's own `app/components/ui/` where their `puzzle build` compiles them
and their Tailwind scan picks up the classes. Everything downstream of that choice — the
registry shape, the CLI, the native rebuilds — follows from it. See
[[DECISION-COPY-IN-DISTRIBUTION]].

## How the repo fits together

- [[DIAGRAM-TOPOLOGY]] — registry → demo / CLI → consumer at a glance.
- [[DOC-REGISTRY]] — `registry/` is the source of truth: `piece.json` manifests,
  the generated `registry.json` index, `theme/pieces.css` tokens, `lib/` helpers.
- [[DOC-DEMO-DOCS-SITE]] — the in-repo `demo/` Puzzle app (port 3070): the
  docs shell + dev/integration harness that consumes copies of the pieces.
- [[FEATURE-ADD-CLI]] — the planned `npx … add <piece>` resolver+copier (next milestone).
- [[RELEASE-V0-1-0]] — the first-publish milestone + current publishing state + open items.

## The settled decisions (don't re-litigate without Cory)

- [[DECISION-COPY-IN-DISTRIBUTION]] — copy-in, not npm import (npm rejected for v1).
- [[DECISION-NATIVE-REBUILD]] — native `.pzl` rebuilds, not wrappers over the existing
  `@magic-spells/*` web components; no custom elements.
- [[DECISION-REGISTRY-SHAPED-REPO]] — registry-shaped from day one so the CLI stays a copier.
- [[DECISION-CONFIG-FIRST-API]] — config-first APIs, because Puzzle has no cross-component context.
- [[DECISION-DOCS-DEMO-SPLIT]] — docs in the demo app now, Astro on `magic-spells-site` later.

## Current state

All 41 pieces are built, compile-verified against the real compiler, and demo-verified
(waves 1–6 + the docs-site refactor — see git history / the soon-to-be-deleted `PLAN.md`
for the build log). Not yet published; the [[FEATURE-ADD-CLI]] is the next milestone.
Sibling repos are linked in `connected_repos` above (`repo:` selector targets each).
