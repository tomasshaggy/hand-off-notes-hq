# Review Step — Handoff

Step 2 of 2 of the New Transfer flow. A **read-only confirmation screen** that summarises everything entered on the form and asks the user to commit. No fields here, no editing in place — to change anything the user goes back to step 1.

The same summary block is reused on the success screen (with a green check + "Successfully Submitted" banner above it), so this doc covers both implicitly.

---

## 1. Purpose

- One screen, one decision: **Confirm** or **Back**.
- Everything the user typed is rendered in a stable, grouped layout so they can scan-verify before sending.
- Order of information mirrors how a human would read a wire transfer: who → who, why, how, with what loan, on what policy, and a free-text note last.

---

## 2. Page actions (bottom bar)

The page-bar action area swaps based on step:

| Step | Left | Right |
|---|---|---|
| Form (step 1) | `Cancel` (outline) | `Review Transfer` (primary) |
| **Review (step 2)** | **`Back` (outline)** | **`Confirm Transfer` (primary)** |
| Success | `Close` (outline) | `View Transfer` (primary) |

- `Back` returns to step 1 with all field values **preserved** — no reset.
- `Confirm Transfer` triggers submission and transitions to the success state. There is no second-stage spinner on the button today; if backend latency becomes user-visible, add a pending state.
- Step counter "**Step 2 of 2**" sits in the same footer region.

---

## 3. Layout

Vertically stacked, inside a centered container (`flex-1 min-h-0 overflow-auto flex justify-center`):

```
┌─────────────────────────────────────────────────┐
│           [asset icon]  1,234.5 USDC            │   ← Amount hero (white card)
│                  $1,234.50                      │
├─────────────────────────────────────────────────┤
│ ╭─────────────────────────────────────────────╮ │   ← bg-secondary/80
│ │ Group 1: From / To (white bordered card)    │ │     wrapper
│ │                                             │ │     p-3, gap-4
│ │ Group 2: Reason / Type / Loan(s) /          │ │
│ │          Collateral / Fordefi Policy /      │ │
│ │          Note (white bordered card)         │ │
│ ╰─────────────────────────────────────────────╯ │
└─────────────────────────────────────────────────┘
```

Two visual layers:
- **Outer wrapper** — soft secondary background (`bg-secondary/80`), padding `p-3`, `gap-4` between the two groups. Same surface treatment used in the form step's "wallet + amount" group, for visual continuity.
- **Inner groups** — white bordered cards (`bg-white rounded-md border`). Rows inside each card are separated by `<Separator />` lines.

The amount hero card sits **above** the secondary wrapper, not inside it — it's a hero, not a row.

---

## 4. Amount hero card

A small white card, centered content:

```
[round asset icon]  1,234.5  USDC
            $1,234.50
```

- Asset icon: `size-6 rounded-full`.
- Amount value: `text-[22px] font-medium leading-none`.
- Asset label (e.g. `USDC`): `text-[22px] font-medium text-muted-foreground leading-none` — same size as the value but muted so the eye lands on the number first.
- USD value below: `text-xs text-muted-foreground` with `mt-1.5`.

The amount is locale-formatted (`toLocaleString('en-US')`) but the asset label is whatever short label the asset meta provides (e.g. `ETH`, `USDC`, `WBTC`).

---

## 5. Detail row anatomy

Every row in both groups uses the same `ReviewDetailRow` component:

```
flex items-center justify-between text-sm py-2 px-3
```

- **Label (left):** `text-muted-foreground shrink-0`. Single word, no colon.
- **Value (right):** `text-right`. Right-aligned, can be a single line of text or a small stacked block (see specific rows below).

Rows are separated inside a card by `<Separator />` (the standard 1 px shadcn separator).

---

## 6. Group 1 — From / To

Two rows. Each value is **two-line, right-aligned**:

```
From               [MIO] MIO Treasury  ⬡ ETH
                   0x1234…5f97

To                 [PPOA] Trading Hot  ◎ SOL
                   0x9a2b…7c11
```

- **Top line:** wallet display name (with workspace bracket) + `<ChainBadge>` for the wallet's chain.
- **Bottom line:** full address, muted (`text-xs text-muted-foreground`), truncating if needed.

The wallet display name uses the same `walletDisplay()` helper as the wallet popovers — so what you saw in the picker chip is exactly what shows here.

---

## 7. Group 2 — context rows

Rows in order. Some are **conditional** — they only render when the field has a value:

| Row | Always? | Value rendering |
|---|---|---|
| **Reason** | Yes | Reason label, `font-medium`. |
| **Type** | Only if a type was selected | Type label, plain weight. |
| **Loan / Loan N** | Only if 1+ loans attached | See §8. |
| **Collateral** | Only when reason is **Liquidation Prep** AND **exactly one loan** is attached | Indented under the loan, see §8. |
| **Fordefi Policy** | Yes (constant value from `NEW_TRANSFER_FORDEFI_POLICY_NAME`) | Plain text. |
| **Note** | Only if note is non-empty | Right-aligned, `max-w-[260px]`, wraps. |

The conditional rows still get their `<Separator />` only when they render — no orphan separators when a row is hidden.

---

## 8. Loan rendering rules

Loans are rendered one row each. Two variations:

