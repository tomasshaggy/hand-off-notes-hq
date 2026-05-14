# Loans Popover — Handoff

A **multi-select** picker for attaching one or more loans to a transfer. Same two-pane shell as the Transfer Type / From Wallet pickers, but with a tag-input style trigger and checkbox rows.

**Read [hand-off-popover.md](./hand-off-popover.md) first** for layout, backdrop, motion, keyboard, and base Tailwind classes. This doc only covers what's different. The Loans picker is the most divergent of the popovers, so the delta list is longer than for the From/To wallet docs.

---

## 1. Multi-select changes a lot

The single biggest difference: this picker returns **`string[]`**, not a single id. That cascades through the behavior:

- **Selecting an item does NOT close the popover.** Clicking a row toggles it on/off; the popover stays open so the user can pick more loans.
- **Clicking a selected row deselects it.** Same control, idempotent toggle.
- **Each row has a checkbox**, in addition to the row click target. The checkbox is `tabIndex={-1}` so it's purely visual — clicking anywhere in the row toggles selection.
- **The trigger shows chips, not a single label** (see §2).
- **Close is explicit**: backdrop click, Esc, or losing focus. There is no "submit" / "done" button in the popover — selections persist live to the form state as the user toggles.

---

## 2. Trigger is tag-input style

The trigger height **grows** to fit selected loans:

```
h-auto min-h-9 py-1.5
```

instead of the fixed `h-9` used everywhere else.

### Empty state
Muted placeholder `Select`.

### Selected state
A wrapping row of **chips**, one per selected loan, separated by `gap-1.5`. Each chip:
- `<Badge variant="secondary">` with `gap-1.5 pl-2 pr-1 py-0.5`
- **Borrower name** (medium weight, foreground color)
- **Loan ID** (normal weight, muted)
- **X button** at the right — clickable square (`size-4 rounded-sm hover:bg-muted-foreground/20`) with `<X size-3 />`. Has `aria-label="Remove <borrower>"`. Stops propagation so it doesn't trigger the popover.

The X is keyboard-accessible (`tabIndex={0}` + Enter/Space handler). Worth keeping — without it, removing a loan would require opening the popover, finding the row, and unchecking, which is a lot of clicks.

### Error state
Destructive-tinted border (same convention as other pickers). The error text `Required` renders below.

---

## 3. Optional / required label

The field can be marked optional. When `required={false}`, the label renders:

```
Loan (optional)
```

with `(optional)` in `text-muted-foreground text-xs`. Other pickers in this form don't have an optional state — Loan is unique in that.

---

## 4. Rail (currently single-row, but designed to expand)

The rail wrapper is the standard 192 px (`w-48`) two-pane shell, **but only one row is rendered**:

```
All           [count]
```

The code's internal `RailKey` type is `'all' | LoanHealth` and the filtering supports the five health buckets:

```
Impairment
Liquidation
Margin Call
At Risk
Healthy
```

These rows are **not rendered in the rail today**, but the filtering and counting infrastructure is in place. If/when you want to expose health filtering as a rail, add a divider and a `RailRow` per health bucket. Counts are already computed (`healthCounts`).

**Why keep the rail shell for one row?** Visual and behavioral consistency with the other pickers in the form. Operators get a predictable shape across all pickers; future expansion doesn't require restructuring the popover.

---

## 5. Item row anatomy

Each row is wider and denser than the wallet rows (`py-2.5` instead of the default cmdk row height). Three zones:

```
☐  Borrower Name  [health badge]                  [⬡] 1,234.56 ETH
   LN-12345                                            $3,456,789.00
```

- **Left:** `<Checkbox>` (`shrink-0`). Reflects selection. `tabIndex={-1}` — focus stays on the row.
- **Center (stacked):**
  - Top: borrower name + colored health badge (same badge classes as Transfer Type: `inline-flex items-center rounded px-1.5 text-[11px] leading-4 font-medium`, with `healthBadgeColors[l.health]` providing bg + text colors).
  - Bottom: loan ID (`text-xs text-muted-foreground`).
- **Right (stacked, right-aligned):**
  - Top: `<AssetIcon>` (`size-4 rounded-full`) + collateral amount.
  - Bottom: collateral USD value (`text-xs text-muted-foreground`).

USD is computed live: `parseFloat(collateral) * assetPrices[collateralAsset]`. The picker doesn't fetch prices.

---

## 6. Sort

**By health severity order**, not alphabetical:

```
Impairment → Liquidation → Margin Call → At Risk → Healthy
```

Defined by `loanHealthOrder`. The riskiest loans come first because the most common use case is attaching a loan to a transfer in response to a margin call or impairment — those should be at the top.

Within a single health bucket, the source-array order is preserved (no secondary sort).

---

## 7. Search

Single search input at the top of the right pane. Matches against:

- Loan ID (e.g. `LN-12345`)
- Borrower name
- Health bucket name

The placeholder reflects scope:
- `Search by Loan ID or Borrower...` when on `All`.
- `Search <health> loans...` when on a specific health bucket (latent today — only `All` exists in the rail).

Scroll-to-top on every keystroke, same as other pickers.

---

## 8. Empty states

| Situation | Copy |
|---|---|
| Search returned nothing on `All` | `No loans found.` |
| Active rail row has zero loans in that health bucket | `No loans in <health>.` |
| Active rail row has loans but search returns nothing | `No loans found.` |

The second case (`No loans in <health>.`) is latent — it only fires when a health rail row is active, which currently never happens. Keep it; it's free correctness for when the health rail is enabled.

---

## 9. Positioning differences

Popover aligns to **the left edge of the trigger**, not the center:

```
align="start" collisionPadding={16}
```

