# From Wallet Popover — Handoff

A two-pane picker for selecting the source wallet of a transfer. Trigger button opens a popover with a workspace rail on the left and a scoped, searchable wallet list on the right. Single-select.

**This doc only covers what's different from the Transfer Type popover.** For everything else — layout & size, backdrop, rail mechanics, scoped search, star affordance, keyboard & focus, motion, base Tailwind classes — read [hand-off-popover.md](./hand-off-popover.md) first. The rules in that doc apply unless overridden below.

The same component (`FromWalletCombobox`) is also used as the To-side picker with a different rail (`Vaults` / `Address Book`). The To-side mode is documented separately in `hand-off-to-wallet.md` (not yet written). This doc covers the **From-side only**.

---

## 1. Categories are workspaces

Rail order, top to bottom:

```
All           [count]
Favorites     [count]
─────────────────────   ← 1 px divider
MIO
PPOA
Maple Trading
```

- Three workspace rows instead of five reason rows.
- `Maple Trading` is **not a single field match** — it bundles a set of entities (Maple Trading, Binance, OKX, Ripple Prime, etc.) defined by `MAPLE_TRADING_WORKSPACE_ENTITIES`. New entities in that bundle land here automatically; new top-level workspaces need a new rail row.
- Otherwise the rail follows the same rules as Transfer Type (meta rows with live counts, category rows without, active row tinted, no hover-to-switch).

### Rail width

Rail is **208 px (`w-52`)** here, not 192 px, so "Maple Trading" doesn't wrap. Everything else in the rail (padding, gap, row classes) is identical to the Transfer Type popover.

---

## 2. Item row anatomy

Each item is **two lines** (denser than the Transfer Type single-line item):

```
Wallet Display Name  [chain badge]      ★
0x1234…5f97
```

- **Top line:** wallet display name + `<ChainBadge>` showing which chain the wallet lives on.
- **Bottom line:** full address, muted small text (`text-xs text-muted-foreground`).
- Star button on the right (same hidden-until-hover behavior as Transfer Type, see [hand-off-popover.md §8](./hand-off-popover.md)).

**No workspace badge inside the row.** The Transfer Type popover shows a colored reason badge on every item; the From Wallet popover does not show a workspace badge — the workspace is already implied by the active rail row, and the chain badge is what users actually need to disambiguate identically-named wallets.

### Trigger displays three pieces

```
Wallet Display Name  [chain badge]  0x1234…5f97 truncated
```

vs. two pieces in Transfer Type (label + badge).

---

## 3. External filters

The From Wallet picker participates in a form where other fields constrain what's selectable. Two filters apply, both **before** rail filtering:

| Filter | Source | Effect |
|---|---|---|
| `filterChain` | Upstream asset/chain selection | Hides any wallet not on the same **chain family**. |
| `excludeAddress` | The To-side wallet (when picker is From) | Hides that specific wallet so the user can't pick From == To. |

### Chain filtering is by family, not by specific chain

`wallet.chain` and `filterChain` are both **chain families** — `'evm' | 'solana' | 'btc'` — not specific chain IDs like `ethereum` or `base`. So:

- If the upstream asset is on **any EVM chain** (Ethereum, Base, Arbitrum, Monad), the picker shows **all EVM wallets** — regardless of which specific EVM chain each wallet is keyed to. EVM wallets are cross-compatible at the address level.
- If the upstream asset is on **Solana**, only Solana wallets are shown.
- If the upstream asset is on **Bitcoin**, only Bitcoin wallets are shown.

This matters because the asset's specific chain (e.g. Monad USDC) is a stricter constraint than the wallet picker enforces. The asset picker handles per-chain selection within a wallet's balances; the wallet picker only asks "can this wallet receive on this address format?".

### Rules

- All `count` values on `All` and `Favorites` shrink to reflect both filters honestly.
- Empty category rows still appear in the rail — stability matters more than saving one click.
- Filter precedence: chain family → exclude-address → rail filter → search.

---

## 4. Empty states (one extra case)

| Situation | Copy |
|---|---|
| Search returned nothing | `No wallets found.` |
| `Favorites` rail row, user has zero favorites at all | `No favorites yet. Star a wallet to add it.` |
| `Favorites` rail row, favorites exist but chain filter zeros them | `No favorites on this chain.` |

The third case is From-Wallet-specific — it exists because `filterChain` is a real external filter here.

---

## 5. Search matches multiple fields

Unlike Transfer Type (where search matches only the type label), the From Wallet search matches against a composite of:

- Wallet name
- Chain label (human-readable, e.g. "Solana")
- Full address
- Group
- Entity

So `0xabc`, `Solana`, `MIO`, or `Binance` all return useful hits. This is intentional — operators identify wallets by many different attributes depending on context.

---

## 6. Sort

- Alphabetical by wallet name within whichever scope is active.
- Starred wallets are pinned to the top of their scope.
- No "workspace order" primary sort like Transfer Type uses for reasons (workspaces don't have a meaningful linear order).

---

## 7. Tailwind deltas

Only the differences from the Transfer Type Tailwind reference are listed here. Anything not listed uses the same classes.

### Left rail wrapper

```
flex flex-col gap-0.5 w-52 shrink-0 border-r p-1.5
```

(`w-52` instead of `w-48` to fit "Maple Trading".)

### Item row (two-line)

Outer:

```
flex items-center justify-between w-full gap-2
```

Left column:

```
flex flex-col min-w-0
```

Top line (name + chain badge):

```
text-sm flex items-center gap-1.5
```

Bottom line (address):

```
text-xs text-muted-foreground
```

### Trigger (selected state, three pieces)

```
flex items-center gap-2 truncate min-w-0
```

with the address span using `text-muted-foreground text-xs truncate`.

### Star button

Uses a dedicated `<WalletStarButton>` component, but visually identical to the Transfer Type star — hidden by default, fades in on row hover and on keyboard focus, amber when starred.

---

## 8. Quick checklist for developers

Read the [Transfer Type checklist](./hand-off-popover.md#12-quick-checklist-for-developers) first. Then add:

- [ ] Rail is workspaces (`All` / `Favorites` / divider / `MIO` / `PPOA` / `Maple Trading`), 208 px wide.
- [ ] Item rows are two lines: name + chain badge on top, full address on bottom.
- [ ] No workspace badge inside the row (chain badge only).
- [ ] Trigger shows name + chain badge + truncated address.
- [ ] `filterChain` and `excludeAddress` apply before rail and search; counts shrink honestly.
- [ ] Empty states cover the three cases in §4, including "No favorites on this chain."
- [ ] Search matches name, chain label, address, group, and entity.
- [ ] Sort: alphabetical by name with favorites pinned first; no workspace-order primary sort.
- [ ] `Maple Trading` membership is driven by the entity bundle, not a single field.
