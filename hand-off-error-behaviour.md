# Error Behaviour — Handoff

How required-field validation surfaces visually on the New Transfer form (step 1). Covers the single error signal we use, where it appears, when it appears, and what we explicitly **don't** show. This pattern is form-wide — every required field in step 1 follows it.

This doc covers behavior. For per-field anatomy / Tailwind, see the individual picker docs ([hand-off-popover.md](./hand-off-popover.md), [hand-off-from-wallet.md](./hand-off-from-wallet.md), [hand-off-loans.md](./hand-off-loans.md)). For the review and success steps, see [hand-off-review-step.md](./hand-off-review-step.md). The review step has no field errors — it's read-only.

---

## 1. Core rule

Two signals only, nothing else:

1. **Field-level red border** on every required field that's missing or invalid.
2. **One summary message** in the page footer that reads `Required fields`, in destructive color.

No inline "Required" labels under fields. No "Asset is required" / "Amount is required" inline texts. No red border on the wallet-group container. No top-of-form alert banner. No icons.

The reduction is deliberate: with one border per offending field and one short footer message, the user can scan the form and immediately see *where* to act, without reading a wall of red text.

---

## 2. When errors appear

Errors appear **on attempt only**.

- The form tracks a single `attempted` flag, set to `true` the first time the user clicks `Review Transfer`.
- Until `attempted` is `true`, no error styling is shown — even if fields are empty.
- After `attempted` is `true`, errors update **live**: as the user fills a missing field, its border returns to normal and the footer message disappears the moment the form becomes valid.
- `attempted` is **not** reset by editing fields. Once raised, it stays for the life of the step.
- Step 2 (review) and step 3 (success) never display error UI — by the time the user reaches step 2 the form is valid, and step 3 is post-submission.

---

## 3. Which fields participate

| Field | Condition for error border |
|---|---|
| Transfer Type | `attempted && !selectedTypeId` |
| From Wallet | `attempted && !fromWallet` |
| Amount card (asset + amount group) | `attempted && (!asset || !amount)` — single border on the white amount card; covers both asset-missing and amount-missing |
| To Wallet | `attempted && !toWallet` |
| Loan | `attempted && isLiquidationPrep && loanIds.length === 0` — only required when reason is Liquidation Prep |
| Note | `attempted && isNoteRequired && !note.trim()` |

### Note: when is it required?

Note is **required by default**. It flips to optional only when the selected reason is Liquidation Prep:

```
const isNoteRequired = !isLiquidationPrep
```

This matches the label, which reads `Note` (no suffix) by default, and `Note (optional)` only when Liquidation Prep is the reason. The visual border, the label, and the form-validity check all derive from the same predicate — keep them in sync.

### Loan: when is it required?

Loan is **optional by default**. It flips to required only when the selected reason is Liquidation Prep. The label reflects this: `Loan (optional)` by default, `Loan` when Liquidation Prep is selected.

---

## 4. The footer summary message

A small `Required fields` message appears in the page footer, **left of the action buttons**, in destructive text-color.

### When it shows

```
step === 'form' && attempted && !isFormValid
```

### When it disappears

Automatically when the form becomes valid, or when the step changes (back to step 2/3, or after `handleReset`). No manual dismiss.

### Where it lives in the footer

The footer is a 480 px-wide row split into two clusters:
- **Left:** step counter `Step 1 of 2` (muted small text).
- **Right:** the action buttons (`Cancel` + `Review Transfer`).

The `Required fields` message is a sibling of the buttons, inside the right-side cluster, **before** them. It inherits the same `text-xs` weight as the step counter on the left, so the two visually balance the footer.

### Why footer, not inline

Putting the message in the footer means it sits next to the action the user just tried to take. The user sees it at the moment of submission, regardless of which part of the (scrollable) form they're focused on. Inline placement (below Note, below Transfer Type, top-of-form) requires the user to either scroll or look away from where they tried to act.

---

## 5. What this system DOES NOT have

Explicit non-features, for the developer's mental model:

- **No inline "Required" text** under fields.
- **No inline validation copy** inside the Amount card (no "Asset is required", "Amount is required", etc).
- **No red border on the `bg-secondary/80` wallet group container.** Field-level borders only — the container stays neutral so the offending children are isolated and the eye doesn't perceive the entire region as wrong.
- **No top-of-form Alert** or banner.
- **No icons** (warning triangle, info circle, etc).
- **No focus-jump to the first error**. The error appears in place; the user navigates by reading.
- **No on-blur validation**. Errors only appear on attempt. A user can tab through the entire form without ever seeing a red border, until they click `Review Transfer`.
- **No per-error copy distinguishing missing-vs-invalid.** Today every error is "field missing"; if future validations need to communicate something specific (e.g. "Amount exceeds balance"), that's a future enhancement — likely inline text scoped to the specific case, not generic.
- **No screen-reader fallback yet.** The visual signal is borders + summary; we don't currently set `aria-invalid` on the field or surface error context via `aria-describedby`. This is a deliberate v1 simplification — flag for follow-up if a11y matters to the audience.

---

## 6. Implementation notes

- Each picker / select / amount card / textarea takes an `error` boolean (or computes its own conditional class) and applies `border-destructive` to its trigger or container when true.
- The Amount card is special: it's not a picker, so the `border-destructive` is applied to the outer white `<div>` that wraps the input + asset picker. The error condition combines both `!asset` and `!amount`, so a single card border covers both missing pieces.
- The Note Textarea applies `border-destructive` via its `className` prop in both branches (the disabled-in-tooltip branch and the active branch), so the error styling survives the disabled state.
- The form's validity (`isFormValid`) and the footer summary are driven by the same predicates as the field-level errors. Don't fork them — if a new required field is added, update `isFormValid` and pass an `error` prop derived from `attempted && <condition>`. The footer will pick up the new field automatically because it watches `isFormValid`.

---

## 7. Tailwind reference

### Field-level border (all pickers + Amount card + Note Textarea)

```
border-destructive
```

Applied conditionally to the field's trigger / container when `error` is true. Replaces the default `border-input` for that field's outer ring.

### Footer summary message

```
text-xs text-destructive
```

A sibling `<span>` immediately before the action buttons in the right-side footer cluster.

### Footer right-side cluster (where the summary lives)

```
flex items-center gap-2
```

### Footer left-side step counter (for contrast / weight matching)

```
text-xs text-muted-foreground
```

Both the summary message and the step counter share the same `text-xs` size — they balance each other visually across the 480 px-wide footer row.

---

## 8. Quick checklist for developers

- [ ] Every required field gets `border-destructive` on attempt, no inline text.
- [ ] No red border on the `bg-secondary/80` wallet-group container — fields only.
- [ ] `attempted` flag flips to `true` on the first `Review Transfer` click and stays true for the rest of step 1.
- [ ] `Required fields` summary appears in the footer right cluster, left of the buttons, when `step === 'form' && attempted && !isFormValid`.
- [ ] Summary auto-clears when the form becomes valid; no manual dismiss.
- [ ] Note is required by default; optional only when reason is Liquidation Prep. Label and border condition both derive from `isNoteRequired = !isLiquidationPrep`.
- [ ] Loan is optional by default; required only when reason is Liquidation Prep.
- [ ] Amount card's single border covers both missing asset and missing amount.
- [ ] Note error condition applies in both branches (disabled-in-tooltip and active).
- [ ] No errors shown on step 2 (review) or step 3 (success).
- [ ] When adding a new required field: update `isFormValid`, pass `error={attempted && !value}` to the field. No footer change needed.
