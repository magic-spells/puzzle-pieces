---
name: The add CLI — resolver + copier
status: planned
connections:
  - DOC-REGISTRY
  - DECISION-REGISTRY-SHAPED-REPO
  - DECISION-COPY-IN-DISTRIBUTION
---

# The add CLI — resolver + copier

**Status: not built — the next milestone.** `npx @magic-spells/puzzle-pieces add <piece…>` is the automated copy-in path for external apps. Because the repo is registry-shaped ([[DECISION-REGISTRY-SHAPED-REPO]]) the CLI is a pure resolver + file-copier, never a build tool.

## Behavior

1. Read `registry.json`; resolve the requested pieces **plus their `registryDependencies` transitively** (dedupe).
2. Copy each file to its manifest `targetDir` — pieces → `app/components/ui/`, `lib/*.js` → `app/lib/`.
3. **Refuse to overwrite an existing target file unless `--overwrite`.**
4. **Print — never auto-run — the npm install** command for the accumulated `dependencies` (e.g. `@magic-spells/morph-engine`).
5. If the app's `styles.css` lacks the pieces tokens, **print the `pieces.css` merge snippet**.
6. **Never rewrite the user's files** — print snippets, following Puzzle's D3 precedent.

## Scope

- In: resolve/copy/refuse-overwrite/print-installs/print-tokens.
- Out (deferred, but designed for): stamp a piece-name + registry-version header in copied files, or a `pieces.lock`, to enable a future `diff`/`update` command. A later `puzzle add piece <name>` framework alias could delegate to `npx @magic-spells/puzzle-pieces` — keep the standalone CLI first.

## Acceptance

- `add date-picker` copies `DatePicker.pzl`, pulls `calendar` + `lib/date-math.js` transitively, prints the `@magic-spells/morph-engine` install line, and prints the tokens snippet if missing — touching no existing file without `--overwrite`.

## Notes

- Model on `puzzle add` / `puzzle generate` (`../puzzle/compiler/cmd/puzzle/generate.go`, `initcmd.go`, `compiler/internal/scaffold/templates/`).
- Local dev can point the registry fetch at this repo's path before anything is published.
- Ships as part of [[RELEASE-V0-1-0]], with the registry embedded in the tarball (`files: ["registry", "bin"]`).
