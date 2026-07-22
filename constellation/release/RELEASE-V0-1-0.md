---
name: v0.1.0 — first publish (public registry + add CLI)
status: in-progress
connections:
  - DOC-REGISTRY
  - FEATURE-ADD-CLI
  - DECISION-COPY-IN-DISTRIBUTION
  - DECISION-DOCS-DEMO-SPLIT
---

# v0.1.0 — first publish (public registry + add CLI)

Theme: make the finished registry something an external Puzzle app can actually pull from. **Publishing = making this GitHub repo public** — no npm package. The shipped [[FEATURE-ADD-CLI]] (`puzzle add piece`, in the Puzzle Go CLI) defaults to `https://raw.githubusercontent.com/magic-spells/puzzle-pieces/main/registry`, so the flip to public activates the default registry URL directly. The earlier plan of an npm-hosted `npx @magic-spells/puzzle-pieces add` (registry embedded in a tarball) was superseded 2026-07-17; consumers by definition already have the `puzzle` CLI ([[DECISION-COPY-IN-DISTRIBUTION]]).

## Done

- All 85 pieces built, compile-verified, demo-verified; registry.json aggregated.
- `puzzle add piece` shipped and verified end-to-end against this registry.
- **MIT license** chosen and committed (`LICENSE`, Magic Spells, 2026).
- Pre-public sweep (2026-07-21): internal planning log `PLAN.md` deleted (rationale absorbed into these cards + CLAUDE.md), README rewritten around `puzzle add piece` with the true piece count, demo README de-scaffolded, competitor-library analogies removed from decision cards.

## Remaining

- Flip the GitHub repo public (after amending/squashing history so pre-sweep internals never appear in public history — the repo was already reduced to a single initial commit for this reason).
- **BLOCKER (in the puzzle repo, not here): the four platform binary packages are unpublished.** `@magic-spells/puzzle@0.1.0` is on npm and the demo is de-localized (registry-resolved dep, plain `puzzle dev`/`puzzle build` scripts, lockfile regenerated), but its optionalDependencies `@magic-spells/puzzle-{darwin-arm64,darwin-x64,linux-x64,linux-arm64}` 404 — they exist built-but-unpublished in `../puzzle/npm/`. Until they publish, `npm install && npm run build` yields "no prebuilt CLI binary available" for every external user (verified 2026-07-21; demo itself compiles clean with a globally installed puzzle 0.1.0).
- **Public docs home undecided** — the demo docs site vs the future Astro pages on `../magic-spells-site` ([[DECISION-DOCS-DEMO-SPLIT]]).
- Docs for the CLI flow + the `diff`/`update` command ([[FEATURE-ADD-CLI]] follow-ups).

## Upgrade notes

None — first release. Consumers install nothing at runtime; they run `puzzle add piece` (or copy by hand) and get `pieces.css` tokens copied in automatically.
