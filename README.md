# eRC-backroom-game-

> The Backrooms — Level 0, rebuilt as a headless onchain game.

A single-file, dependency-free WebGL game where the smart contract *is* the
backend. No game server, no build step, no asset downloads. Paste your deployed
contract addresses and it runs.

---

## What's here

| File | What it is |
|---|---|
| `index.html` | The entire game — engine, renderer, economy, wallet. 2,880 lines, zero external references. |
| `contract.md` | Full smart-contract specification: reference Solidity, ABI, events, invariants, solvency math, threat model, audit checklist. |

---

## Run it

Open `index.html` in a browser. That is the whole install.

For practice mode nothing else is needed. For onchain play, open the **CONTRACT**
box in the main menu and paste:

- the `BackroomCoin` (BACK) address
- the `BackroomVault` address
- the chain ID (1 for Ethereum mainnet)

Saved to `localStorage`. You can also edit the `CONFIG` object at the top of the
script directly.

---

## Token

| | |
|---|---|
| Name / Symbol | Backroom Coin / **BACK** |
| Total supply | `666,666,666,666` |
| Decimals | **10** |
| Max supply (raw) | `6,666,666,666,660,000,000,000` |

> **⚠️ The single most important integration constraint**
>
> `6.67e21` is roughly **740,000× larger than `Number.MAX_SAFE_INTEGER`**.
> Every raw amount crossing the contract boundary must be a `BigInt`.
> `parseFloat`, `Number(balance)`, or any formatter returning a JS `number`
> will silently corrupt balances with no error.
>
> The bundled game never routes a raw value through `Number`.

---

## How the game works

1. **Enter a run** — escrow the entry fee (default `500 BACK`) on-chain. The
   contract mints a `runId` and opens a settlement window.
2. **Play** — the whole building, generated procedurally from the run seed.
   No two runs are the same.
3. **Collect** — coins accrue in your wallet's run counter.
4. **Settle** — on death or extraction, call `settleRun(coins, died)`. The stake
   is slashed on death and returned on survival.
5. **Claim** — `claimAll()` moves everything settled into your wallet.

Practice mode runs the *same* code against a local ledger that enforces the
identical entry fee, rate ceiling, daily cap and run limit. It is a rehearsal,
not a faucet.

---

## Controls

| | |
|---|---|
| `W` `A` `S` `D` | move |
| `shift` | sprint |
| mouse | look (click to capture the pointer) |
| `E` | extract / settle |
| `B` | buy BACK |
| `C` | claim |
| `R` | die |
| `H` | summon a horde |
| `M` | sound |
| `esc` | pause |

On touch devices: left stick moves, drag on the right to look, plus an on-screen
pad.

---

## Honesty about the model

This is a **client-authoritative** game. The browser computes the run and reports
how many coins it collected; the contract cannot see what happened inside it.
Anyone claiming otherwise about a headless realtime game is describing a server
they have not mentioned.

The mitigation is **economic and bounded, not absolute**: a wall-clock time lock,
a per-second rate ceiling, a hard per-run cap, a UTC daily cap, a per-wallet run
limit, a cooldown, and a slashable stake. Faking the maximum costs real time and
real BACK. A determined multi-wallet attacker can still beat the caps — the cost
of doing so is funding every wallet with the entry fee.

See `contract.md` §12 for the full threat analysis.

### The reserve is finite

Payouts come from a **pre-funded emission reserve** (default 3% of supply), not
from entry fees. It only ever decreases.

| Active wallets hitting the daily cap | Reserve exhausted in |
|---|---|
| 100 | ~2.2 years |
| 500 | ~5.2 months |
| 1,000 | ~2.6 months |
| 10,000 | **~8 days** |

This is arithmetic, not risk. Size the reserve with the formula in
`contract.md` §4.4 before you launch, and watch the `reserveFloor` governor.

---

## Deployment status

- **`contract.md`** — specification and reference implementation. Complete and
  compilable, **unaudited**.
- **`index.html`** — complete and tested against a local ledger.
- **Contracts** — not yet deployed. Paste addresses when you have them.

Before mainnet, work through the audit checklist in `contract.md` §16. The top
item is a timelock on the admin functions; that removes most of the centralization
risk in one change.

---

## Licence

MIT.
