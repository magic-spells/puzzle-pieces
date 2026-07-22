---
name: v0.1.0 — first publish (registry + add CLI)
status: planned
connections:
  - DOC-REGISTRY
  - DECISION-COPY-IN-DISTRIBUTION
  - DECISION-DOCS-DEMO-SPLIT
---

# v0.1.0 — first publish (registry + add CLI)

Theme: turn the finished registry into something an external Puzzle app can actually pull from — publish the package that carries the registry + [[FEATURE-ADD-CLI]]. Nobody imports the package at runtime; it exists to host the CLI and the copyable registry ([[DECISION-COPY-IN-DISTRIBUTION]]).

## Current state (not yet shipped)

- **Not on npm.** The 41 pieces are built and demo-verified, but there is no root `package.json` and no CLI yet.
- **`@magic-spells/puzzle` (the framework) is itself unpublished** — the demo builds against the LOCAL sibling Go binary (`../../puzzle/puzzle`) via a machine-local lockfile. Puzzle must publish before puzzle-pieces can de-localize.
- **`@magic-spells/morph-engine` IS published** — the one real npm dependency morph pieces declare.

## Publish plan

1. Puzzle publishes first (its own release ceremony).
2. Then the puzzle-pieces CLI publishes with the **registry embedded in the tarball**: `files: ["registry", "bin"]`. No `main`/import surface — the package is a CLI host, not a library.
3. De-localize the demo (point at published `@magic-spells/puzzle` instead of the local binary).

## Open items

- **License decision pending** — MIT likely, not yet chosen; no `LICENSE` file in the repo.
- **[[FEATURE-ADD-CLI]] must be built** — it is the reason to publish at all.
- **README count is stale** — says "40 pieces", registry has 41 (see [[DOC-REGISTRY]]).
- **Public docs home undecided** — the demo docs site vs the future Astro pages on `../magic-spells-site` ([[DECISION-DOCS-DEMO-SPLIT]]).

## Upgrade notes

None yet — this is the first release. Consumers install nothing at runtime; they run the CLI (or copy by hand) and merge `pieces.css` tokens once.
