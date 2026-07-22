# puzzle-pieces

A component library built specifically for the [Puzzle framework](../puzzle/):
Tailwind-only, copy-into-your-project, morph-aware. This file is the living plan —
roadmap, conventions, and the spec notes for the `add` CLI. If you're a fresh session
picking this up, everything you need to build the next wave (or the CLI) is here.

## Why this exists, and why the copy-in model specifically

Cory has ~15 mature web components under `@magic-spells/*` (dialog-panel, select-dropdown,
dropdown-panel, tab-group, quantity-input, collapsible-content, notification-stack, …).
They run anywhere. puzzle-pieces is different: components built FOR Puzzle, distributed
by copying source into the user's app. That model is not an aesthetic choice
here; it's structurally required:

- **`.pzl` files cannot ship via node_modules.** The Puzzle compiler prunes `node_modules`
  from its scan and `.pzl` isn't in resolve extensions — an npm-distributed Puzzle
  component must hand-write `render()` against `ViewNode`/`SLOT_TAG` (the pain is
  documented in `../tarot-puzzle/docs/PUZZLE-FRICTION.md`, item 7). Copied `.pzl` files
  land in the user's `app/components/`, compile with their own `puzzle build`, and their
  Tailwind `@source "../**/*.pzl"` scan picks up the utility classes automatically.
- **Tailwind v4 is Puzzle's blessed styling pipeline** (`puzzle.config.js` →
  `styles: { use: ['tailwindcss'] }`; `@theme` tokens; Sass deliberately unsupported, D35).
- **Morph is guaranteed available** in Puzzle apps (`@magic-spells/puzzle/morph`,
  `enableMorph(app)`, `data-puzzle-morph`), so overlay pieces can offer first-class
  morph support instead of duck-typing it the way `dialog-panel` does today.
- **A proto version already exists**: `../pyramid-puzzle/app/components/ui/` and
  `fields/` (Spinner, Toast, ColorPicker, PersonPicker, FieldCell…) — Tailwind-styled
  `.pzl` ports of the pyramid-web Svelte design system. Mine these for API shape.

## Core decisions (settled 2026-07-16 — don't re-litigate without Cory)

### 1. Native `.pzl` rebuilds, not web-component wrappers

Pieces are real Puzzle components — `<Select …/>`, `<Button …/>` capitalized tags —
compiling to plain semantic HTML (`<button>`, `<ul>`, `<li>`) with ARIA and Tailwind
classes. **No custom elements. No `<select-dropdown-trigger>` tags. No
`customElements.define`.**

- Why not wrap the existing web components (the Radix analogy): Puzzle's vdom owns the
  DOM. Self-mutating web components (state attributes, injected backdrops) fight
  reconciliation, and the sanctioned escape hatch (`island`) freezes children — which
  would kill reactivity *inside* dialog/panel content. Dealbreaker for overlays.
- Why rebuilds win: the rewrites get dramatically smaller. Puzzle gives free what the
  web components hand-roll — declarative listeners (no `attachListeners`/
  `detachListeners`), `@keydown:escape`/`:enter` modifiers (D38), `animations = { in, out }`
  (Web Animations API), reactive state, `<Slot/>`/named slots, morph. The transferable
  IP is the *behavior design* (keyboard nav, ARIA patterns, state machines), not the code.
- The web-component repos keep living for non-Puzzle contexts (Shopify themes, etc.).
  Accept the two-lineage reality; don't try to share code.

### 2. Registry-shaped repo now; `add` CLI immediately after

The repo is structured as a registry from day one (see below) so the CLI is a
file-copier, not a build tool. **The CLI itself is a follow-up task for another
session** — full spec notes in "The `add` CLI" section at the bottom.

### 3. Docs on magic-spells-site (Astro) + a demo Puzzle app here

- Long-form docs/component pages go in the existing `../magic-spells-site/` Astro app
  (pieces of it are already built).
- A `demo/` Puzzle app lives in this repo (later wave): showcases the components,
  doubles as the dev harness and integration test bed. Scaffold with `puzzle init`;
  pick a free dev port — 3000 is taken by other projects (split-panel=3020,
  morph-engine=3011, colored-text=3040, date-picker=3050, animation-engine=3060).

### 4. Wave 1 = Form core + Button

Button, Label, Field (Input), Password, Search, Checkbox, RadioGroup, Select, Spinner.
Establishes the token system, sizing scale, and form conventions everything else inherits.

## Repo layout

```
puzzle-pieces/
├── PLAN.md                     # this file — the living plan
├── README.md                   # short public-facing intro
├── registry/
│   ├── registry.json           # index aggregating every piece manifest
│   ├── theme/
│   │   └── pieces.css          # @theme token set users merge into app/styles/styles.css
│   ├── lib/                    # shared copied utils (plain .js): position.js, focus.js, …
│   └── ui/
│       └── <piece>/
│           ├── <Piece>.pzl     # one or more component files
│           └── piece.json      # per-piece manifest
└── demo/                       # Puzzle app consuming the pieces (later wave)
```

### Piece manifest (`piece.json`)

Deliberately registry-shaped so the CLI stays a resolver + copier:

```json
{
  "name": "select",
  "description": "Listbox-pattern select with typeahead, keyboard nav, optional morph",
  "files": ["Select.pzl"],
  "registryDependencies": ["lib/position.js"],
  "dependencies": ["@magic-spells/morph-engine"],
  "targetDir": "app/components/ui"
}
```

- `files` — copied to `targetDir` (default `app/components/ui/`).
- `registryDependencies` — other registry files copied alongside (lib utils →
  `app/lib/`, sibling pieces → their own targetDir). Resolved transitively.
