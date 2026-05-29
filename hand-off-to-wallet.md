# To Wallet Popover — Handoff

A two-pane picker for selecting the destination wallet of a transfer. Same component as the From Wallet picker (`FromWalletCombobox`), but rendered in **contextual mode** — the rail's meaning depends on the From wallet that's already been chosen.

**Read these first:**
- [hand-off-popover.md](./hand-off-popover.md) for layout, backdrop, motion, keyboard, star affordance, and the base Tailwind classes.
- [hand-off-from-wallet.md](./hand-off-from-wallet.md) for the **item row anatomy** (two-line: name + chain badge over full address), the **trigger** (three pieces), and the **multi-field search**. All of these are identical in the To picker.

This doc only covers what's different from the From picker.

---

## 1. The rail is contextual

The rail has **three rows**:

```
Vaults
Exchange Vault
Address Book
```

- `Vaults` — wallets in the **same workspace** as the From wallet (excluding exchange vaults).
- `Exchange Vault` — Maple-Trading-owned external sub-accounts at the exchange venues (Binance / OKX / Ripple Prime), tagged `EXT-MT` in Fordefi (`isExchangeVault(w)` ⇒ `w.workspaceTag === 'EXT-MT'`).
- `Address Book` — wallets in **any other workspace** (excluding exchange vaults).
- **No `All` row.** No `Favorites` row. No divider.

`Vaults` and `Address Book` are contextual — their membership depends on the From wallet's workspace. `Exchange Vault` is **workspace-independent on purpose**: an exchange deposit is a valid destination regardless of the From workspace, so the rail is always populated. Note the row is named **`Exchange Vault`** (singular). This is the "Mode B — Contextual" pattern from the original two-pane-picker spec, with the exchange rail layered on top.

Why drop `All` and `Favorites` as rail rows? They're *unscoped* views that would mix wallets across the contextual buckets — exactly what the contextual model is trying to prevent. The user shouldn't be able to skip the Vaults / Address Book distinction by clicking `All`. Favorites still exist as a per-row affordance and still appear grouped inside each scope (see §3) — the shortcut survives without the rail row.

---

## 2. Default rail on open

| Situation | Default rail |
|---|---|
| Fresh open | `Vaults` |
| Re-opening with an existing selection | The rail that selection currently falls into (Exchange Vault if it's an `EXT-MT` venue; otherwise Vaults if same workspace as From, else Address Book) |

There's no "favorites if any" fallback — Vaults is always the fresh-open default. It's the most common destination (intra-workspace transfers dominate).

---

## 3. Favorites still exist — but only at the row level

- **No `Favorites` rail row.**
- Stars on individual wallets still work (same hidden-until-hover star button as From Wallet, same global per-user set).
- Inside `Vaults`, `Exchange Vault`, and `Address Book`, starred wallets get the in-scope favorites grouping: a `Favorites` heading at the top, the starred wallets below, a separator, then the remaining unstarred wallets. Same rule as the From picker for in-category favorites.

So a power user inside `Vaults` still sees their starred Vault wallets at the top — they just can't get a flat cross-workspace favorites view from this picker. That's intentional: it would mix categories.

---

## 4. No counts on the rail

`Vaults`, `Exchange Vault`, and `Address Book` rows show **no count badges**. Unlike `All` and `Favorites` in the From picker (which always show live counts), category rows never get counts — and in To mode there are *only* category rows.

If the From wallet's workspace has zero peers (e.g. only one wallet in MIO) and the chain filter eliminates the rest, `Vaults` will appear with an empty pane — that's the correct behavior. Don't hide an empty rail row.

---

## 5. External filters

| Filter | Source | Effect |
|---|---|---|
| `filterChain` | Upstream asset/chain selection | Hides any wallet not on the same **chain family**. |
| `excludeAddress` | The **From** wallet | Hides the From wallet so the user can't pick From == To. |

Same precedence as the From picker: chain family → exclude-address → rail filter → search.

### Chain filtering is by family, not by specific chain

`filterChain` matches the **chain family** (`evm` / `solana` / `btc`), so:

- **EVM From wallet** → all EVM destinations are eligible. Ethereum, Base, Arbitrum, and Monad wallets are all cross-compatible at the address level, so they all appear in either Vaults or Address Book depending on workspace.
- **Solana From wallet** → only Solana destinations.
- **Bitcoin From wallet** → only Bitcoin destinations.

Note that `excludeAddress` here is the From wallet (vs. the To wallet in the From picker). Same mechanism, opposite direction.

The `Exchange Vault` rail is workspace-independent but **not filter-independent**: `filterChain` and `excludeAddress` still apply, so an exchange sub-account only appears if it's on the From wallet's chain family. The rail being "always populated" is about workspace scope, not chain scope.

---

## 6. Empty states

Reduced to two cases (no `Favorites` meta row means no "no-favorites-yet" copy for the To picker):

| Situation | Copy |
|---|---|
| Search returned nothing | `No wallets found.` |
| Rail scope eliminated by filters/exclude | `No wallets found.` |

If you want a sharper copy for the second case (e.g. `No wallets in Address Book on this chain.`), that's a future enhancement — current implementation falls back to the generic empty copy.

---

## 7. Sort, search, item row, trigger

All identical to the From Wallet picker. See [hand-off-from-wallet.md](./hand-off-from-wallet.md):
- §2 Item row anatomy (two-line)
- §5 Search matches multiple fields
- §6 Sort (alphabetical with starred pinned first; no workspace-order primary sort)
- Trigger displays three pieces (name + chain badge + truncated address)

---

## 8. Tailwind deltas from the From picker

Only the rail differs visibly. Everything else is identical.

### Rail wrapper

Same as From picker:
```
flex flex-col gap-0.5 w-52 shrink-0 border-r p-1.5
```

### Rail rows

Same `<RailRow>` component as everywhere else. The To rail just passes **no `count` prop** to either row — without a count, the right-side number doesn't render.

There's **no divider element** (`<div className="my-1 h-px bg-border" />`) in the To rail since there are no meta rows to separate from category rows.

---

## 9. Quick checklist for developers

Read both the [Transfer Type checklist](./hand-off-popover.md#13-quick-checklist-for-developers) and the [From Wallet checklist](./hand-off-from-wallet.md#8-quick-checklist-for-developers) first. Then add:

- [ ] Rail has exactly three rows: `Vaults`, `Exchange Vault` (singular), and `Address Book`. No meta rows, no divider.
- [ ] `Vaults` = same workspace as From wallet (minus exchange vaults). `Address Book` = any other workspace (minus exchange vaults). `Exchange Vault` = `EXT-MT`-tagged venue sub-accounts, workspace-independent (always populated regardless of From workspace).
- [ ] No counts on any rail row.
- [ ] Default rail on fresh open: `Vaults`. Re-opening with a selection: the rail that selection falls into (Exchange Vault if it's an `EXT-MT` venue).
- [ ] Stars + in-scope Favorites grouping still work inside Vaults, Exchange Vault, and Address Book — only the `Favorites` rail row is removed.
- [ ] `excludeAddress` is the **From** wallet (not the To wallet, as it is in the From picker).
- [ ] Empty state copy: `No wallets found.` (single case in v1).
- [ ] Item row, trigger, search, sort: unchanged from the From picker.
