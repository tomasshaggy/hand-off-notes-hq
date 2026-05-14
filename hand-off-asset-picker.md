# Asset Picker — Handoff

A single-pane picker for selecting an asset+chain pair from the From wallet's balances. Triggered by a compact chip in the amount row; opens an overlay scoped to the form panel (not the viewport) with a searchable balance list.

This is a **different shape** from the Transfer Type / From Wallet popovers — flat list, no rail, no favorites. Don't try to inherit from those docs.

---

## 1. Trigger (chip, not a form input)

The trigger is a **pill / chip**, not a full-width outlined input. It sits inline in the amount row.

Three states:

| State | Appearance |
|---|---|
| No value | Text `Select` + chevron. |
| Selected | Asset logo (with chain-logo "ring badge" stuck to its bottom-right corner) + asset label + chevron. |
| Disabled | 50% opacity, no pointer events. Disabled when no From wallet is selected. |

The combined asset+chain badge ("logo + small chain logo ringed at its bottom-right") is a distinctive marker for this picker — it tells the user, at a glance, *which* chain's USDC they picked.

Chip Tailwind:

```
flex items-center gap-1.5 h-8 rounded-full border border-input bg-background px-2.5
text-sm font-medium shrink-0 cursor-pointer hover:bg-accent transition-colors
disabled:opacity-50 disabled:pointer-events-none
```

The inner asset logo: `size-5.5 rounded-full`. The chain badge superimposed on it: `size-3 rounded-full ring-2 ring-background`, positioned `absolute -bottom-0.5 -right-0.5`.

---

## 2. Overlay (panel-scoped, not viewport)

This is the biggest difference from the two-pane pickers:

- The overlay is **absolutely positioned inside the form panel** (`absolute inset-0 z-50 flex items-center justify-center`), not `fixed` to the viewport.
- The backdrop is **transparent** — it's a click-catcher, not a dim scrim. Clicking it closes the picker.
- The panel itself: `relative w-[calc(100%-3rem)] rounded-lg border bg-popover shadow-lg ring-1 ring-foreground/10`. Width is "form width minus 24 px gutter on each side", **not** a fixed pixel size.
- Height is **content-driven** within bounds: the list uses `min-h-48 max-h-80` (192–320 px). The container itself has no fixed height — it grows to fit.

Closing behavior:
- Click on the (transparent) backdrop closes.
- Selecting an item closes immediately.
- `Esc` closes (via cmdk default).
- On close, the **search query is reset to empty** — opening it again starts fresh.

---

## 3. List & search

- One search input at the top of the panel (`<CommandInput placeholder="Search assets...">`).
- Search matches against **asset code, asset label, asset full name, and chain name** (e.g. typing `Sol` matches Solana-chain USDC; typing `USD Coin` matches USDC; typing `eth` matches ETH or Ethereum-chain rows).
- Manual search state with `shouldFilter={false}` — cmdk's built-in filter is bypassed so the matching logic above can include chain name explicitly.
- Empty state: `No assets found.` Only one case (no favorites mode, no upstream filter that can produce alternate copy).

---

## 4. Sort order

**Descending by USD value** (balance × price), highest first. Not alphabetical, not by asset code. Operators care about "where's the value," so the largest position is always at the top.

If two rows tie on USD, native sort is fine — no explicit tiebreaker needed.

---

## 5. Item row anatomy

Each row is a **three-zone layout**:

```
[logo]   USDC  |  ⬡ Ethereum                      12,345.67
         USD Coin                                  $12,345.67
```

- **Left:** asset logo, `size-7 rounded-full`.
- **Center (stacked):**
  - Top line: asset label (e.g. `USDC`), a thin vertical separator (`w-px h-2.5 bg-primary/30`), chain logo (`size-3 rounded-full`), chain name in muted small text.
  - Bottom line: asset full name (e.g. `USD Coin`), muted small text.
- **Right (stacked, right-aligned):** balance (tabular nums), USD value below in muted small text (also tabular nums).

