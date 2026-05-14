# Success Step — Handoff

The post-submission state of the New Transfer flow. Renders **the same summary block as the review step**, with a `Successfully Submitted` banner above it and different page-bar actions below.

**Read [hand-off-review-step.md](./hand-off-review-step.md) first** for the summary block structure (amount hero card, From/To group, context-rows group, all detail-row anatomy and Tailwind). This doc only covers what's different.

---

## 1. What's reused from the review step

- The `transferSummary` block is rendered **identically** to step 2. Same amount hero card. Same two grouped white cards inside the `bg-secondary/80` wrapper. Same `ReviewDetailRow` for every row. Same conditional rendering for Type / Loan / Collateral / Note.
- Read-only. No editing here either.
- The scroll container, centered wrapper, and spacing all stay the same.

The success state is a **wrapper** around the review summary, not a different layout.

---

## 2. The success banner

A short, centered banner sits **above** the summary block:

```
[✓]  Successfully Submitted
```

- A green circle (`bg-green-600`) with a white check icon inside.
- A soft green ring (`bg-green-500/40`) sits behind the circle and animates outward.
- Banner text `Successfully Submitted` in `text-green-700`.
- Both the circle and the text animate in on mount.

The banner sits inside the same scroll container as the summary, so it scrolls with the content. It's not pinned.

---

## 3. Page-bar actions (different from review)

The page footer keeps the same layout (480 px-wide row, step counter on the left, actions on the right), but the buttons change:

| Step | Left | Right |
|---|---|---|
| Review (step 2) | `Back` (outline) | `Confirm Transfer` (primary) |
| **Success** | **`Close` (outline)** | **`View Transfer` (primary)** |

- `Close` runs `handlePageExit` — resets the form and navigates back to `/workflows/asset-movement`.
- `View Transfer` navigates to the asset-movement list with the new transfer id in a query param (e.g. `?focus=tx-12345`), so the consuming page can scroll to / highlight the created row. If no id was returned from `onTransferCreate`, it falls back to the plain list URL.
- The step counter region on the left is **replaced** in this state by a `New Transfer` secondary button (start another transfer), instead of the `Step N of 2` text. This is the only step where the left cluster is a button rather than a label.
- No `Required fields` summary message — error UI only runs on step 1.

---

## 4. Animations

Four named utilities power the banner's entrance. Names referenced here so they can be reused for future success surfaces (e.g. loan repayment confirmation):

| Class | Target | Effect |
|---|---|---|
| `animate-success-ring` | Soft green halo behind the check circle | Expanding ring, fades out |
| `animate-success-pop` | Green check circle | Scale-up entrance |
| `animate-success-check` | The white check icon inside the circle | Stroke/scale in |
| `animate-success-fade-up` | The `Successfully Submitted` text | Fade + slight Y translate |

Keep these utility names centralised — if you ship a second success surface, use the same animations so the motion language stays consistent.

`prefers-reduced-motion` should suppress all four; that's a project-wide concern, not specific to this state.

---

## 5. Tailwind reference (banner only)

The rest of the surface uses the review-step Tailwind reference verbatim.

### Banner wrapper

```
flex justify-center pt-4 gap-2 items-center
```

### Check circle (outer relative box)

```
relative size-7
```

### Animated halo

```
absolute inset-0 rounded-full bg-green-500/40 animate-success-ring
```

Note `aria-hidden` on this element — it's pure decoration.

### Solid circle (parent of the check icon)

```
relative size-7 rounded-full bg-green-600 flex items-center justify-center animate-success-pop
```

### Check icon

```
size-5 text-white animate-success-check
```

With `strokeWidth={3}` on the Lucide `<Check>` component for a slightly bolder mark — the default 2-stroke reads too thin inside a small circle.

### Banner text

```
text-base text-green-700 animate-success-fade-up
```

---

## 6. Reset & re-entry

- The form state is **not** automatically cleared when the success banner appears. The summary needs the data to render. Clearing happens when the user clicks `Close` (via `handlePageExit` → `handleReset`) or `New Transfer` (which calls `handleReset` directly and stays on the page).
- Pressing `View Transfer` does **not** clear — it leaves the form populated (so a back-navigation returns the user to the populated success screen rather than an empty form). Project decision: re-entry into the New Transfer route from elsewhere should always start fresh, which is handled at the route level, not here.

---

## 7. Quick checklist for developers

Read the [Review step checklist](./hand-off-review-step.md#12-quick-checklist-for-developers) first. Then add:

- [ ] The summary block (`transferSummary`) is identical to step 2 — no per-step variant.
- [ ] Success banner sits **above** the summary, inside the same scroll container.
- [ ] Banner has four animated pieces: outer ring (`animate-success-ring`), solid circle (`animate-success-pop`), check icon (`animate-success-check`), text (`animate-success-fade-up`).
- [ ] Page-bar actions: `Close` (outline) + `View Transfer` (primary).
- [ ] Left cluster of the footer is the `New Transfer` secondary button — not the step counter.
- [ ] `View Transfer` includes the new transfer id as `?focus=...` when available; falls back to the plain list URL when not.
- [ ] No error UI shown — error system only runs on step 1.
- [ ] Form state stays populated on this screen; `handleReset` only runs on `Close` / `New Transfer`.
