# Two-Pane Picker — Handoff

A reusable picker pattern for selecting one item from a categorised set. Used wherever the user must browse a finite taxonomy *and* search a long list at the same time. In our app it powers three fields (a categorised type picker and two wallet pickers), but the pattern is intentionally generic.

This doc describes the **behavior, states, and interactions** of the pattern. It does not include code or domain-specific category names. Implement in your stack of choice — the spec is the contract.

---

## 1. Purpose & when to use

Use this pattern when **all** of the following are true:

- The user picks one item from a list that's too long to browse linearly (dozens to hundreds).
- The items can be cleanly categorised into a small, stable set of buckets (≤ 8 categories typical).
- Users span two personas: power users who repeat the same handful of items, and occasional users who need the categorisation as scaffolding to find anything.
- Categories may or may not be **contextual** (their meaning can depend on another field's value — see *Modes* below).

Do **not** use when:

- The list is short enough that a flat dropdown works (< 15 items, no real growth).
- There's no meaningful categorisation (single-dimension search is enough — use a command palette).
- Multi-select is required (this pattern is single-select).

---

## 2. Anatomy

The picker is a **trigger button** that opens a **popover** containing two panes:

```
┌──────────────────────────────────────────────────────┐
│ <Field label>                                        │
│ ┌──────────────────────────────────────────────────┐ │
│ │ <current selection or placeholder>            ▾ │ │ ← trigger
│ └──────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
                  │ click
                  ▼
┌──────────────────────────────────────────────────────┐
│ ┌──────────────┬────────────────────────────────┐    │
│ │              │ 🔍 Search…                    │    │
│ │   LEFT RAIL  ├────────────────────────────────┤    │
│ │  (vertical)  │                                │    │
│ │              │  RIGHT PANE                    │    │
│ │              │  (scrollable item list)        │    │
│ │              │                                │    │
│ └──────────────┴────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

**Dimensions** (fixed):
- Popover: **700 px × 400 px**. No `min-*` or `max-*` constraints — the popover is exactly this size, not "at least" or "up to."
- Left rail: 192 px wide (`w-48`), full popover height.
- Right pane: fills the remainder (~508 px), full popover height.
- Border between rail and pane (1 px).
- The inner scroll list explicitly overrides the host component's default max-height cap so the list fills the full 400 px.

**Why fixed**: predictable silhouette across every instance of the pattern. Variable height creates jitter as filters change list length; users learn the size and start to ignore it.

---

## 3. Left rail

The rail is the **category navigator**. Each row is a button that filters the right pane to a single category. Exactly one rail row is active at any time.

### Row anatomy

- Plain text label, left-aligned.
- Optional **count badge** on the right (muted, tabular numerals).
- Active state: filled background tint. Inactive: transparent with subtle hover background.
- Active state must be marked with `aria-current` for screen readers.
- No icons, no colored pills, no font-weight changes between active/inactive (prevents layout shift).

### Row types

Two kinds of rows can appear, in this order:

1. **Meta rows** — pinned at the top. These are *aggregate* views, not categories.
   - `Favorites` — items the user has starred. Shows a live count.
   - `All` — every item. Shows a live count.
2. **Category rows** — the actual categories the list is split into. No count by default (see *Counts* below).

A 1 px divider line separates meta rows from category rows when both groups are present.

### Counts

- Meta rows (`Favorites`, `All`) **always show a count**.
- Category rows **do not show counts** by default.
- Counts must be **live** — recompute when any upstream filter changes (see *External filters* below). If the user has 3 favorites but a chain filter reduces this to 0, the rail shows `Favorites 0`.

### Interaction

- **Click** to switch. Right pane updates immediately.
- No hover-to-preview — hover-to-switch creates a "diagonal mouse" bug where the pane changes mid-cursor-trajectory toward the item the user actually wanted.
- Keyboard navigation in the rail is optional in v1; if added, ↑/↓ should switch rows while focus is in the rail.

---

## 4. Right pane

The right pane shows the items in the currently-selected rail category. It has its own search input at the top.

### Search

- Search is **scoped to the current rail row**. Typing while `Category A` is selected filters within Category A only.
- The placeholder reflects scope, e.g. `Search <Category A>…`, `Search favorites…`, `Search all…`. Operators see immediately *what* they're searching.
- To search across categories, the user clicks `All` (when that meta row exists).
- **On every keystroke**, the list must scroll back to top so the first match is visible. Otherwise users who scrolled down then started typing see "nothing happened." No animation — instant snap.

### Item list

- Vertically scrollable inside the pane.
- Each item is a single row with whatever fields the domain needs (a label is required; everything else is optional).
- Selected item gets a visible checkmark or filled state.
- Clicking an item selects it and **immediately closes the popover** (no separate confirm step).

### Empty states

The empty-state copy adapts to **why** the list is empty. Three cases:

| Situation | Copy |
|---|---|
| Search returned nothing | `No <items> found.` |
| Favorites row, user has zero favorites at all | `No favorites yet. Star a <item> to add it.` |
| Favorites row, user has favorites but an upstream filter eliminates them all | `No favorites <under this filter>.` |

The third case is load-bearing: it tells the user the filter is the cause, not their behavior.

---

## 5. Favorites mechanism

Items can be starred to add them to a global per-user `Favorites` set.

### Where favorites appear

1. **In the `Favorites` meta row** (when it exists) — flat list of all starred items across all categories.
2. **Inside any category** that contains at least one favorited item — when the user selects a category in the rail, that category's pane is grouped:
   - `Favorites` heading
   - Starred items (sorted alphabetically)
   - Full-width separator line
   - The remaining unstarred items in that category
3. If the category has zero favorites, the pane shows a flat unheaded list.

This in-scope favorites pattern is important: a power user inside `Category A` should see their A-favorites at the top *without* having to switch to the `Favorites` meta row. The shortcut works at every level.

### Star affordance

- Star icon on the right of each row.
- Hidden by default; visible on row hover and on keyboard focus.
- Filled (amber) when starred, outlined when not.
- Has an `aria-label` describing the action (`Add <item> to favorites` / `Remove <item> from favorites`) — tooltips alone don't cover screen readers before hover.
- Click must not propagate to the row (clicking the star toggles, it does not select the item).

---

## 6. Modes

The pattern supports multiple **modes** — variations in rail content and default behavior. Modes are determined by the host field's role, not by user input. Three modes have shipped:

### Mode A — Standard

The default. Used when categories are static and independent of other fields.

- Rail: `Favorites | All | <category 1> | <category 2> | …`
- Default on fresh open: `Favorites` if the user has any, else `All`.
- Editing an existing selection: open with that selection's category preselected.

### Mode B — Contextual

Used when categories are **derived from another field's value**. The category set is fixed in size, but the *membership* of each category depends on context (e.g. "items in the same group as field X").

- Rail: `<category 1> | <category 2>` only — no `Favorites` row, no `All` row.
- Default on fresh open: a fixed primary category (the natural starting point in the model).
- Editing an existing selection: open with the category the selection currently falls into (which may differ from the default).
- Favorites still appear *inside* each category (per §5.2) — the user shortcut survives without the rail row.

The reason `Favorites` and `All` are dropped in this mode: they're *unscoped* views that would mix items across the contextual categories, which is exactly what the contextual model is trying to prevent. The wiki/decision-doc layer of the project should record this choice explicitly when a new contextual instance is added.

### Mode C — Category-aware standard

Variant of A used when the host field doesn't have a notion of context but you still want users to think "category first, item second." Same rail as A; the only difference from A is that *editing an existing selection* opens with that selection's category (instead of `Favorites`/`All`).

Picking a mode = answering two questions:
1. Is there an upstream context that changes what the categories *mean*? → B.
2. Otherwise: do you want re-opening a filled field to preselect the item's category? Yes → C. No → A.

---

## 7. Positioning

The picker's positioning is **viewport-adaptive**:

| Viewport height | Behavior |
|---|---|
| **> 1080 px** | Anchored popover. Sits below the trigger; flips above if no room. Horizontally aligned to the centre of the trigger. Collision-padded so it never clips a screen edge. |
| **≤ 1080 px** | Modal-style. `position: fixed` centered in the viewport. Backdrop (semi-transparent black, click to close) covers the rest of the screen. |

The threshold can be tuned, but the *principle* matters: on a tall viewport, the spatial link between trigger and popover is useful; on a short viewport, the anchored popover would overflow or flip into awkward positions, so the modal pattern dominates.

Resize listener must update the state — if the user resizes the window while the popover is open, the positioning should adapt (or close the popover safely; either is acceptable, just don't leave it half-positioned).

---

## 8. External filters

The picker often participates in a form where **other fields constrain what's selectable**. Examples we've handled:

- A "type" upstream of the picker that restricts the list to compatible items.
- A "from-side" value that must be excluded from a "to-side" picker (preventing self-selection).

Rules:

1. External filters apply **before** rail filtering. The full list is reduced to "what's eligible" first, then the user picks a rail row within that.
2. **Counts must respect external filters.** `Favorites N` and `All N` shrink honestly.
3. **Empty rail rows still appear.** Don't dynamically hide a category row when an external filter reduces its contents to 0 — rail stability is more important than saving the user one click. The empty-state copy in the pane handles the dead-end clearly.

---

## 9. Selection & trigger states

The trigger button reflects the picker's three states:

| State | Trigger appearance |
|---|---|
| No selection | Muted placeholder text (`Select…`). |
| Has selection | The selected item's primary label, plus the most useful secondary metadata (e.g. a categorical badge, or a piece of identifying detail). Truncates if needed; never wraps. |
| Validation error | Border tinted with the error colour. The trigger does **not** display the error message itself; render that separately below. |

Clicking the trigger opens the popover regardless of state. Re-opening with a value present should follow the mode rules in §6.

---

## 10. Accessibility checklist

- Active rail row marked with `aria-current="true"`.
- Star buttons have descriptive `aria-label`s; tooltips alone are insufficient.
- All interactive elements show a visible focus ring (rail rows, items, star buttons, search input). The focus ring must be the same style across the picker for consistency.
- Focus management: opening the popover moves focus to the search input. Closing returns focus to the trigger.
- `Esc` closes the popover; arrow keys navigate the item list (cmdk-style behavior).
- Hover effects must not be the *only* affordance — they all have keyboard/focus equivalents.
- Touch-target sizes inside the picker are **not** 44 px in v1 — this is a desktop-first dense workflow. Document this if you're shipping to a tablet/touch surface; switch to a sheet/dialog at narrow viewports rather than scaling up the popover.

---

## 11. Animation & motion

- The pattern is **frequent-interaction UI**. Skip non-essential animation.
- Popover open/close: a brief fade + small Y offset is fine (use your library's defaults). 150 ms ease-out maximum.
- Rail row activation: no animation. Background colour change is instant.
- Right-pane content swap on rail click: no animation. The change should feel like flipping a switch.
- Scroll reset on search keystroke: instant, no smooth scroll.
- Respect `prefers-reduced-motion` for any motion you do add.

---

## 12. What *not* to do

A few patterns we explicitly considered and rejected:

- **Two separate fields** (one to pick the category, one to pick the item). Hides the item list until the category is committed; doesn't fit power users who know the item name.
- **Single flat searchable list** (no rail at all). Works for shallow taxonomies; fails when there are hundreds of items and a mixed audience that includes browsers.
- **Horizontal pill row of categories** above the list. Fine for ≤ 4 categories; breaks (overflow-x-scroll smell, then truncation) as soon as the category set grows or includes long labels.
- **Tree-style nested expansion** in the rail. Adds depth the data doesn't actually have; categories here are flat, not hierarchical.
- **Inline "create new category/item"** in the picker. Mid-transfer is the wrong moment to make permanent taxonomy decisions; route that through a separate admin surface owned by the relevant team.

---

## 13. Quick checklist for implementers

- [ ] Trigger button with three states (empty / selected / error).
- [ ] Popover or modal, switched by a viewport-height threshold.
- [ ] Left rail with active row, `aria-current`, optional count badges on meta rows only.
- [ ] Right pane with scoped search, context-aware placeholder, vertically scrolling list.
- [ ] Favorites: global star set, `Favorites` meta row (Mode A/C only), in-scope `Favorites` heading + separator inside each category.
- [ ] Scroll-to-top on every keystroke in search.
- [ ] Empty-state copy handles three cases (no results, no favorites at all, no favorites under current filter).
- [ ] Counts on meta rows are live and respect external filters.
- [ ] Mode-aware defaults: fresh open vs editing-existing-selection differ per mode.
- [ ] Click on the star does not select the row.
- [ ] Focus moves to search on open; returns to trigger on close.
- [ ] Resize listener updates positioning behavior.
- [ ] `prefers-reduced-motion` honored.

---

If you implement this, the litmus test for whether you got the pattern right: a user who has used one instance of the picker should be able to use a brand new instance — with a completely different category set — without re-reading any docs. That's the bar.