vs. `align="center"` everywhere else. Reason: the trigger is wide (full-width form input that can grow tall with chips) and aligning to the start gives a more predictable popover position when the trigger has multiple chips.

Same `w-175 h-100` (700 × 400 px) fixed popover size. Same short-viewport modal fallback as the other pickers.

---

## 10. Disabled state

Loans can be marked disabled (e.g. when an upstream condition isn't met). Disabled behavior:

- The trigger becomes `pointer-events-none` and ignores clicks.
- If a `disabledTooltip` is provided, the trigger is wrapped in a tooltip that explains why the field is disabled.
- The trigger keeps the same visual; no opacity reduction, no separate styling.

The tooltip wrapper uses `span tabIndex={0} cursor-not-allowed` around a `pointer-events-none` button so the tooltip can still be triggered by hover/focus on the disabled state.

---

## 11. Tailwind deltas

### Trigger (grows with chips)
```
justify-between font-normal h-auto min-h-9 py-1.5 w-full min-w-0
```
Error variant adds `border-destructive`.

### Trigger inner (chip row)
```
flex flex-wrap items-center gap-1.5 min-w-0
```

### Loan chip
```
gap-1.5 pl-2 pr-1 py-0.5
```
On a `<Badge variant="secondary">`. Borrower span: `font-medium text-foreground`. Loan id span: `font-normal text-muted-foreground`.

### Chip remove (X) button
```
inline-flex items-center justify-center rounded-sm hover:bg-muted-foreground/20 size-4 cursor-pointer
```
Icon: `<X className="size-3" />`. Keyboard handlers: Enter and Space both trigger remove, with `stopPropagation` so the trigger doesn't open.

### Item row (`<CommandItem>`)
```
py-2.5
```
Taller than the default cmdk row to fit the two-line layout + collateral block on the right.

### Item outer
```
flex items-center w-full gap-3
```

### Item right column wrapper
```
flex items-center gap-2 shrink-0
```
With the inner stack: `flex flex-col items-end`.

### Asset icon in row
```
size-4 rounded-full
```

### Health badge
Same classes as the Transfer Type reason badge:
```
inline-flex items-center rounded px-1.5 text-[11px] leading-4 font-medium
```
Pulled from `healthBadgeColors[l.health]` (background + text pair).

### Optional label suffix
```
text-muted-foreground text-xs
```
Rendered inline after `Loan `.

---

## 12. Field-only summary (for Figma annotation)

Self-contained description of the **collapsed field** behavior — chip trigger, inline removal, states. Ignores everything that happens inside the popover. Lift directly for Figma annotations.

### Field shell
- **Optional.** Label reads `Loan (optional)` with the suffix muted. Other fields in this form are required; Loan is the exception.
- **Can be disabled** when an upstream condition isn't met (e.g. transfer type doesn't allow a loan attachment). When disabled, the trigger ignores clicks and is wrapped in a tooltip explaining why.

### Trigger — empty state
- Muted placeholder `Select`, chevron on the right.
- Standard `h-9` height (matches the other form inputs in the row).

### Trigger — selected state
- **Height grows** to fit the chips (`h-auto min-h-9`). With 1–2 loans it stays the same height as other inputs; with more, the chips wrap onto a second/third line and the field gets taller. No clipping, no overflow scroll — the form below shifts down.
- Inside the trigger: a wrapping row of chips, one per selected loan, with `gap-1.5` between them.
- **Each chip contains three pieces:**
  - `Borrower Name` — medium weight, foreground color.
  - `LN-12345` — normal weight, muted.
  - `×` button — small square, hover-tinted, right-aligned inside the chip.
- The chevron stays at the far right of the trigger regardless of how many chips there are.

### Inline chip removal
- Clicking the **×** on a chip removes that loan immediately, **without opening the popover**. The chip disappears and the trigger height re-flows.
- The × is **keyboard-reachable**: Tab into it, then Enter or Space removes the loan.
- Clicking the × does not propagate — the popover stays closed.

### Opening
- Clicking anywhere else on the trigger (chip body, empty space, chevron) opens the popover.
- Toggling loans inside the popover updates the chip row live.

### Error state
- Destructive-tinted border + `Required` text below. Only fires if the field is required upstream and submitted empty — by default the field is optional and this state never appears.

### Disabled state
- Same visual as a normal trigger but non-interactive. Hover/focus reveals a tooltip with the reason.

---

## 13. Quick checklist for developers

Read the [Transfer Type checklist](./hand-off-popover.md#13-quick-checklist-for-developers) first. Then add:

- [ ] Multi-select: value is `string[]`, selecting a row toggles (does not close the popover).
- [ ] Trigger renders one removable chip per selected loan; `h-auto min-h-9` so it grows with selections.
- [ ] Each chip has a keyboard-accessible X button with `aria-label="Remove <borrower>"`.
- [ ] Optional state: label appends ` (optional)` in muted small text when `required={false}`.
- [ ] Rail currently shows only the `All` row, but the filtering supports five health buckets — leave the infrastructure in place.
- [ ] Item row: checkbox (visual, `tabIndex={-1}`) + borrower + health badge + loan ID + collateral icon/amount + USD value.
- [ ] Sort by health severity (`Impairment → Healthy`), not alphabetical.
- [ ] Search matches loan ID, borrower name, and health bucket; placeholder is `Search by Loan ID or Borrower...` on `All`.
- [ ] Empty state: `No loans found.` for search misses; `No loans in <health>.` when a health rail row is empty (latent today).
- [ ] Popover aligns to the start of the trigger, not center.
- [ ] Disabled state wraps the (`pointer-events-none`) trigger in an optional tooltip to explain why.
