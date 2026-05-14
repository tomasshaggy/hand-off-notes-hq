# Transfer Type Popover — Handoff

A two-pane categorised picker for selecting one transfer type. Trigger button opens a popover with a category rail on the left and a scoped, searchable item list on the right. Single-select.

This doc covers behavior, interaction, and the specific rules that are easy to miss from the visuals alone.

---

## 1. Layout & size

- Popover size: **700 × 400 px**, fixed. Tailwind classes `w-175 h-100`. No `min-*`/`max-*` constraints.
- Both dimensions are spacing-scale values (variables): `w-175` = 175 × 0.25rem = 700 px; `h-100` = 100 × 0.25rem = 400 px. They can be tuned later (e.g. if the reason list grows), but they should always stay fixed values — never `min-h-…` or content-driven height.
- Left rail: 192 px (`w-48`), full height, 1 px right border.
- Right pane: fills the remaining ~508 px, full height.
- A dimmed backdrop (scrim) sits behind the popover whenever it's open. Click on the backdrop closes the popover. `Esc` also closes.
- Desktop-only surface — no responsive variant; no touch sizing.

---

## 2. Trigger

Three states:

| State | Appearance |
|---|---|
| Empty | Muted placeholder `Select transfer type…` |
| Selected | Type label + colored reason badge. Truncates, never wraps. |
| Error | Destructive-tinted border. The trigger does **not** render the error message — that goes below. |

Clicking the trigger opens the popover regardless of state.

---

## 3. Left rail (categories)

Order, top to bottom:

```
All           [count]
Favorites     [count]
─────────────────────   ← 1 px divider
Revenue
Expense
Maple Trading
BAU
Liquidation Prep
```

- `All` and `Favorites` are **meta rows** — they always show a live count on the right.
- Category rows (Revenue, Expense, …) **do not show counts**.
- Counts are live: if an upstream filter reduces the eligible set, `All` and `Favorites` shrink honestly.
- Active row: filled background tint, `aria-current="true"`. Inactive: transparent with subtle hover background. Activation is instant — no animation.
- Click to switch. No hover-to-switch.

---

## 4. Default rail on open

| Situation | Default rail |
|---|---|
| Fresh open, user has favorites | `Favorites` |
| Fresh open, user has no favorites | `All` |
| Re-opening with an existing selection | The category that selection belongs to (e.g. Expense) |

Recomputed every time the popover opens.

---

## 5. Right pane — search

- One search input at the top of the right pane.
- Search is **scoped to the current rail row**. Typing while `Expense` is selected filters within Expense only. To search across categories, switch to `All`.
- Placeholder reflects scope: `Search transfer types…` / `Search favorites…` / `Search <reason> types…`.
- Every keystroke instantly snaps the list back to top (no smooth scroll). Prevents the "I scrolled down then typed and nothing happened" bug.

---

## 6. Right pane — item list

Each item row:

- Type label on the left.
- Colored reason badge next to it — **same color as the rail/trigger badge, just smaller**. Don't introduce a separate palette for the small variant.
- Star button on the right (see §8).

Selecting an item is **one click — the popover closes immediately**. No confirm step, no animation.

### Sort order

- Inside `All`: grouped by reason order, then alphabetical within each reason. Favorites are **not** pinned to the top in `All` (`All` is the unbiased global view).
- Inside a category: alphabetical, with starred items pinned above an in-pane separator (see §7).
- Inside `Favorites`: grouped by reason, alphabetical within each group (see §7).

---

## 7. Grouping rules

### Inside a category (e.g. Expense)

If the category contains ≥1 starred item:

```
Favorites          ← heading
  Gas Purchase     ← starred
  Incentives       ← starred
─────────────────  ← separator
  Custody Fee
  Injection Bot
  Mint Fee
  …
```

If the category has zero starred items: flat unheaded list.

### Inside the `Favorites` rail row

Items are grouped by **reason**, each group has its label and is separated by a full-width separator:

```
Revenue
  Liquidation Fee
  Other
─────────────────
Expense
  Gas Purchase
  Incentives
```

This is so the user can still tell which category each favorite belongs to when looking at the flat favorites view.

### When the user is searching

Search collapses the in-category Favorites/non-Favorites grouping into a single flat result list. The Favorites heading and separator disappear during an active search — they come back when the search clears.

---

## 8. Star / favorites

- Star icon sits on the right of each item row.
- **Hidden by default.** Visible only on row hover and on keyboard focus.
- Filled amber when starred, outlined when not.
- `aria-label` flips between `Add <type> to favorites` / `Remove <type> from favorites`.
- Clicking the star toggles favorite **without** selecting the row — the click must not propagate to the row.