The item key is composite: `${asset}:${chain}`. Same asset on multiple chains creates multiple rows — they're distinct selectable items.

Selected state: `data-checked` set on the row; relies on the shadcn `CommandItem` checked styling.

---

## 6. What feeds this picker

- The `assets` prop is the **From wallet's balances**, already fanned-out per chain (the form fans an asset balance like "1,000 USDC" across the wallet's supported chains using a weight table).
- The picker is **disabled when no From wallet is selected** — there are no balances to show.
- USD values come from `assetPrices[asset]` × parsed balance; the picker doesn't fetch prices itself.

If the upstream wallet has zero balances on any chain, the empty state appears.

---

## 7. What this picker does NOT have

To avoid confusion with the two-pane pickers, the asset picker has none of:

- Rail / categories
- Favorites / stars
- Meta rows (`All`, `Favorites`)
- Counts on rows
- Fixed 700 × 400 size (it's content-sized)
- Viewport-level backdrop (it's panel-scoped)
- Multi-line truncation rules — asset/chain names are always short
- Error state on the trigger — the wallet field handles that upstream

---

## 8. Tailwind reference (full, since this picker is its own shape)

### Trigger chip
```
flex items-center gap-1.5 h-8 rounded-full border border-input bg-background px-2.5
text-sm font-medium shrink-0 cursor-pointer hover:bg-accent transition-colors
disabled:opacity-50 disabled:pointer-events-none
```

### Asset logo on trigger
```
relative size-6 shrink-0 -ml-1
```
Inner image: `size-5.5 rounded-full`.
Chain badge: `absolute -bottom-0.5 -right-0.5 size-3 rounded-full ring-2 ring-background`.

### Overlay wrapper (inside form panel)
```
absolute inset-0 z-50 flex items-center justify-center
```

### Backdrop (transparent click-catcher)
```
absolute inset-0
```

### Panel
```
relative w-[calc(100%-3rem)] rounded-lg border bg-popover shadow-lg ring-1 ring-foreground/10
```

### CommandList
```
min-h-48 max-h-80
```
With inline `style={{ height: 'auto' }}` to let the list grow to fit within those bounds.

### Row outer
```
flex items-center gap-3 w-full
```

### Row logo
```
size-7 rounded-full shrink-0
```

### Row center column
```
flex flex-col min-w-0
```

Top line:
```
text-sm font-medium inline-flex items-center gap-1.5
```
- Asset label is plain text.
- Separator: `w-px h-2.5 bg-primary/30` (a thin vertical line, semi-opaque primary).
- Chain block: `font-normal text-xs text-muted-foreground inline-flex items-center gap-1`.
- Chain logo: `size-3 rounded-full shrink-0`.

Bottom line (full name):
```
text-xs text-muted-foreground
```

### Right (balance) column
```
flex flex-col items-end ml-auto shrink-0
```

- Balance: `text-sm tabular-nums`.
- USD: `text-xs text-muted-foreground tabular-nums`.

---

## 9. Quick checklist for developers

- [ ] Trigger is a chip (`rounded-full h-8`), not a full-width input. Disabled when no From wallet.
- [ ] Trigger asset logo carries a chain-logo "ring badge" at its bottom-right when an asset is selected.
- [ ] Overlay is `absolute` inside the form panel, not `fixed` to the viewport.
- [ ] Backdrop is transparent (click-to-close), not a dim scrim.
- [ ] Panel width is `calc(100% - 3rem)`; height is content-driven within `min-h-48 max-h-80`.
- [ ] List sorted **descending by USD value**, not alphabetical.
- [ ] Search uses manual state (`shouldFilter={false}`) and matches asset code, label, full name, and chain name.
- [ ] Items are `${asset}:${chain}` composite keys — same asset on multiple chains shows as multiple rows.
- [ ] Row layout: logo · (label + chain) over (full name) · (balance) over (USD).
- [ ] Empty state copy: `No assets found.`
- [ ] Selecting an item closes the picker and clears the search query.
- [ ] No favorites, no rail, no counts.