### One loan attached
- Row label: `Loan`.
- Value (stacked):
  - Top: loan ID (e.g. `LN-12345`).
  - Bottom: borrower name, muted small text.

### Multiple loans attached
- Each loan gets its own row, labelled `Loan 1`, `Loan 2`, `Loan 3`, … in attachment order.
- Same stacked value: loan ID over borrower.
- No grouping / heading — just sequential rows.

### Liquidation Prep with exactly one loan
Adds a **Collateral** row immediately after the loan:

```
Loan               LN-12345
                   Galaxy US

  ↳ Collateral     1,234.5 ETH
```

- Label uses an indent indicator: `<CornerDownRight className="size-4 text-muted-foreground" />` + the word `Collateral`, with `gap-2` between them.
- The indent indicator signals "belongs to the loan above" — purely visual.
- Value is the loan's collateral string as-is (e.g. `1,234.5 ETH`).
- **Multi-loan Liquidation Prep does not show Collateral rows** — the ambiguity (which loan's collateral?) is sidestepped by simply not showing it. If you ever need to show collateral per loan, group each loan + its collateral as a pair.

---

## 9. Read-only contract

- Nothing on this screen is interactive except the page-bar buttons (`Back`, `Confirm Transfer`).
- No fields, no popovers, no inline edit, no copy buttons. If the user spots a mistake, they go back.
- Addresses are not click-to-copy here (consider adding later — operators have asked for it in other screens, but it's out of scope for v1).

---

## 10. Success state (transitions out)

When `Confirm Transfer` is clicked, the same `transferSummary` block is re-rendered with a banner above it:

```
✓  Successfully Submitted
```

- Green check icon in a green-600 circle, with a soft green ring animation.
- Text in `text-green-700`, with a small fade-up animation.
- The summary content below the banner is **identical** to the review screen.
- Page actions switch to `Close` + `View Transfer`.

The animation classes (`animate-success-ring`, `animate-success-pop`, `animate-success-check`, `animate-success-fade-up`) are local utilities — keep them centralised so future success screens (e.g. loan repayment confirmation) can reuse the same motion language.

---

## 11. Tailwind reference

### Outer scroll container
```
flex-1 min-h-0 overflow-y-auto
mt-4
```

### Centered wrapper (page-level)
```
flex-1 min-h-0 overflow-auto flex justify-center
```

### Amount hero card
```
bg-white rounded-md px-3 pb-4 pt-3 flex flex-col items-center gap-0.5 min-w-0
```
Asset icon line:
```
flex items-center justify-center gap-1 min-w-0
```
Asset icon: `size-6 mr-1.5 rounded-full`.
Amount + label: `text-[22px] font-medium leading-none` (label adds `text-muted-foreground`).
USD: `text-xs mt-1.5 text-muted-foreground`.

### Outer secondary wrapper
```
bg-secondary/80 rounded-md p-3 flex flex-col gap-4 min-w-0
```

### Group card (white, bordered)
```
bg-white rounded-md border min-w-0
```

### Detail row
```
flex items-center justify-between text-sm py-2 px-3
```
Label: `text-muted-foreground shrink-0`. Value: `text-right`.

### From / To value (two-line, right-aligned)
```
flex flex-col items-end gap-1 truncate min-w-0 justify-end
```
Top line wrapper: `flex justify-end items-center gap-1.5`.
Bottom line (address): `text-xs text-muted-foreground truncate`.

### Loan value (stacked)
```
flex flex-col
```
Top span: plain. Bottom span (borrower): `text-xs text-muted-foreground`.

### Collateral row label (with indent indicator)
```
flex items-center gap-2
```
With `<CornerDownRight className="size-4 text-muted-foreground" />` preceding the label text.

### Note row value
```
text-sm text-right max-w-[260px]
```

### Success banner
Wrapper:
```
flex justify-center pt-4 gap-2 items-center
```
Check icon ring (animated): `absolute inset-0 rounded-full bg-green-500/40 animate-success-ring`.
Inner circle: `relative size-7 rounded-full bg-green-600 flex items-center justify-center animate-success-pop`.
Check icon: `size-5 text-white animate-success-check` with `strokeWidth={3}`.
Text: `text-base text-green-700 animate-success-fade-up`.

---

## 12. Quick checklist for developers

- [ ] Review step is **read-only** — no fields, no inline edit.
- [ ] Page actions: `Back` (returns to step 1, preserving state) + `Confirm Transfer` (submits).
- [ ] Step counter shows "Step 2 of 2" in the footer.
- [ ] Amount hero card sits **above** the secondary wrapper; detail groups sit **inside** it with `gap-4`.
- [ ] Two detail groups: From/To, then Reason/Type/Loan(s)/Collateral/Fordefi Policy/Note.
- [ ] All rows use `ReviewDetailRow`: muted label left, right-aligned value, `py-2 px-3`.
- [ ] Conditional rows (Type, Loan(s), Collateral, Note) render with their `<Separator />` only when present.
- [ ] Loan labelling: single loan → `Loan`; multiple → `Loan 1`, `Loan 2`, …
- [ ] Collateral row only renders for Liquidation Prep with exactly 1 loan; uses a `CornerDownRight` indent indicator.
- [ ] Note value capped at `max-w-[260px]` and wraps.
- [ ] Success state reuses the same summary block with a green check banner above and updated page actions (`Close` + `View Transfer`).