---

## 9. Empty states

| Situation | Copy |
|---|---|
| Search returned nothing | `No transfer types found.` |
| `Favorites` rail row, user has zero favorites | `No favorites yet. Star a type to add it.` |
| `Favorites` rail row, favorites exist but upstream filter eliminates them all | `No favorites under this filter.` |

The third copy is what tells the user "the filter caused this, not your behavior" — don't fall back to the generic empty copy.

Empty *category* rows still appear in the rail (don't dynamically hide a category row when it's empty) — rail stability is more important than saving one click.

---

## 10. Keyboard & focus

- Opening the popover moves focus to the search input.
- Closing returns focus to the trigger.
- `Esc` closes.
- Arrow keys navigate the item list. `Enter` selects.
- Visible focus ring on every interactive element (rail rows, items, star, search) — same ring style across the popover.

---

## 11. Motion

- Popover open/close: ≤150 ms fade only.
- Rail row activation: no animation.
- Right-pane content swap on rail click: no animation — it should feel like a switch.
- Scroll-reset on keystroke: instant.
- Respect `prefers-reduced-motion`.

---

## 12. Tailwind reference

The classes below are the actual values from the implementation. They reference shadcn theme tokens (`bg-popover`, `text-muted-foreground`, etc.) — don't replace with raw colors.

### Popover container

```
fixed top-1/2 left-1/2 z-50 -translate-x-1/2 -translate-y-1/2
w-175 h-100 rounded-md border bg-popover shadow-lg overflow-hidden
```

### Backdrop

```
fixed inset-0 z-50 bg-black/40
```

### Body (flex split)

```
flex h-100
```

### Left rail wrapper

```
flex flex-col gap-0.5 w-48 shrink-0 border-r p-1.5
```

### Rail row

```
flex items-center justify-between gap-2 rounded-sm px-2 py-1.5 text-left text-sm
focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring
```

- Active: `bg-accent text-accent-foreground`
- Inactive: `text-foreground hover:bg-accent/50`
- Count badge: `text-xs text-muted-foreground tabular-nums`

### Rail divider (between meta and category rows)

```
my-1 h-px bg-border
```

### Right pane wrapper

```
flex-1 min-w-0 flex flex-col
```

The inner `<CommandList>` overrides the default cap with `flex-1 max-h-none` so the list fills the full 400 px.

### Trigger button

```
justify-between font-normal h-9 w-full min-w-0
```

Error variant adds `border-destructive`. Chevron: `size-4 shrink-0 text-muted-foreground`. Placeholder text: `text-muted-foreground`.

### Reason badge (item rows + trigger — same classes, same colors)

```
inline-flex items-center rounded px-1.5 text-[11px] leading-4 font-medium
```

Color pairs (background + text), per reason:

| Reason | Classes |
|---|---|
| Revenue | `bg-fuchsia-50 text-fuchsia-700` |
| Expense | `bg-emerald-50 text-emerald-700` |
| BAU | `bg-cyan-50 text-cyan-700` |
| Liquidation Prep | `bg-pink-50 text-pink-700` |
| Maple Trading | `bg-stone-100 text-stone-800` |

### Star button

```
shrink-0 opacity-0 group-hover/command-item:opacity-100 focus-visible:opacity-100
```

Icon:
- Starred: `size-3.5 fill-amber-400 text-amber-400`
- Unstarred: `size-3.5 text-muted-foreground`

### Error text below trigger

```
text-xs text-destructive
```

### Field wrapper (label + trigger)

```
flex flex-col gap-2 min-w-0
```

---

## 13. Quick checklist for developers

- [ ] Trigger with three states (empty / selected / error).
- [ ] Popover 700 × 400 px (`w-175 h-100`), fixed, no min/max, with backdrop.
- [ ] Rail: `All` / `Favorites` (with live counts) / divider / category rows (no counts).
- [ ] Default rail follows §4 (fresh vs editing vs has-favorites).
- [ ] Scoped search with context-aware placeholder; scroll-to-top on every keystroke.
- [ ] Item rows: label + small reason badge (same color as trigger badge) + hidden-until-hover star.
- [ ] In-category Favorites heading + separator when applicable.
- [ ] Favorites view grouped by reason with labels and separators.
- [ ] Search collapses in-category grouping into a flat list.
- [ ] Selecting an item closes the popover immediately.
- [ ] Star click toggles favorite, does not select the row.
- [ ] Empty states cover three cases (§9).
- [ ] Focus moves to search on open, returns to trigger on close; `Esc` closes; arrow keys + Enter on list.
- [ ] Motion timings per §11; respect `prefers-reduced-motion`.
