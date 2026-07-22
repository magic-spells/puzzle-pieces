# puzzle-pieces

Beautifully-behaved UI components for the [Puzzle framework](https://github.com/magic-spells/puzzle).
Tailwind-styled, accessible, morph-aware — and **copied into your app, not installed
from npm**.

## Why copy-in?

Puzzle compiles `.pzl` single-file components, and `.pzl` files can't live in
`node_modules` — the compiler doesn't scan there. So instead of fighting that,
puzzle-pieces embraces it: each piece is source you copy into `app/components/ui/`.
Your `puzzle build` compiles it, your Tailwind scan picks up its classes, and the
code is yours to restyle and rework.

## Status

**40 pieces built** — the full wave 1–5 roadmap plus extras: Button, Label, Field,
PasswordField, SearchField, Textarea, Checkbox, RadioGroup, Switch, Select,
QuantityInput, Spinner, Dialog, DropdownMenu, Popover, Tooltip, Sheet, Command,
Tabs, Collapsible, Toast, Progress, Avatar, Badge, Alert, Skeleton, Separator, Kbd,
Card, Empty, Breadcrumb, Pagination, Table, Toggle, ToggleGroup, Accordion,
AlertDialog, Slider, Calendar, DatePicker (Slider and Calendar/DatePicker are
native ports of the range-slider and date-picker web components).

Browse them in `registry/ui/` (one folder per piece, `piece.json` manifest each) or
run the demo app: `cd demo && npm run dev` (port 3070). Architecture, conventions,
and the CLI spec live in [PLAN.md](./PLAN.md). The `add` CLI is the next milestone.

## How it will work

```sh
npx @magic-spells/puzzle-pieces add button select
```

…copies `Button.pzl` and `Select.pzl` (plus any shared lib files) into your app and
prints anything else you need. Until the CLI ships, copy files from `registry/ui/`
by hand.
