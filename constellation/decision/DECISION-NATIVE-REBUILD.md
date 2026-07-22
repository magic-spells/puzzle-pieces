---
name: Native .pzl rebuilds, not web-component wrappers
status: built
connections:
  - DOC-REGISTRY
  - DECISION-CONFIG-FIRST-API
  - DECISION-COPY-IN-DISTRIBUTION
---

# Native .pzl rebuilds, not web-component wrappers

## Context

The `@magic-spells` ecosystem already has ~15 mature web components (dialog-panel, select-dropdown, dropdown-panel, tab-group, quantity-input, collapsible-content, notification-stack, range-slider, date-picker, panel-stack, …). The tempting shortcut is to wrap them as Puzzle pieces — a thin layer over existing primitives.

## Decision

Every piece is a **native `.pzl` rebuild** — real Puzzle components (`<Select …/>`, capitalized tags) compiling to plain semantic HTML with ARIA + Tailwind. **No custom elements, no `customElements.define`, no `<select-dropdown-trigger>` tags, no `<styles>` blocks.** The transferable IP is the *behavior design* (keyboard nav, ARIA state machines, gesture handling), re-expressed against Puzzle's primitives — not the code.

## Alternatives rejected

- **Wrap the existing web components** — rejected. Puzzle's vdom owns the DOM; self-mutating web components (state attributes, injected backdrops) fight reconciliation. The sanctioned escape hatch (`island`) freezes children, which would kill reactivity *inside* dialog/panel content — a dealbreaker for overlays specifically.
- **Share code between the two lineages** — rejected. The web-component repos keep living for Shopify themes and other non-Puzzle contexts.

## Consequences

- Rewrites get dramatically smaller: Puzzle gives for free what the web components hand-roll — declarative listeners, `@keydown:escape`/`:enter` modifiers, `animations = { in, out }` (WAAPI), reactive state, `<Slot/>`/named slots, morph.
- The behavioral notes worth preserving per piece live in each file's header comment and `piece.json`, not here (see [[DOC-REGISTRY]]).
- Config-first APIs follow from having no cross-component context ([[DECISION-CONFIG-FIRST-API]]).