- `dependencies` — real npm packages, **plain JS only** (those DO work in
  node_modules; `.pzl` never appears here). The CLI prints an install command,
  never runs it.
- `registry.json` aggregates all manifests for one-fetch resolution.

## Component conventions

Grounded in `../puzzle/examples/blog/app/components/Button.pzl` and
`../pyramid-puzzle/app/components/ui/*.pzl` — follow the grain that already exists.

**Structure**
- One component per `.pzl` file, PascalCase filename, single root element,
  `export default class X extends PuzzleView`.
- **Config-first APIs, not compound components.** Puzzle has no context API for
  cross-component coordination, so `<Select options={…} value={…} @change={…}/>`
  beats a `<SelectTrigger/>`+`<SelectContent/>` family. Presentational structure
  (dialog header/footer) = named slots (v1.21, D53) or documented Tailwind markup
  patterns — never coordinating subcomponents.

**Props & events**
- Props in, callback props out: parent writes `@change={ handler }`, child calls
  `this.props.change(value)`.
- Standard prop vocabulary: `variant`, `size`, `disabled`, `value`, `label`,
  `placeholder`, `class` (merged onto the root element).
- Standard callbacks: `@change`, `@press`, `@show`, `@hide`, `@ready` (the blessed
  callback-ref idiom for imperative handles — DOC-EVENTS). Whether `@show`/`@hide`
  are cancelable via return value is a wave-2 design point; decide once at Dialog.
- This replaces the web components' inconsistent event naming
  (`quantity-input:change` vs `beforeShow` vs `tabchange`) — standardize here, once.
- Reference-stability discipline (PUZZLE-FRICTION lessons, D62/D64): object/array
  props built with `this.memo()`; static callbacks rely on compiled cached handlers.

**Styling**
- Tailwind utilities against semantic `@theme` tokens only — `bg-surface`, `text-ink`,
  `border-border`, `bg-brand`, `text-danger`, … No hex values inside components.
- `registry/theme/pieces.css` defines the token set. Start from the blog example's
  `--color-brand/-ink/-border` naming plus pyramid's `--accent`/`--status-*`; settle
  final names when writing the file (wave 1, before the first component).
- Variant maps as plain objects in `<scripts>` (like Spinner's `PX = { sm: 16, … }`),
  composed into class strings in `data()` — no cva, no tailwind-merge dependency.
  `{#if}`-in-class stays fine for simple two-way variants (Button.pzl precedent).

**Accessibility**
- Carry the web components' a11y work forward: full keyboard nav, ARIA state
  (`aria-expanded`, `aria-selected`, `aria-activedescendant`), visible focus.
- `prefers-reduced-motion` is respected (Puzzle already zeroes animation durations).

**Morph**
- Overlay pieces expose an opt-in `morph` prop.
- Morphable roots must not use transform positioning, stylesheet `opacity`, a CHANGING
  dynamic `style={}` binding, or `animations.in/out` (`puzzle/client-runtime/morph.js`
  header, lines 26–34).
- Component-level trigger↔panel morphs import `@magic-spells/morph-engine` directly
  (plain JS, npm-safe) and declare it in `dependencies`. Note: today `select-dropdown`
  imports morph-engine via a relative monorepo path — the rebuild must use the npm
  package instead.

## Roadmap

| Wave | Piece | Source of truth to port |
|---|---|---|
| 1 | Button | blog `Button.pzl` (extend: sizes; variants incl. destructive/ghost/outline; loading state w/ Spinner) |
| 1 | Label, Field (Input), Password, Search | `../pyramid-puzzle/app/components/fields/` + pyramid-web Svelte design system |
| 1 | Checkbox, RadioGroup | new (simple; ARIA from WAI-ARIA patterns) |
| 1 | Select | `../select-dropdown` v0.2 (gen 2 — listbox, typeahead, dividers, group labels, morph). **NOT** `dropdown-select` (gen 1, legacy, superseded) |
| 1 | Spinner | `../pyramid-puzzle/app/components/ui/Spinner.pzl` (near-verbatim) |
| 2 | Dialog | `../dialog-panel` (native `<dialog>`/`showModal()` for focus trapping; state lifecycle; morph opt-in) |
| 2 | DropdownMenu | `../dropdown-panel` (the existing "dropdown menu": hover-aware open, submenus, deliberately no `role="menu"` per Adrian Roselli) |
| 2 | Popover, Tooltip | new + `lib/position.js` (port positioning from select-dropdown/dropdown-panel; CSS anchor positioning isn't cross-browser yet — JS util) |
| 3 | Tabs | `../tab-group` |
| 3 | Collapsible / Accordion | `../collapsible-content` (incl. group coordinator) |
| 3 | QuantityInput | `../quantity-input` |
| 3 | Progress, Avatar | new (Avatar: mine pyramid's deterministic `avatarVM` hue trick) |
| 4 | Toast | `../notification-stack` (biggest lift; its architecture is the newest) |
| 4 | Command (palette) | new |
| 4 | Sheet (bottom sheet) | `../bottom-sheet` (drag physics — hardest rewrite, deliberately last) |
| 5 | Card, Empty, Breadcrumb, Pagination, Table, Toggle, ToggleGroup, Accordion, AlertDialog | new (easy parity batch — briefs below) |
| 5 | Slider | `../range-slider` (port; brief below) |
| 5 | Calendar, DatePicker | `../date-picker` (port, split in two; brief below) |
| 6 | PanelStack | `../panel-stack` (port; brief below — piece + lib/panel-stack.js helpers) |

Wave 0 (before the first component): write `registry/theme/pieces.css` (the token set)
and `lib/` utils as needed. Each piece ships with: the `.pzl` file(s), `piece.json`,
an entry in `registry.json`, and a demo-app page once `demo/` exists.

## Wave 5 briefs (planned 2026-07-16, evening)

Same build model as waves 1–4: these briefs are the spec; Opus agents build against
them referencing the existing exemplars (Button, Select, Dialog); each agent
compile-verifies in a scratch app with the local Go binary; Fable reviews everything.
All conventions above apply unchanged (config-first, controlled components, callbacks
value-first, semantic tokens only, no `<styles>`, single root, inline SVG).

### Easy batch (nine pieces, no new dependencies)

- **Card** — presentational container: `title`, `description` props render a header
  (both optional, `empty:hidden` discipline like Dialog's footer); default `<Slot/>`
  body; named `slot:footer`. Rounded-xl border-border bg-surface — match the demo's
  existing section-card styling so apps can replace hand-rolled cards 1:1.
- **Empty** — empty-state block: named `slot:icon` (static SVG from the consumer),
  `title`, `description`, default `<Slot/>` for actions. Centered, muted, generous
  padding.
- **Breadcrumb** — config-first `items` array `[{ label, href }]`; last item is
  plain text with `aria-current="page"`; `<nav aria-label="Breadcrumb">` + `<ol>`;
  chevron separator inline-SVG between items (`aria-hidden`).
- **Pagination** — `page`, `count` (total pages), `@change(page)`; windowing math as
  a pure function in `<scripts>` (boundary=1, siblings=1, ellipsis when gapped);
  prev/next buttons disabled at the ends; current page gets `aria-current="page"`;
  root `<nav aria-label="Pagination">`.
- **Table** — styled semantic table tier, NOT a data-table: config-first
  `columns` `[{ key, label, align? }]` + `rows` (array of objects), optional
  `caption`. No sorting/selection/virtualization — document that as out of scope.
  Row striping off by default; hover highlight; `divide-border` hairlines.
- **Toggle** — a pressed-state `<button aria-pressed>`: `pressed`, `@change(pressed)`,
  `variant` (default|outline), `size` (sm|md|lg), `disabled`, default `<Slot/>`.
  Pressed = bg-faint text-ink (not brand — it's a tool state, not a CTA).
- **ToggleGroup** — config-first `items` `[{ value, label, icon?, disabled? }]`
  (label may be visually hidden when icon-only — still required for a11y);
  `type` single|multiple; `value` (string or array to match type); `@change`
  value-first. `role="group"` + `aria-pressed` per button (NOT radio semantics —
  single-select-with-deselect is still a pressed-buttons pattern); joined pill
  styling via `divide-x` like QuantityInput.
- **Accordion** — the group coordinator over Collapsible's animation approach:
  config-first `items` `[{ id, title, content }]` where `content` is plain text,
  `type` single|multiple, `value`/`@change`. Honest limitation to document: Puzzle
  named slots are static, so rich-content accordions should compose `Collapsible`
  directly with parent-managed exclusivity — show that recipe in the demo.
  Heading buttons follow the WAI-ARIA accordion pattern (`aria-expanded`,
  `aria-controls`, heading wrapper).
- **AlertDialog** — confirm/cancel modal for destructive actions. Own thin piece on
  native `<dialog>` following Dialog's discipline (showModal sync, cancel-event
  preventDefault, focus return) — do NOT compose Dialog.pzl; the deltas (no backdrop
  dismiss, no close X, `role="alertdialog"`, initial focus on Cancel) touch its
  internals anyway. Props: `open`, `title`, `description`, `confirmLabel`,
  `cancelLabel`, `destructive` (confirm button styling), `@confirm`, `@cancel`.
  Escape still cancels (a11y); backdrop click does nothing.

### Slider (port of `../range-slider`, ~960 LOC → expect a much smaller .pzl)

Controlled component: `value` (single) OR `valueMin`+`valueMax` (their presence =
range mode, same convention as the source), `min`/`max`/`step` (`step="any"`
supported), `disabled`, `label`/`labelMin`/`labelMax` (aria-labels), `name`
(→ hidden input(s) for form posts), `@input` (live) + `@change` (commit) —
value-first: `(value)` single, `({ valueMin, valueMax }, thumb)` range.
Tailwind-token styling (track bg-faint, fill bg-brand, thumb bg-surface border
shadow); percent positioning so no resize handling needed.

Behaviors that MUST survive the port (see `src/range-slider.js`):
1. **Gesture arming** (~L730/779): thumb press drags immediately; track press only
   *arms* — >3px horizontal promotes to a drag on the nearest thumb, vertical
   movement disarms so the page scrolls. Stacked equal-value thumbs pick a thumb
   from first-move direction.
2. **Touch capture avoidance** (~L860): `setPointerCapture` for mouse/pen only —
   explicit capture mid-touch fires spurious `pointercancel` on mobile engines.
   Touch rides implicit capture + `touch-action: pan-y` and JS axis-lock.
3. **normalize() idempotency** (~L451): quantize to step → clamp to bounds →
   crossover-clamp against the opposing thumb → round. Loop prevention by value
   equality, not guard flags.
4. **`[dragging]` kills transitions** so a track *tap* animates the thumb across
   (no drag = transition stays on) while drags track the pointer raw. Use a
   `dragging` data flag → conditional `transition-none` class in Puzzle.
Keyboard: arrows ±step, PageUp/Down ±10×step, Home/End (native-range parity).
Drag state is internal (`setData`); committed values flow through the parent.
SKIP from the source: `data-content-*` bound inputs and value bubbles (Puzzle
reactivity makes them a parent concern — demo shows value display next to the
slider), tick marks/labels (defer; add `ticks` in a follow-up if wanted).

### Calendar + DatePicker (port of `../date-picker`, ~2800 LOC, split in two)

Two pieces + one lib file:
- **`lib/date-math.js`** — port `src/utils/date-math.js` (~300 LOC) near-verbatim
  into `registry/lib/` as a copied registryDependency. Its invariant is the whole
  ballgame: **local-time date math only** — always `new Date(y, m-1, d)`, never
  `Date.parse`/`new Date('YYYY-MM-DD')` (UTC parse shifts a day across zones);
  `addMonths` clamps day-of-month overflow (Jan 31 + 1mo = Feb 28/29).
- **Calendar** — the inline grid, no popover: `value` (ISO `YYYY-MM-DD`), `min`/
  `max`, `disabledDates` (array or predicate), `locale`, `firstDayOfWeek`,
  `@change(isoValue, date)`, `@monthChange({ year, month })`. Fixed 6×7 42-cell
  grid from `getCalendarGrid` so panel height never jumps; drill-down view state
  machine days↔months↔years (3×4 grids; label button zooms out, selection zooms
  in = pure navigation in date mode); nav arrows repurpose per view (±1mo/±1yr/
  ±12yr) and disable when the whole unit is out of bounds. All month/day names via
  `Intl.DateTimeFormat` — zero hardcoded strings. ARIA: `role="grid"`, column
  headers, roving tabindex, `aria-selected`, `aria-current="date"`; arrows/Home/
  End/PageUp/PageDown move focus, crossing month edges re-renders and keeps focus
  (vdom re-render breaks the source's persistent-button trick — re-focus the
  active cell in `afterUpdate` off a `pendingFocus` data flag, the pattern
  Select uses).
- **DatePicker** — Field-shaped trigger + popover composing Calendar
  (`registryDependencies: ["calendar", "lib/date-math.js"]`): `value`, `open`
  optional-controlled like Select, `label`/`hint`/`error` (Field-consistent),
  `placeholder`, `format` (Intl preset name), `locale`, `min`/`max`, `disabled`,
  `@change`, `@show`/`@hide`, optional `morph` (dependency:
  `@magic-spells/morph-engine`, same opt-in shape as Select). Panel measurement
  rule from the source: hidden panel keeps layout via `visibility:hidden`, never
  `display:none` — auto-flip and morph target-rect measurement read it while
  hidden. Focus lands on selected/today on open, returns to trigger on close.
SCOPE CUT for v1 (deliberate; the source proves them portable later): **range
mode** (the draft-over-committed two-layer selection is the hairiest 500 LOC —
port it as a follow-up once single-date is reviewed) and `type=month` mode.
Events keep the source's semantics: `@change` fires only on commit.

### Sequencing + demo

1. Easy batch: nine Opus agents in parallel (Card/Empty/Breadcrumb/Pagination/
   Table/Toggle/ToggleGroup/Accordion/AlertDialog).
2. Slider: one agent, the gesture notes above verbatim in its brief.
3. Calendar → DatePicker: sequential (DatePicker's brief needs Calendar's final
   props); date-math.js ported first by the Calendar agent.
4. Regenerate `registry.json`; demo gets a fourth nav section **Content** (Card,
   Table, Breadcrumb, Pagination, Empty, Accordion, Toggle/ToggleGroup);
   Slider + DatePicker join **Forms**; AlertDialog joins **Overlays**.
   Browser smoke test in a foregrounded tab (rAF finding, see status log).

Wave 6 candidates (not scoped): Combobox (Select listbox + Command filtering),
HoverCard (Popover positioning + Tooltip delay), ContextMenu, InputOTP, range
mode + `type=month` for DatePicker, Slider ticks.

## Docs-site demo refactor (planned 2026-07-16, night — with Cory)

Turn the 5-page section-dump demo into a proper documentation site:
one page per component, big featured preview boxes with code below, left
component nav, right "On This Page" column, near-black dark theme, premium
typography. The pieces themselves don't change (except theme token values) —
this is a presentation layer over the finished registry.

### 1. Theme: darker dark (registry change, affects all consumers)

Re-tune the dark block of `registry/theme/pieces.css` from slate to near-black,
then re-merge into `demo/app/styles/styles.css`:
`--color-page #09090b` (zinc-950), `--color-surface #101013`,
`--color-surface-sunken #18181c`, `--color-border #1f1f23`,
`--color-border-strong #2e2e35`, `--color-faint` nudged to keep disabled text
legible on the blacker ground. Text/brand/danger stay. Light theme untouched.
Fable hand-tunes and eyeballs every page in the browser (contrast on borders
and muted text is the risk — check Table hairlines, Skeleton pulse, Switch
track, code blocks).

### 2. Information architecture

- Header (sticky, backdrop-blur, border-b): brand **"Puzzle Pieces"** left
  (links to `/`), right side a quiet `40 pieces · v0.1` tag. No page links —
  the sidebar owns navigation now.
- Left sidebar (~240px, sticky, own scroll, `lg:` up): "Getting started" →
  Introduction (`/`), Components index (`/components`); then "Components" →
  all 40 alphabetically. Active item = soft pill (bg-surface-sunken text-ink),
  rest text-muted hover:text-ink. Driven by ONE config module
  `demo/app/docs/nav.js` (name, title, path per piece) so the sidebar, the
  components index, and prev/next (future) all read the same list.
- Main column: `max-w-3xl`, one component per page at `/components/<name>`
  (registry kebab names).
- Right column (~200px, `xl:` up, sticky): "On This Page" anchor list, one
  entry per example section. Static links v1 (smooth scroll); scroll-spy via
  IntersectionObserver is a stretch goal, not required to ship.
- `/` = Introduction: what puzzle-pieces is, the copy-in model (why not npm),
  quick start (copy file → build), theme merge snippet, link to components.
- `/components` = index: a Card grid of all 40 (name + piece.json description)
  linking to their pages. The old Overview/Forms/Overlays/Feedback/Content
  routes die; their example wiring gets mined into the per-component pages.
- Mobile (v1): sidebar hidden below `lg`; a "Components ▾" disclosure under
  the header opens the same nav list (a Sheet-based drawer is the stretch
  dogfood, not required).

### 3. Docs primitives (demo-only components, `demo/app/components/docs/`)

NOT registry pieces — they live with the demo. Fable writes all three (shell
quality is the whole point of this refactor):

- **ExampleBox.pzl** — the featured demo frame, the heart of the docs look:
  props `title` (optional section heading rendered above), `id` (anchor),
  `code` (string). Renders a tall bordered rounded-xl box on `bg-page` (NOT
  surface — the component's own surfaces must pop) with the demo slot centered
  (`min-h-[350px]`, generous padding), then a code area below: collapsed shows
  ~3 dimmed lines with a fade-out gradient and a centered "View Code" pill;
  expanded shows the full `<pre class="whitespace-pre">{ code }</pre>` with a
  Copy button (navigator.clipboard). Code is plain text-muted monochrome v1 —
  no highlighter dependency; a tiny tokenizer is a later nicety.
- **Toc.pzl** — config-first `items` `[{ label, href }]` → "On This Page"
  column. Clicks scroll via scrollIntoView({behavior:'smooth'}) in a handler
  (do NOT rely on raw `#hash` anchors — the SPA router may intercept; this is
  the one router risk, verify early on the Button exemplar page).
- **SideNav.pzl** — renders the nav.js config with section labels + active
  path highlighting (layout passes the current path).

### 4. Page template (every component page follows the Button exemplar)

H1 (tracking-tight) + one-line description (from piece.json, hand-tightened) →
**Installation** section (the file(s) to copy from `registry/ui/<name>/`, npm
deps called out when piece.json has them, pieces.css reminder) → **main
ExampleBox** (the hero demo, no title) → one ExampleBox per additional example
(Variants, Sizes, Disabled, With hint, Keyboard, Morph, …) with `title`+`id` →
Toc gets Installation + those ids. Code strings live as template-literal
consts in the view's `<scripts>` (JS side — braces/backticks are safe there)
and mirror the example markup exactly. State wiring is lifted from the
existing five demo views — every component already has working examples to
mine; pages mostly re-house them.

### 5. Execution (same delegation model)

1. **Fable**: theme re-tune + docs shell (Default.pzl rewrite, nav.js,
   SideNav, ExampleBox, Toc) + the **Button page as exemplar** + Introduction
   + `/components` index + routes.js restructure. Verify the anchor-scroll
   router question here.
2. **Opus agents, 7 parallel**, each owning a family of pages built against
   the Button exemplar (lift examples from the old views before they're
   deleted; compile-verify per agent):
   form-text (Label, Field, PasswordField, SearchField, Textarea) ·
   choice-controls (Checkbox, RadioGroup, Switch, Select, QuantityInput,
   Slider) · dates (Calendar, DatePicker) · overlays (Dialog, AlertDialog,
   DropdownMenu, Popover, Tooltip, Sheet, Command) · feedback (Toast,
   Progress, Spinner, Skeleton, Avatar, Badge, Alert) · content (Card, Table,
   Breadcrumb, Pagination, Separator, Kbd, Empty) · disclosure (Tabs,
   Collapsible, Accordion, Toggle, ToggleGroup).
3. **Fable**: review pass, delete the five old views, footer/count truthing,
   production build + doctor, browser smoke (foregrounded window for the
   interactive checks), commit.

Risks to watch: hash-anchor vs SPA router (mitigated in Toc); 40 routes in
routes.js is fine but keep view imports lazy if the bundle balloons (measure —
current app.js is 59KB gzip, 40 thin pages should stay well under ~120KB);
sidebar scroll position persistence across navigations (nice-to-have).

## Wave 6: PanelStack brief (planned 2026-07-16, late night — with Cory)

Port of `../panel-stack` (the `<panel-stack>`/`<stack-panel>` web component —
nested-panel navigation stack with fluid push/pop slide transitions; mobile
menus, settings drill-downs, wizards). Name stays **panel-stack / PanelStack**:
no external library has an equivalent to align to, and QuantityInput set the
keep-the-name precedent. Source of truth: `../panel-stack/src/panel-stack.js` (~300 LOC) and
`src/panel-stack.css` (state-attribute driven). NOTE: `src/panel-stack.css:84`
has a `background: blue` debug leftover — do not port it (and it should be
deleted upstream).

### Architecture: the Tabs split, adapted for animation

The web component hosts N rich panels and flips a `state` attribute per panel.
Puzzle can't do that in one piece (named slots are static — no N dynamic
panels) and panels can't be `{#if}`'d (unmount = no slide-out; every panel must
stay mounted). So, like Tabs renders only the tablist, PanelStack renders only
the **frame + behavior**; panels are parent-authored divs inside the default
slot, classed per-state by a lib helper. A state flip swaps classes on the same
mounted node, so CSS transitions fire naturally.

**`lib/panel-stack.js`** (registryDependency, pure functions):
- `panelState(stack, handle)` → `'current' | 'previous' | 'next'` —
  top of stack = current; elsewhere in stack = previous; not in stack = next.
- `panelClass(stack, handle, effect?)` → the panel's full Tailwind class string.
- `panelInert(stack, handle)` → `panelState(...) !== 'current'`.

**`registry/ui/panel-stack/PanelStack.pzl`** — controlled component:
- Props: `stack` (array of handles, root-first — parent-owned, like every
  controlled piece), `effect` (`'slide'` default | `'stack'`), `class`.
- Callback: `@change(nextStack, { type: 'push'|'pop'|'reset', fromHandle,
  toHandle })` — value-first. Cancel = parent declines to update `stack`
  (replaces the source's cancelable event).
- Template: single root div `relative block w-full h-full overflow-hidden
  rounded-[inherit] touch-manipulation` (+ props.class), default `<Slot/>`.
  The parent gives the WRAPPER of the stack a size and radius (panels are
  absolute inset-0 and inherit the radius) — same rule as the source.
- Delegated click handler on the root: `closest('[data-action-stack-push]')`
  with a `target` attr → push (preventDefault, remember trigger);
  `closest('[data-action-stack-pop]')` → pop. Reject a push whose handle is
  already in the stack (deviation from source, which only rejected pushing the
  current handle — a dup would make one handle two states at once).
- Escape keydown on the root: bail if `event.defaultPrevented`; bail at root
  (depth ≤ 1) so a wrapping Dialog/Sheet closes naturally; otherwise
  preventDefault + stopPropagation + emit pop. Use a plain `@keydown` handler
  unless the `:escape` modifier is verified not to auto-preventDefault.
- Focus (the ported IP, all imperative — safe): trigger-memory map
  `handle → trigger el` populated by delegated pushes; in `afterUpdate`,
  diff the previous stack (Collapsible's `_openState` guard pattern):
  push/reset → focus `[data-stack-focus]` in the new current panel, else its
  first focusable; pop → focus the remembered trigger if still connected,
  else fall back to the current panel. ALWAYS `{ preventScroll: true }` (the
  source comment explains: without it the browser scrolls the clipped frame's
  ancestor sideways). Find the current panel via `[data-stack-panel="…"]`.
  Port gotcha: the source ordered focus BEFORE the outgoing panel went inert
  (stranded-focus warning); in Puzzle the vdom applies inert and the refocus
  lands in afterUpdate — a one-frame blur to body is accepted.

### Panel recipe (parent-authored, documented like Tabs' tabpanel recipe)

```
<PanelStack stack={ stack } @change={ setStack }>
  <div data-stack-panel="root" role="group"
       class={ panelClass(stack, 'root') }
       inert={ panelInert(stack, 'root') }>
    <button data-action-stack-push target="shop">Shop</button>
  </div>
  <div data-stack-panel="shop" role="group"
       class={ panelClass(stack, 'shop') }
       inert={ panelInert(stack, 'shop') }>
    <button data-action-stack-pop>Back</button>
  </div>
</PanelStack>
// parent: created() { this.setData({ stack: ['root'] }); }
// events = { setStack: (next) => this.setData('stack', next) };
```

### Motion values (CSS custom props → class maps in the lib)

Customization becomes edit-your-copied-file (the pieces model), so the
`--ps-*` custom-property surface goes away. Class fragments, per state:
- base (every panel): `absolute inset-0 overflow-hidden rounded-[inherit]
  transition-[translate,scale,filter,opacity] duration-[420ms]
  ease-[cubic-bezier(0.16,0.87,0.64,1)] motion-reduce:transition-none`
  — RULE (found in browser): Tailwind v4's translate-x-*/scale-* utilities set
  the CSS `translate`/`scale` PROPERTIES, not `transform`; a
  `transition-[transform,…]` list never animates them (panels snap). Name
  `translate` and `scale` explicitly in any transition that moves a panel.
- current: `translate-x-0 z-[1]`
- previous (slide): `translate-x-[calc(-100%-50px)] scale-x-110 blur-[2px]
  opacity-10 z-0 pointer-events-none motion-reduce:opacity-0`
- next (slide): `translate-x-[calc(100%+50px)] scale-x-110 blur-[2px]
  opacity-10 z-[2] pointer-events-none motion-reduce:opacity-0`
- previous (effect=stack): stays in place behind the current panel —
  `translate-x-0 scale-95 blur-[1px] brightness-50 -z-[1] pointer-events-none`
  plus the inner-shadow overlay the source painted on `::after` (above the
  panel's own content): `after:absolute after:inset-0 after:rounded-[inherit]
  after:pointer-events-none after:content-[''] after:transition-shadow
  after:duration-[420ms]` with
  `after:shadow-[inset_0_0_40px_rgba(0,0,0,0.5)]` only in the previous state.
  `next` under effect=stack keeps full opacity (source sets opacity-next: 1).
- Panels need their own opaque `bg-surface` (they stack). Demo panels show it.
- Deliberately dropped from the source: `--ps-perspective` (no 3D transform
  ever uses it), `will-change`, the `:not(:defined)` FOUC guard (no custom
  elements), z-index knobs.

### Morph + a11y notes for the file header

Panels position with transforms → neither the stack root nor panels may be
morph roots (same note Select carries). Escape composition with Dialog is a
feature: drill in, Esc pops; at root, Esc bubbles and the dialog closes.
Focus-flash rule applies: any focusable styling in demo panels sets outline
color unconditionally.

### Deliverables

1. `registry/lib/panel-stack.js` + `registry/ui/panel-stack/PanelStack.pzl` +
   `piece.json` (`registryDependencies: ["lib/panel-stack.js"]`, no npm deps).
2. `registry/registry.json` regenerated (alphabetical: pagination → panel-stack
   → password-field; mirror how date-math.js is represented for the lib file).
3. Demo copy-in: `demo/app/components/ui/PanelStack.pzl`,
   `demo/app/lib/panel-stack.js`.
4. Docs page `demo/app/views/components/PanelStackDoc.pzl` off the TabsDoc/
   ButtonDoc exemplar: hero = drill-down menu (root → shop → categories) in a
   sized rounded wrapper; `effect="stack"` example; inside-a-Dialog example
   (the Esc layering story); the panel recipe spelled out in the code strings.
   Toc entries per section. Entries in `demo/app/docs/nav.js` (Content section,
   title "Panel Stack") and `demo/app/routes.js` (alphabetical).
5. Compile-verify: `cd demo && npm run build` (local Go binary
   `../../puzzle/puzzle`). Fable browser-smokes on the dev server (port 3070).

## The `add` CLI

**SHIPPED 2026-07-17 — as `puzzle add piece <name…>` in the Go CLI** (Cory chose the
framework-CLI integration over the standalone-npx-first note below; spec otherwise
executed as written). Lives in `../puzzle/compiler/internal/pieces/` + `add.go`.
Registry source: `--registry <path|url>` flag → `PUZZLE_PIECES_REGISTRY` env var →
`https://raw.githubusercontent.com/magic-spells/puzzle-pieces/main/registry` (works
pre-publish by pointing the env var at this checkout's `registry/`). Steps 1–4 below
shipped as specced (transitive resolution incl. `lib/*` deps, refuse-unless
`--overwrite` with all-or-nothing pre-flight, printed-never-run npm installs).
Theme handling (amended 2026-07-17, same day): the theme is COPIED like a piece —
`add piece` writes `theme/pieces.css` verbatim to `app/styles/pieces.css` (+ lock
entry) when the app has neither the tokens nor the file, and prints the one-line
`@import './pieces.css';` wiring step (styles.css is user-owned, D3). Detection
keys on the `puzzle-pieces design tokens` header comment or a `pieces.css`
reference in styles.css — don't reword that comment without updating the CLI's
marker. Step 5 resolved: NO header
stamp (copies stay byte-identical to the registry, diffable); instead `pieces.lock`
at the app root records sha256 content hashes per piece/lib — hashes, not version
numbers, so nothing needs bumping and a future `diff`/`update` can distinguish
upstream-changed vs locally-customized. Original spec kept below for reference.

`npx @magic-spells/puzzle-pieces add <piece…>` (original note — superseded above):

1. Fetch/read `registry.json`; resolve requested pieces + `registryDependencies`
   transitively (dedupe).
2. Copy files to each manifest's `targetDir` (default `app/components/ui/`; lib files
   to `app/lib/`). **Refuse if a target file exists unless `--overwrite`.**
3. Print — never auto-run — the npm install command for accumulated `dependencies`.
4. If the app's `styles.css` lacks the pieces tokens, print the `pieces.css` merge
   snippet. **Never rewrite user files** (Puzzle D3 precedent: print snippets).
5. Consider stamping a header comment (piece name + registry version) in copied files,
   or a `pieces.lock`, to enable a future `diff`/`update` command.

Precedent to model on: `puzzle add` / `puzzle generate`
(`../puzzle/compiler/cmd/puzzle/generate.go`, `initcmd.go`,
`compiler/internal/scaffold/templates/`). A `puzzle add piece:button` framework
integration is possible later; keep the CLI standalone first. Local dev can point the
registry fetch at this repo's path before anything is published.

## Status log

- **2026-07-17** — `add` CLI SHIPPED as `puzzle add piece <name…>` in the Go CLI
  (see the amended "The `add` CLI" section above for the full contract: registry
  source chain, verbatim copies, `pieces.lock` sha256 hashes instead of piece
  versions, D3 print-don't-run). 22 new Go tests; verified end-to-end against
  this registry (button, then date-picker → calendar + lib/date-math.js
  transitively; conflict refusal; app builds with pieces + morph-engine).
  Registry untouched. Remaining from the original follow-ups: docs pages for
  the CLI flow, `diff`/`update` command, and publish (which activates the
  default GitHub raw URL).

- **2026-07-16 (late night)** — PANELSTACK SHIPPED (wave 6 brief above executed;
  registry now 41 pieces). Implementation delegated to Codex against the brief,
  Fable-reviewed, Cory browser-verified. `registry/ui/panel-stack/` +
  `registry/lib/panel-stack.js` (second lib file), demo copy-in, docs page wired
  into nav/routes, piece counts truthed 40→41 across the demo shell.
  - RULE (found in browser, now also in the brief): Tailwind v4's
    `translate-x-*`/`scale-*` utilities set the CSS `translate`/`scale`
    PROPERTIES, not `transform` — a `transition-[transform,…]` never animates
    them (panels snap, only blur/fade moves). Name `translate` and `scale`
    explicitly in any transition that moves an element.
  - Upstream: `../panel-stack/src/panel-stack.css:84` carries a
    `background: blue` debug leftover (not yet in its dist) — delete before the
    web component's next build.
  - Session gotcha: a parallel session's docs-layout sweep (max-w-3xl →
    mx-auto max-w-5xl) missed the then-untracked PanelStackDoc; synced by hand.
    Untracked files don't ride along in another session's refactor commits —
    re-check new pages against the current exemplar before shipping.

- **2026-07-16 (night)** — DOCS-SITE REFACTOR SHIPPED (plan above executed): the demo
  is now a full docs site. Near-black dark tokens landed in
  registry/theme/pieces.css (page #09090b — a REGISTRY change, all consumers get it);
  docs shell (sticky blurred header + "Puzzle Pieces" brand + GitHub link, grouped
  sidebar from generated app/docs/nav.js — Forms/Overlays/Feedback/Content sections
  with text-ink labels, mobile nav in a Collapsible), docs primitives in
  demo/app/components/docs/ (ExampleBox with View-Code peek + copy, CodeBlock, Toc
  with JS smooth-scroll — raw #hash anchors avoided on purpose), 40 per-component
  pages at /components/<name> built by 7 Opus agents against the ButtonDoc exemplar,
  Introduction + /components card index, old five views deleted. Gotchas recorded:
  literal angle-bracket tag names AND literal {#if}/{#for} tokens break template
  PROSE (fine inside JS template-literal code strings); ExampleBox's frame must NOT
  be overflow-hidden (clips opened popovers — the code area clips itself instead).
  Placeholder-file pattern kept routes.js single-writer while agents fanned out.

- **2026-07-16** — Repo created; decisions settled with Cory (native rebuild, registry
  shape, Astro docs on magic-spells-site + demo app here, wave 1 = form core + Button).
- **2026-07-16 (same day, full-auto session)** — ALL FOUR WAVES BUILT: 28 pieces live in
  `registry/ui/` (waves 1–4 above, plus a bonus batch: Textarea, Switch, Badge, Alert,
  Skeleton, Separator, Kbd). `registry/registry.json` generated from the piece manifests
  (regenerate by re-running the aggregation after adding pieces). `registry/theme/pieces.css`
  carries the token set (light + dark). Every piece was compile-verified against the real
  compiler and code-reviewed; conventions audit passes (no hex, no `<styles>`, no custom
  elements, one exported PuzzleView per file). `demo/` is a live showcase app (port 3070)
  that consumes the pieces via the copy-in flow.
  - Patterns that emerged (follow them in new pieces): `this.element` is the root-DOM
    getter; native `<dialog>` overlays always preventDefault the `cancel` event and let
    the parent flip `open`; a `{#for}` body needs a single element root (precompute
    row role/class in `data()`, or use a `display:contents` wrapper); `inert={ !open }`
    compiles, `aria-hidden` needs the string form; controlled-component discipline
    everywhere (parent owns value/open; callbacks value-first).
  - Deferred per-piece follow-ups are documented in each file's header comment (Dialog
    exit animation, Toast deck layout + progress bars, DropdownMenu hover/mega/submenus,
    Tooltip horizontal flipping, indeterminate Progress shimmer).
  - Still open: the `add` CLI (spec above — next session), Astro docs on
    magic-spells-site, npm publish of the CLI package, Slider (port range-slider),
    Card/Pagination/Breadcrumbs (wave 5 candidates).
  - Browser smoke test (Chrome, demo app) verified: tokens + dark mode, Button
    variants + @press→parent flow, Field error/hint states, Checkbox/Switch/
    RadioGroup rendering, Select full keyboard round-trip (open → ArrowDown →
    Enter → value through parent → focus return), Toast live region, routing,
    zero console errors. Caught + fixed: a bare `flex` on Sheet's <dialog>
    defeated `dialog:not([open]){display:none}` (closed sheet rendered in-flow)
    — now `open:flex`; RULE: never put a bare display utility on a <dialog>.
- **2026-07-16 (evening, cont.)** — WAVE 5 BUILT: all 12 pieces live, registry now
  40 pieces. Same delegation model (10 parallel Opus agents + a demo-wiring agent;
  DatePicker sequenced after Calendar's API landed; every piece compile-verified in
  a scratch app and Fable-reviewed). `registry/lib/` now exists (date-math.js is the
  first shared lib file; calendar + date-picker declare it via registryDependencies).
  Demo gained the /content view + nav entry; Slider/Calendar/DatePicker on /forms,
  AlertDialog on /overlays; production build + doctor clean (59.3 KB app.js gzip).
  Also: Tooltip grew a placement-aware arrow (diamond straddling the panel edge,
  follows runtime top/bottom flips); ToggleGroup documents the icon limitation
  (no per-item SVG props in Puzzle — use short/emoji labels until a slots-in-loop
  primitive exists); AlertDialog verified bare `autofocus` compiles and showModal()
  honors it. Wave 6 candidates unchanged (Combobox, HoverCard, ContextMenu,
  InputOTP, DatePicker range/month modes, Slider ticks).
  - RULE (found by Cory in browser): Tailwind v4's `transition-colors` includes
    `outline-color`, so a color set only inside `focus-visible:` animates from the
    default (white in dark mode) on every focus — a visible flash. Every focusable
    piece now sets the outline COLOR unconditionally (`outline-ring` /
    `outline-danger` alongside the `focus-visible:outline-2` reveal); do the same
    in all new pieces. Also this pass: Tooltip gained a placement-following arrow;
    Collapsible demo moved Feedback → Content (beside Accordion, with a
    composed-exclusivity example teaching the Collapsible-vs-Accordion split:
    rich slot content vs config-first group coordination).
- **2026-07-16 (evening)** — Wave 5 planned with Cory (briefs above): nine easy
  parity pieces + Slider (port `../range-slider`) + Calendar/DatePicker
  (port `../date-picker`, single-date v1, range mode deferred). Both source repos
  recon'd; gotchas captured in the briefs. Also this session: Overview page demo
  groups got component-name captions; Switch reworked into a settings row (label
  left, control right — layout note in its header comment); demo's disabled
  checkbox/switch examples given hints so the disabled state reads intentionally.
  - **Upstream puzzle finding to file:** the view update scheduler
    (client-runtime/views/PuzzleView.js ~line 675) uses requestAnimationFrame
    with a fallback only when rAF is absent — in a HIDDEN tab rAF is suspended,
    so setData()-driven re-renders stall until the tab is visible. The store
    flush got exactly this fix in D63 (hidden-tab setTimeout path); the view
    scheduler needs the same treatment. Surfaced by driving the demo in a
    backgrounded automation tab.
