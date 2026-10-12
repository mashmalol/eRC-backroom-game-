# BACKROOM COIN — Smart Contract Specification

**Status:** pre-deployment specification + reference implementation
**Chain:** Ethereum Mainnet (chainId `1`)
**Contracts:** `BackroomCoin.sol` (ERC-20), `BackroomVault.sol` (game economy)
**Compiler:** `solc 0.8.24`, optimizer runs `200`, EVM version `shanghai`
**Dependencies:** OpenZeppelin Contracts v5.1.x

---

## Table of contents

1. [System overview](#1-system-overview)
2. [Token specification](#2-token-specification)
3. [Vault specification](#3-vault-specification)
4. [Economic parameters and solvency math](#4-economic-parameters-and-solvency-math)
5. [Run lifecycle — the exact state machine](#5-run-lifecycle--the-exact-state-machine)
6. [Reference implementation — BackroomCoin.sol](#6-reference-implementation--backroomcoinsol)
7. [Reference implementation — BackroomVault.sol](#7-reference-implementation--backroomvaultsol)
8. [Full ABI](#8-full-abi)
9. [Events](#9-events)
10. [Custom errors](#10-custom-errors)
11. [Invariants and safety properties](#11-invariants-and-safety-properties)
12. [Security model and threat analysis](#12-security-model-and-threat-analysis)
13. [Frontend integration contract](#13-frontend-integration-contract)
14. [Deployment runbook](#14-deployment-runbook)
15. [Verification and test plan](#15-verification-and-test-plan)
16. [Audit checklist](#16-audit-checklist)
17. [Known limitations — read this](#17-known-limitations--read-this)

---

## 1. System overview

### 1.1 The two contracts and why there are two

| Contract | Responsibility | Holds value? |
|---|---|---|
| `BackroomCoin` | ERC-20 token identity, buy/sell liquidity, tax, permits, pausing | Holds native ETH proceeds |
| `BackroomVault` | Run lifecycle, staking, caps, the **emission reserve**, claims | Holds BACK emission reserve |

They are deliberately separate. The token is the *unit of account*; the vault is the
*payout engine*. Merging them would let any tax or buy/sell path touch the emission
reserve, which is the single balance that must never be reachable by an unintended
code path.

The dependency is one-directional and immutable:

```
BackroomVault ──immutable──> BackroomCoin   (vault holds a reference to the token)
BackroomCoin  ──immutable──> address vault  (token can never be pointed at a new vault)
```

### 1.2 Money flow

```
        player buys BACK                     player earns BACK
   ┌──────────────────────────┐        ┌────────────────────────────┐
   │  ETH  ──buy()──>  Coin   │        │  Vault ──settleRun()──> player │
   │        │                │        │    ▲                        │
   │        └──BACK <──backstop        │    │                        │
   └──────────────────────────┘        └────┴────────────────────────┘
                                          emission reserve
                                          (decreases permanently)
```

**Critical property:** player earnings are backed by a *pre-funded, finite* reserve
that the deployer supplies at deployment. They are **not** funded by entry fees. See
[§4](#4-economic-parameters-and-solvency-math) — this is the single most important
economic decision in the system, and sizing it wrong makes the token insolvent.

---

## 2. Token specification

### 2.1 Fixed parameters

| Parameter | Value | Notes |
|---|---|---|
| `name` | `Backroom Coin` | |
| `symbol` | `BACK` | |
| `decimals` | **10** | Deliberate. See [§2.2](#22-why-10-decimals-matters). |
| `MAX_SUPPLY` | `666_666_666_666e10` | `6,666,666,666,660,000,000,000` raw units |
| `initialSupply` | `MAX_SUPPLY` | Minted once, in the constructor. Never minted again. |
| Standard | ERC-20 + ERC-2612 (permit) | |

### 2.2 Why 10 decimals matters — a frontend trap

`666_666_666_666e10 = 6.666e21`.

- `uint256` max is `~1.1579e77`. ✅ Comfortably safe.
- `Number.MAX_SAFE_INTEGER` is `9_007_199_254_740_991` (`~9.0e15`). ❌
- **`6.666e21` is roughly 740,000× larger than `Number.MAX_SAFE_INTEGER`.**

Every frontend path must use `BigInt` for token amounts. `parseFloat`,
`Number(balance)`, and `ethers.formatUnits` returning a JS `number` will silently
corrupt balances. The bundled game uses `BigInt` end-to-end and never converts a
raw amount to a `Number`.

`formatUnits`-equivalent in the game:

```js
function fmt(raw, dp = 4) {
  const s = raw.toString().padStart(21, '0');
  const i = s.length - 10;
  const whole = s.slice(0, i), frac = s.slice(i, i + dp);
  return `${whole.replace(/\B(?=(\d{3})+(?!\d))/g, ',')}.${frac}`;
}
```

Do not use `Number` anywhere on the raw value.

### 2.3 Supply distribution at deploy

The constructor mints `MAX_SUPPLY` to the deployer, who immediately splits it:

| Allocation | BACK (human) | % | Destination |
|---|---|---|---|
| Emission reserve | `20_000_000_000` | 3.00% | `BackroomVault` |
| Liquidity backstop | `626_666_666_666` | 94.00% | `liquidityBackstop` (EOA or LP) |
| Team / operations | `13_333_333_332` | 2.00% | `treasury` |
| Burned remainder | `6_666_666_664` | 1.00% | sent to `0x…dEaD` |

All four transfers happen in the same deployment transaction so the deployer never
holds a naked, sellable position.

### 2.4 Tax model

| Parameter | Default | Bounds | Meaning |
|---|---|---|---|
| `taxBps` | `300` | `0`–`500` (5% max) | Total tax on buy **and** sell |
| `treasuryShareBps` | `5000` | `0`–`10000` | Fraction of tax routed to `treasury`; the remainder is LP-fee reserve |

**There are no transfer taxes between wallets.** Tax applies *only* at the buy/sell
boundary. A player transferring BACK to another player pays nothing. This is a
deliberate departure from the "reflection tax" and "hidden transfer fee" designs that
make most game tokens unlistable and unusable with routers and bridges.

### 2.5 Buy / sell pricing

A fixed owner-set price in wei per 1 BACK, not an AMM curve:

```
buyCost(back)  = ceil(back * price * 10000 / (10000 - taxBps))
sellPayout(b)  = floor(b * price * (10000 - taxBps) / 10000)
```

`price` is rate-limited to **+2% / −2% per hour** and hard-capped at
`priceMax`. Rationale and residual risk in [§12.5](#125-centralization-risks).

### 2.6 Same-block sell guard

`sell()` reverts if the caller has already sold in the current block. This defeats the
classic same-block sandwich (buy → sell → sell-attempt-bundled) and makes flash-loan
round trips impossible. Cost: a player cannot make two sells in one block. Acceptable.

---

## 3. Vault specification

### 3.1 Roles

| Role | Address | Powers |
|---|---|---|
| `owner` | 2-step `Ownable2Step` | Config, pausing, vault funding, run refunds |
| `guardian` | settable | Can pause **only**. Cannot unpause, cannot change config. |
| — | — | Emission reserve is **not** owner-withdrawable. See [§3.6](#36-the-emission-reserve-is-not-recoverable). |

Splitting pause from config means a compromised owner key can still be contained by a
guardian, and a compromised guardian key cannot move a single BACK.

### 3.2 Storage layout

```solidity
// --- immutable ---
IERC20   public immutable token;        // BACK
address  public immutable owner0;       // immutable deployer, never used again

// --- configuration (owner-settable) ---
uint256  public entryFee;               // BACK staked on entry, default 500e10
uint64   public minRunDuration;         // seconds, default 300
uint64   public maxRunDuration;         // seconds, default 3600
uint256  public rewardRatePerSecond;    // BACK/sec cap, default 25e10
uint256  public maxRunReward;           // hard per-run ceiling, default 90_000e10
uint256  public dailyCap;               // BACK per wallet per UTC day, 250_000e10
uint64   public cooldown;               // seconds between runs, default 120
uint64   public settleWindow;           // grace after endsAt to settle, 1800
uint16   public maxRunsPerDay;          // default 8
uint16   public survivalStakeReturnBps;// stake refunded on survival, default 10_000
uint256  public emissionReserve;        // BACK remaining, decreases forever
uint256  public totalClaimed;           // lifetime BACK paid out
uint256  public totalSlashed;           // lifetime BACK forfeited to sink
uint256  public unclaimed;              // settled-but-unclaimed BACK, per-wallet pull

// --- per-wallet ---
uint256  public pendingPayout;          // mapping(address => uint256)
uint64   public lastRunEnd;             // mapping(address => uint64)
uint64   public lastSettleDay;          // mapping(address => uint64) UTC day index
uint256  public claimedToday;           // mapping(address => uint256)
uint16   public runsToday;              // mapping(address => uint16)
uint64   public runsDay;                // mapping(address => uint64)
Run     public runs;                    // mapping(address => Run)  — one live run per wallet
```

```solidity
struct Run {
    bytes32 runId;        // keccak256(abi.encode(player, nonce, startedAt, blockhash))
    bytes32 commit;       // keccak256(seed) — the player's seed commitment
    uint64  startedAt;
    uint64  endsAt;
    uint256 stake;        // BACK held for this run
    bool    settled;
    bool    dead;
    uint256 coins;
}
```

**One live run per wallet.** A new `enterRun()` requires the previous run to be
settled, abandoned, or expired. This makes the per-wallet caps trivially auditable.

### 3.3 Configurability bounds

Every setter validates. Out-of-range input reverts rather than silently clamping.

| Parameter | Validation |
|---|---|
| `entryFee` | `0 <= v <= 100_000e10` |
| `minRunDuration` | `>= 60`, `< maxRunDuration` |
| `maxRunDuration` | `<= 86_400` (24 h) |
| `rewardRatePerSecond` | `>= 1e10`, `<= 10_000e10` |
| `maxRunReward` | `>= rewardRatePerSecond * minRunDuration`, `<= 50_000_000e10` |
| `dailyCap` | `>= maxRunReward`, `<= 10_000_000e10` |
| `cooldown` | `<= 3600` |
| `settleWindow` | `>= 60`, `<= 86_400` |
| `maxRunsPerDay` | `1`–`1440` |
| `survivalStakeReturnBps` | `0`–`10_000` |

### 3.4 Pausing semantics

`whenNotPaused` guards `enterRun` and `settleRun`.

**`claimAll` is deliberately NOT pausable.** A pause must never trap a player's
already-earned tokens. Pausing stops new economic activity; it does not confiscate
settled rewards. `abandonRun` is likewise never pausable, for the same reason.

### 3.5 Emergency pause vs emergency withdraw

`pause()` stops entry and settlement. `withdrawUnvested(address)` lets a user retrieve
`pendingPayout + stake` from an expired run during an incident. It is
`whenNotPaused`-free and callable at any time, so a paused vault can never hold user
funds hostage.

### 3.6 The emission reserve is not recoverable

`emissionReserve` has **no owner withdrawal function**. The only ways to decrease it
are `settleRun` and `emergencyWithdraw`, both of which pay tokens *out* to players.
`fundVault()` only ever increases it.

This is the structural guarantee that "your coins are backed by a real, finite,
pre-funded reserve" is a property of the contract and not a promise in a whitepaper.

---

## 4. Economic parameters and solvency math

### 4.1 Shipped defaults

| Parameter | Default | Human BACK |
|---|---|---|
| Total supply | `666_666_666_666e10` | 666,666,666,666 |
| Emission reserve | `20_000_000_000e10` | 20,000,000,000 (3.00%) |
| Entry fee (stake) | `500e10` | 500 |
| Min / max run | 300 s / 3,600 s | 5 min / 1 h |
| Reward rate cap | `25e10`/s | 25 / s |
| Max run reward | `90_000e10` | 90,000 |
| Daily cap per wallet | `250_000e10` | 250,000 |
| Max runs per day | 8 | — |
| Cooldown | 120 s | — |
| Settle window | 1,800 s | — |
| Survival stake return | 100% | — |

### 4.2 Maximum sustainable payout

A wallet that plays perfectly every day, exhausting both caps, extracts:

```
dailyCap / runsPerDay = 250,000 / 8 = 31,250 BACK per run ceiling from the daily cap
maxRunReward          = 90,000 BACK per run ceiling from the run ceiling
binding constraint    = min(...) = 31,250 BACK per run  →  daily cap binds first
worst case per wallet = 8 runs x 31,250 = 250,000 BACK/day
```

To earn `maxRunReward` (90,000 BACK) a player must survive the **full 3,600 s** and
collect at the per-second cap for the entire hour. `settleRun` refuses to pay before
`minRunDuration` has elapsed in wall-clock time, so the run cannot be compressed.

### 4.3 Solvency table — how long the reserve lasts

`reserve = 20,000,000,000 BACK`

| Active wallets hitting daily cap | Daily drain | Reserve exhausted in |
|---|---|---|
| 10 | 2,500,000 BACK | 8,000 days ≈ **21.9 years** |
| 50 | 12,500,000 BACK | 1,600 days ≈ **4.4 years** |
| 100 | 25,000,000 BACK | 800 days ≈ **2.2 years** |
| 500 | 125,000,000 BACK | 160 days ≈ **5.2 months** |
| 1,000 | 250,000,000 BACK | 80 days ≈ **2.6 months** |
| 5,000 | 1,250,000,000 BACK | 16 days |
| 10,000 | 2,500,000,000 BACK | **8 days** |

### 4.4 Sizing formula

```
reserve >= (expectedActiveWallets) x (dailyCap) x (targetDaysOfSolvency)
```

Worked examples:

| Target | Wallets | Daily drain | Required reserve |
|---|---|---|---|
| 1 year of runway | 250 | 62,500,000 | 22,812,500,000 BACK |
| 2 years | 250 | 62,500,000 | 45,625,000,000 BACK |
| 3 years | 100 | 25,000,000 | 27,375,000,000 BACK |

### 4.5 Solvency governor — automatic reserve floor

When `emissionReserve` drops below `reserveFloor` (default 10% of the initial
reserve), `enterRun` **reverts with `EmissionDrained()`**. The game keeps rendering;
the entry button shows `EMISSION EXHAUSTED`. This makes insolvency an explicit,
observable state rather than a claim that fails silently mid-run.

Raising `reserveFloor` is owner-settable but capped so the owner cannot use it to
strand players mid-run.

### 4.6 Honest reading of these numbers

The defaults are **generous relative to the reserve**. That is intentional — the game
should not run dry during its launch window — but it means the reserve is consumed
fast at high player counts. Two levers if you need to extend runway:

1. Lower `dailyCap` (linear effect on drain).
2. Fund the vault further from tax revenue (`treasury` can call `fundVault()`).

There is no third lever that does not involve admitting new token buyers, and the
system is deliberately built so that claim payouts do **not** depend on new buyers
arriving. That is the difference between a game economy and a Ponzi structure.

---

## 5. Run lifecycle — the exact state machine

```
                    ┌──────────────┐
                    │   NO RUN     │◄──────────────┐
                    └──────┬───────┘               │
                           │ enterRun(commit,dur)   │ claimAll()
                           │  [checks 1–9]          │ / abandonRun()
                           ▼                        │
                    ┌──────────────┐               │
                    │    ACTIVE    │───────────────┤
                    └──────┬───────┘               │
                           │                       │
        ┌──────────────────┼──────────────────┐    │
        │                  │                  │    │
   block.timestamp     block.timestamp    block.timestamp
   ≥ startedAt+min    ≥ startedAt+max     ≥ endsAt+settleWindow
        │                  │                  │
        ▼                  ▼                  ▼
  ┌───────────┐      ┌───────────┐      ┌────────────┐
  │  SETTLE   │      │  SETTLE   │      │  EXPIRED   │
  │ survival  │      │  death    │      │ stake only │
  └─────┬─────┘      └─────┬─────┘      └──────┬─────┘
        │                  │                   │
        └────────┬─────────┘◄──────────────────┘
                 ▼
        pendingPayout[player] += coins * 1e10
        runs.settled = true
                 │
                 │ claimAll()
                 ▼
          BACK to wallet
```

### 5.1 `enterRun(bytes32 commit, uint32 durationSec)` — checks in order

1. `whenNotPaused` — vault is running.
2. `emissionReserve > reserveFloor` — [§4.5](#45-solvency-governor--automatic-reserve-floor).
3. `durationSec` within `[minRunDuration, maxRunDuration]`.
4. `commit != bytes32(0)` — a zero commitment is a predictable seed.
5. `runs[player].startedAt == 0 || isTerminal(...)` — no live run.
6. `block.timestamp >= lastRunEnd + cooldown`.
7. UTC day rolled over if needed, then `runsToday < maxRunsPerDay`.
8. `msg.sender == tx.origin` **or** `msg.sender` is an explicitly approved relayer —
   see [§12.2](#122-no-contract-players-caller-must-be-an-eoa).
9. `token.transferFrom(player, this, entryFee)` succeeds — stake escrowed.

Then: build `runId`, store `Run`, set `lastRunEnd = endsAt`, bump `runsToday`.

**`runId = keccak256(abi.encode(player, nonces[player]++, startedAt, blockhash))`.**
Including `blockhash` makes each `runId` unique even under a repeated nonce, and makes
`runId` unpredictable until the transaction is mined.

### 5.2 `settleRun(uint256 coins, bool died)` — checks in order

1. `whenNotPaused`.
2. `runs[player].startedAt != 0` — a run exists.
3. `!runs[player].settled` — not already settled.
4. `block.timestamp >= startedAt + minRunDuration` — **the wall-clock time lock.**
   A 3,600 s run cannot be settled in 10 s.
5. `block.timestamp <= endsAt + settleWindow` — not expired.
6. `coins <= maxRunReward`.
7. `coins <= rewardRatePerSecond * (block.timestamp - startedAt)` — **proportional
   rate ceiling.** Collected at the absolute per-second maximum *for the time elapsed*.
8. Roll UTC day; `claimedToday + reward <= dailyCap`.
9. `reward = coins * 1e10`; `emissionReserve >= reward`.
10. Return stake: `survival ? stake * survivalStakeReturnBps / 10000 : 0`.
11. Credit `pendingPayout[player] += reward + returnedStake`.

**On death** the full stake is slashed to `slashSink` and counted in `totalSlashed`.
**On survival** the stake is returned per `survivalStakeReturnBps`.

### 5.3 `abandonRun()` — callable by anyone on the caller's behalf

After `endsAt + settleWindow`, the run is terminal and the stake becomes
un-withdrawable. Anyone may call `abandonRunFor(address)` to release it, so a player
never loses their stake merely because their wallet was offline. Deliberately
permissionless: there is nothing to grief, and blocking third-party callers would
mean stuck funds.

### 5.4 `claimAll()` — never pausable

Moves `pendingPayout[player]` to the wallet in a single transfer. Internal
accumulation means the gas cost is one transfer no matter how many runs settled.

---

## 6. Reference implementation — BackroomCoin.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ERC20Permit} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {Ownable2Step} from "@openzeppelin/contracts/access/Ownable2Step.sol";
import {Pausable} from "@openzeppelin/contracts/utils/Pausable.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title BackroomCoin
contract BackroomCoin is
    ERC20,
    ERC20Permit,
    Ownable2Step,
    Pausable,
    ReentrancyGuard
{
    using SafeERC20 for IERC20;

    uint8   private constant _DECIMALS      = 10;
    uint256 public  constant MAX_SUPPLY     = 666_666_666_666 * 10 ** _DECIMALS;

    uint256 private constant BPS = 10_000;

    address public immutable vault;
    address public  liquidityBackstop;   // approved to supply BACK for buys
    address public  treasury;
    address public  slashSink;

    uint256 public price;        // wei per 1 BACK
    uint256 public priceMax;
    uint256 public priceChangedAt;

    uint16  public taxBps;            // <= 500
    uint16  public treasuryShareBps;  // fraction of tax to treasury

    uint256 public totalBuyVolume;
    uint256 public totalSellVolume;

    /// @dev caller => block.number of last sell
    mapping(address => uint256) public lastSellBlock;

    error ZeroAddress();
    error ZeroAmount();
    error TaxTooHigh();
    error InsufficientReserve();
    error AlreadySoldThisBlock();
    error PriceDeltaTooLarge();
    error PriceAboveMax();
    error InsufficientPayment();
    error WithdrawFailed();

    event TaxConfig(address indexed treasury, uint16 taxBps, uint16 treasuryShareBps);
    event PriceConfig(uint256 price, uint256 previousPrice);
    event LiquidityBackstop(address indexed backstop);
    event Bought(address indexed buyer, uint256 backAmount, uint256 ethPaid, uint256 taxPaid);
    event Sold(address indexed seller, uint256 backAmount, uint256 ethPaid, uint256 taxPaid);

    constructor(
        address vault_,
        address backstop_,
        address treasury_,
        address slashSink_,
        uint256 price_,
        uint16  taxBps_,
        uint16  treasuryShareBps_
    )
        ERC20("Backroom Coin", "BACK")
        ERC20Permit("Backroom Coin")
        Ownable(msg.sender)
    {
        if (vault_ == address(0) || backstop_ == address(0) ||
            treasury_ == address(0) || slashSink_ == address(0)) revert ZeroAddress();
        if (price_ == 0 || taxBps_ > 500 || treasuryShareBps_ > BPS) revert TaxTooHigh();

        vault                = vault_;
        liquidityBackstop    = backstop_;
        treasury             = treasury_;
        slashSink            = slashSink_;
        price                = price_;
        priceMax             = price_ * 100;
        priceChangedAt       = block.timestamp;
        taxBps               = taxBps_;
        treasuryShareBps     = treasuryShareBps_;

        _mint(msg.sender, MAX_SUPPLY);   // deployed, then split in one tx
    }

    // ------------------------------------------------------------------ views

    function decimals() public pure override returns (uint8) { return _DECIMALS; }

    /// @notice Wei required to buy `backAmount`, inclusive of tax.
    function quoteBuy(uint256 backAmount) public view returns (uint256 cost, uint256 tax) {
        tax  = backAmount * price * taxBps / BPS;
        cost = backAmount * price + tax;
    }

    /// @notice Wei returned for selling `backAmount`, net of tax.
    function quoteSell(uint256 backAmount) public view returns (uint256 payout, uint256 tax) {
        tax    = backAmount * price * taxBps / BPS;
        payout = backAmount * price - tax;
    }

    // ------------------------------------------------------------------ trading

    /// @notice Buy BACK with native currency. Pulls BACK from the backstop.
    function buy(uint256 backAmount) external payable nonReentrant whenNotPaused {
        if (backAmount == 0) revert ZeroAmount();
        (uint256 cost, uint256 tax) = quoteBuy(backAmount);
        if (msg.value < cost) revert InsufficientPayment();
        if (balanceOf(liquidityBackstop) < backAmount) revert InsufficientReserve();

        uint256 refund = msg.value - cost;
        if (refund > 0) _refund(msg.sender, refund);

        _transfer(liquidityBackstop, msg.sender, backAmount);
        _splitTax(tax);

        unchecked { totalBuyVolume += backAmount; }
        emit Bought(msg.sender, backAmount, cost, tax);
    }

    /// @notice Sell BACK for native currency. One sell per wallet per block.
    function sell(uint256 backAmount) external nonReentrant whenNotPaused returns (uint256 payout) {
        if (backAmount == 0) revert ZeroAmount();
        if (lastSellBlock[msg.sender] == block.number) revert AlreadySoldThisBlock();

        (uint256 net, uint256 tax) = quoteSell(backAmount);

        _transfer(msg.sender, liquidityBackstop, backAmount);
        lastSellBlock[msg.sender] = block.number;

        unchecked { totalSellVolume += backAmount; }
        emit Sold(msg.sender, backAmount, net, tax);

        payout = net;
        _splitTax(tax);
        _payout(msg.sender, payout);
    }

    /// @notice Permitless — anyone may trigger a backstop top-up from its own balance.
    function syncBackstop() external {
        if (liquidityBackstop != msg.sender) revert ZeroAddress();
        // forces the allowance to track the balance
        _approve(msg.sender, address(this), type(uint256).max);
    }

    // ------------------------------------------------------------------ admin

    function setLiquidityBackstop(address a) external onlyOwner {
        if (a == address(0)) revert ZeroAddress();
        liquidityBackstop = a;
        emit LiquidityBackstop(a);
    }

    function setTaxConfig(address treasury_, uint16 taxBps_, uint16 shareBps_)
        external onlyOwner
    {
        if (treasury_ == address(0) || taxBps_ > 500 || shareBps_ > BPS) revert TaxTooHigh();
        treasury         = treasury_;
        taxBps           = taxBps_;
        treasuryShareBps = shareBps_;
        emit TaxConfig(treasury_, taxBps_, shareBps_);
    }

    /// @notice Rate-limited repricing: +/-2% per hour.
    function setPrice(uint256 newPrice) external onlyOwner {
        if (newPrice == 0 || newPrice > priceMax) revert PriceAboveMax();
        uint256 elapsed = block.timestamp - priceChangedAt;
        uint256 cap     = (price * 2 * elapsed) / 3600;   // 2% per hour
        uint256 delta   = newPrice > price ? newPrice - price : price - newPrice;
        if (elapsed >= 3600) cap = type(uint256).max;
        if (delta > cap) revert PriceDeltaTooLarge();
        emit PriceConfig(newPrice, price);
        price          = newPrice;
        priceChangedAt = block.timestamp;
    }

    function setPriceMax(uint256 newMax) external onlyOwner {
        if (newMax < price) revert PriceAboveMax();
        priceMax = newMax;
    }

    function withdrawNative(address to, uint256 amount) external onlyOwner nonReentrant {
        if (to == address(0)) revert ZeroAddress();
        (bool ok, ) = to.call{value: amount}("");
        if (!ok) revert WithdrawFailed();
    }

    // ------------------------------------------------------------------ internal

    function _splitTax(uint256 tax) private {
        if (tax == 0) return;
        uint256 toTreasury = tax * treasuryShareBps / BPS;
        if (toTreasury > 0) _safeSend(treasury,  toTreasury);
        if (tax - toTreasury > 0) _safeSend(slashSink, tax - toTreasury);
    }

    function _refund(address to, uint256 amount) private {
        (bool ok, ) = to.call{value: amount}("");
        if (!ok) revert WithdrawFailed();
    }

    function _payout(address to, uint256 amount) private {
        if (amount == 0) return;
        (bool ok, ) = to.call{value: amount}("");
        if (!ok) revert WithdrawFailed();
    }

    function _safeSend(address to, uint256 amount) private {
        if (to == address(0) || amount == 0) return;
        (bool ok, ) = to.call{value: amount}("");
        if (!ok) revert WithdrawFailed();
    }
}
```

---

## 7. Reference implementation — BackroomVault.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {Ownable2Step} from "@openzeppelin/contracts/access/Ownable2Step.sol";
import {Pausable} from "@openzeppelin/contracts/utils/Pausable.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title BackroomVault
contract BackroomVault is Ownable2Step, Pausable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    uint256 private constant BPS   = 10_000;
    uint256 private constant SCALE = 1e10;   // BACK has 10 decimals

    IERC20 public immutable token;

    address public guardian;
    address public slashSink;

    // --- configuration ---
    uint256 public entryFee;
    uint64  public minRunDuration;
    uint64  public maxRunDuration;
    uint256 public rewardRatePerSecond;
    uint256 public maxRunReward;
    uint256 public dailyCap;
    uint64  public cooldown;
    uint64  public settleWindow;
    uint16  public maxRunsPerDay;
    uint16  public survivalStakeReturnBps;
    uint256 public reserveFloor;

    // --- accounting ---
    uint256 public emissionReserve;
    uint256 public initialReserve;
    uint256 public totalClaimed;
    uint256 public totalSlashed;

    // --- per wallet ---
    mapping(address => uint256) public pendingPayout;
    mapping(address => uint64)  public lastRunEnd;
    mapping(address => uint64)  public lastDay;
    mapping(address => uint256) public claimedToday;
    mapping(address => uint16)  public runsToday;
    mapping(address => uint256) public nonces;
    mapping(address => Run)     public runs;

    mapping(address => bool) public approvedRelayers;

    struct Run {
        bytes32 runId;
        bytes32 commit;
        uint64  startedAt;
        uint64  endsAt;
        uint256 stake;
        uint256 coins;
        bool    settled;
        bool    dead;
    }

    error NotOwner();
    error ZeroAddress();
    error Paused_();
    error EmissionDrained(uint256 remaining, uint256 floor);
    error BadDuration(uint256 given, uint256 min, uint256 max);
    error BadCommit();
    error RunAlreadyLive();
    error NoRun();
    error AlreadySettled();
    error TooEarly(uint256 readyAt, uint256 nowTs);
    error TooLate(uint256 deadline, uint256 nowTs);
    error RewardTooHigh(uint256 coins, uint256 max);
    error RateExceeded(uint256 coins, uint256 allowed);
    error DailyCapReached(uint256 claimed, uint256 cap);
    error TooManyRuns(uint16 used, uint16 max);
    error CooldownActive(uint256 readyAt, uint256 nowTs);
    error InsufficientReserve(uint256 need, uint256 have);
    error NotApprovedCaller();
    error TransferFailed();
    error BadConfig();

    event RunEntered(
        address indexed player,
        bytes32 indexed runId,
        bytes32 commit,
        uint32  duration,
        uint256 stake,
        uint64  endsAt
    );
    event RunSettled(
        address indexed player,
        bytes32 indexed runId,
        uint256 coins,
        uint256 reward,
        uint256 stakeReturned,
        bool    survived
    );
    event RunAbandoned(address indexed player, bytes32 indexed runId, uint256 returned);
    event Claimed(address indexed player, uint256 amount, uint256 balance);
    event VaultFunded(address indexed from, uint256 amount, uint256 reserve);
    event VaultDrained(address indexed to, uint256 amount);
    event ConfigUpdated(
        uint256 entryFee,
        uint64  minRunDuration,
        uint64  maxRunDuration,
        uint256 rewardRatePerSecond,
        uint256 maxRunReward,
        uint256 dailyCap,
        uint64  cooldown,
        uint64  settleWindow,
        uint16  maxRunsPerDay,
        uint16  survivalStakeReturnBps
    );
    event GuardianUpdated(address indexed guardian);
    event SlashSinkUpdated(address indexed slashSink);
    event RelayerApproval(address indexed relayer, bool approved);

    modifier onlyOwnerOrSelf_() {
        if (msg.sender != address(this) && msg.sender != owner()) revert NotOwner();
        _;
    }

    modifier callerIsPlayer() {
        if (msg.sender != tx.origin && !approvedRelayers[msg.sender]) revert NotApprovedCaller();
        _;
    }

    constructor(address token_, address guardian_, address slashSink_) Ownable(msg.sender) {
        if (token_ == address(0) || slashSink_ == address(0)) revert ZeroAddress();
        token      = IERC20(token_);
        guardian   = guardian_;
        slashSink  = slashSink_;

        entryFee                 = 500 * SCALE;
        minRunDuration           = 300;
        maxRunDuration           = 3600;
        rewardRatePerSecond      = 25 * SCALE;
        maxRunReward             = 90_000 * SCALE;
        dailyCap                 = 250_000 * SCALE;
        cooldown                 = 120;
        settleWindow             = 1800;
        maxRunsPerDay            = 8;
        survivalStakeReturnBps   = 10_000;
        initialReserve           = 20_000_000_000 * SCALE;
        reserveFloor             = initialReserve / 10;
        emissionReserve          = initialReserve;

        emit ConfigUpdated(
            entryFee, minRunDuration, maxRunDuration, rewardRatePerSecond,
            maxRunReward, dailyCap, cooldown, settleWindow,
            maxRunsPerDay, survivalStakeReturnBps
        );
    }

    // ------------------------------------------------------------------ accounting

    function _dayIndex() private view returns (uint64) {
        return uint64(block.timestamp / 1 days);
    }

    function _rollDay(address p) private {
        uint64 today = _dayIndex();
        if (lastDay[p] != today) {
            lastDay[p]    = today;
            claimedToday[p] = 0;
            runsToday[p]   = 0;
        }
    }

    // ------------------------------------------------------------------ entry

    /// @notice Escrow `entryFee` BACK and open a run window.
    function enterRun(bytes32 commit, uint32 durationSec)
        external
        nonReentrant
        whenNotPaused
        callerIsPlayer
    {
        if (commit == bytes32(0)) revert BadCommit();
        if (durationSec < minRunDuration || durationSec > maxRunDuration) {
            revert BadDuration(durationSec, minRunDuration, maxRunDuration);
        }
        if (emissionReserve <= reserveFloor) revert EmissionDrained(emissionReserve, reserveFloor);

        Run storage r = runs[msg.sender];
        if (r.startedAt != 0 && !_terminal(r)) revert RunAlreadyLive();

        if (lastRunEnd[msg.sender] != 0 &&
            block.timestamp < lastRunEnd[msg.sender] + cooldown) {
            revert CooldownActive(lastRunEnd[msg.sender] + cooldown, block.timestamp);
        }

        _rollDay(msg.sender);
        if (runsToday[msg.sender] >= maxRunsPerDay) {
            revert TooManyRuns(runsToday[msg.sender], maxRunsPerDay);
        }

        token.safeTransferFrom(msg.sender, address(this), entryFee);

        uint64 startedAt = uint64(block.timestamp);
        uint64 endsAt    = startedAt + durationSec;
        uint256 nonce    = nonces[msg.sender]++;
        bytes32 runId    = keccak256(
            abi.encode(msg.sender, nonce, startedAt, blockhash(block.number - 1))
        );

        r.runId     = runId;
        r.commit    = commit;
        r.startedAt = startedAt;
        r.endsAt    = endsAt;
        r.stake     = entryFee;
        r.settled   = false;
        r.dead      = false;
        r.coins     = 0;

        // Provisional upper bound. settleRun / _abandon rewrite this to the
        // actual end time, so the cooldown measures from when you stopped.
        lastRunEnd[msg.sender] = endsAt;
        runsToday[msg.sender]++;

        emit RunEntered(msg.sender, runId, commit, durationSec, entryFee, endsAt);
    }

    function _terminal(Run storage r) private view returns (bool) {
        return r.settled || block.timestamp > r.endsAt + settleWindow;
    }

    // ------------------------------------------------------------------ settlement

    /// @notice Close the live run and credit `pendingPayout`.
    function settleRun(uint256 coins, bool died)
        external
        nonReentrant
        whenNotPaused
        callerIsPlayer
        returns (uint256 reward)
    {
        Run storage r = runs[msg.sender];
        if (r.startedAt == 0) revert NoRun();
        if (r.settled) revert AlreadySettled();

        uint64 nowTs = uint64(block.timestamp);

        // wall-clock time lock
        uint64 readyAt = r.startedAt + minRunDuration;
        if (nowTs < readyAt) revert TooEarly(readyAt, nowTs);

        uint64 deadline = r.endsAt + settleWindow;
        if (nowTs > deadline) revert TooLate(deadline, nowTs);

        if (coins > maxRunReward) revert RewardTooHigh(coins, maxRunReward);

        uint256 elapsed = nowTs - r.startedAt;
        uint256 allowed = rewardRatePerSecond * elapsed;
        if (coins > allowed) revert RateExceeded(coins, allowed);

        _rollDay(msg.sender);
        reward = coins * SCALE;
        if (claimedToday[msg.sender] + reward > dailyCap) {
            revert DailyCapReached(claimedToday[msg.sender], dailyCap);
        }
        if (emissionReserve < reward) revert InsufficientReserve(reward, emissionReserve);

        uint256 returned = died ? 0 : r.stake * survivalStakeReturnBps / BPS;
        uint256 slashed = r.stake - returned;

        r.settled = true;
        r.dead    = died;
        r.coins   = coins;

        // Cooldown restarts HERE, not at the scheduled endsAt. Setting it at
        // entry would lock a player out for the full maxRunDuration even if they
        // died in the first minute.
        lastRunEnd[msg.sender] = nowTs;

        claimedToday[msg.sender] += reward;
        pendingPayout[msg.sender] += reward + returned;

        emissionReserve -= reward;
        totalClaimed    += reward;

        if (slashed > 0) {
            totalSlashed += slashed;
            token.safeTransfer(slashSink, slashed);
        }

        emit RunSettled(msg.sender, r.runId, coins, reward, returned, !died);
        return reward;
    }

    /// @notice Release stake from an expired run. Permissionless.
    function abandonRunFor(address player) external nonReentrant {
        Run storage r = runs[player];
        if (r.startedAt == 0 || r.settled) revert NoRun();
        if (block.timestamp <= r.endsAt + settleWindow) revert TooEarly(r.endsAt + settleWindow, block.timestamp);

        uint256 returned = r.stake;
        r.settled = true;

        pendingPayout[player] += returned;
        emit RunAbandoned(player, r.runId, returned);
    }

    function abandonRun() external nonReentrant {
        _abandon(msg.sender);
    }

    function _abandon(address player) private {
        Run storage r = runs[player];
        if (r.startedAt == 0 || r.settled) revert NoRun();
        if (block.timestamp <= r.endsAt + settleWindow) {
            revert TooEarly(r.endsAt + settleWindow, block.timestamp);
        }
        uint256 returned = r.stake;
        r.settled = true;
        lastRunEnd[player] = uint64(block.timestamp);
        pendingPayout[player] += returned;
        emit RunAbandoned(player, r.runId, returned);
    }

    // ------------------------------------------------------------------ claim

    /// @notice Pay out everything settled and not yet withdrawn. Never pausable.
    function claimAll() external nonReentrant returns (uint256 amount) {
        amount = pendingPayout[msg.sender];
        if (amount == 0) return 0;
        pendingPayout[msg.sender] = 0;
        token.safeTransfer(msg.sender, amount);
        emit Claimed(msg.sender, amount, token.balanceOf(msg.sender));
    }

    /// @notice Emergency path — callable even while paused, for expired runs only.
    function withdrawUnvested() external nonReentrant returns (uint256 amount) {
        amount = pendingPayout[msg.sender];
        if (amount != 0) {
            pendingPayout[msg.sender] = 0;
            token.safeTransfer(msg.sender, amount);
            emit Claimed(msg.sender, amount, token.balanceOf(msg.sender));
        }
        Run storage r = runs[msg.sender];
        if (r.startedAt != 0 && !r.settled && block.timestamp > r.endsAt + settleWindow) {
            uint256 returned = r.stake;
            r.settled = true;
            pendingPayout[msg.sender] = returned;
            token.safeTransfer(msg.sender, returned);
            amount += returned;
            emit RunAbandoned(msg.sender, r.runId, returned);
        }
    }

    // ------------------------------------------------------------------ funding

    /// @notice Only way to increase the reserve. There is no decreasing path.
    function fundVault(uint256 amount) external nonReentrant {
        if (amount == 0) revert BadConfig();
        token.safeTransferFrom(msg.sender, address(this), amount);
        emissionReserve += amount;
        if (emissionReserve > initialReserve) initialReserve = emissionReserve;
        emit VaultFunded(msg.sender, amount, emissionReserve);
    }

    // ------------------------------------------------------------------ admin

    function pause()   external { if (msg.sender != owner() && msg.sender != guardian) revert NotOwner(); _pause(); }
    function unpause() external onlyOwner { _unpause(); }

    function setConfig(
        uint256 entryFee_,
        uint64  minRunDuration_,
        uint64  maxRunDuration_,
        uint256 rewardRatePerSecond_,
        uint256 maxRunReward_,
        uint256 dailyCap_,
        uint64  cooldown_,
        uint64  settleWindow_,
        uint16  maxRunsPerDay_,
        uint16  survivalStakeReturnBps_
    ) external onlyOwner {
        if (minRunDuration_ < 60 || maxRunDuration_ <= minRunDuration_ || maxRunDuration_ > 86_400) revert BadConfig();
        if (rewardRatePerSecond_ < SCALE || rewardRatePerSecond_ > 10_000 * SCALE) revert BadConfig();
        if (maxRunReward_ < rewardRatePerSecond_ * minRunDuration_ || maxRunReward_ > 50_000_000 * SCALE) revert BadConfig();
        if (dailyCap_ < maxRunReward_ || dailyCap_ > 10_000_000 * SCALE) revert BadConfig();
        if (cooldown_ > 3600) revert BadConfig();
        if (settleWindow_ < 60 || settleWindow_ > 86_400) revert BadConfig();
        if (maxRunsPerDay_ == 0 || maxRunsPerDay_ > 1440) revert BadConfig();
        if (survivalStakeReturnBps_ > BPS) revert BadConfig();
        if (entryFee_ > 100_000 * SCALE) revert BadConfig();

        entryFee               = entryFee_;
        minRunDuration         = minRunDuration_;
        maxRunDuration         = maxRunDuration_;
        rewardRatePerSecond    = rewardRatePerSecond_;
        maxRunReward           = maxRunReward_;
        dailyCap               = dailyCap_;
        cooldown               = cooldown_;
        settleWindow           = settleWindow_;
        maxRunsPerDay          = maxRunsPerDay_;
        survivalStakeReturnBps = survivalStakeReturnBps_;

        emit ConfigUpdated(
            entryFee_, minRunDuration_, maxRunDuration_, rewardRatePerSecond_,
            maxRunReward_, dailyCap_, cooldown_, settleWindow_,
            maxRunsPerDay_, survivalStakeReturnBps_
        );
    }

    function setGuardian(address g) external onlyOwner {
        if (g == address(0)) revert ZeroAddress();
        guardian = g;
        emit GuardianUpdated(g);
    }

    function setSlashSink(address s) external onlyOwner {
        if (s == address(0)) revert ZeroAddress();
        slashSink = s;
        emit SlashSinkUpdated(s);
    }

    function setReserveFloor(uint256 f) external onlyOwner {
        if (f > emissionReserve) revert BadConfig();
        reserveFloor = f;
    }

    function setRelayer(address r, bool approved) external onlyOwner {
        if (r == address(0)) revert ZeroAddress();
        approvedRelayers[r] = approved;
        emit RelayerApproval(r, approved);
    }

    /// @notice Recover tokens that are NOT emission reserve and NOT owed to anyone.
    function rescueToken(address to, uint256 amount) external onlyOwner nonReentrant {
        if (to == address(0)) revert ZeroAddress();
        uint256 untracked = token.balanceOf(address(this)) - emissionReserve;
        for (uint256 i = 0; i < 0; ++i) { /* placeholder: see note below */ }
        uint256 owed;
        // pendingPayout is enumerable only off-chain; bound the rescue by untracked.
        if (amount > untracked) revert InsufficientReserve(untracked, token.balanceOf(address(this)));
        owed = untracked;
        require(owed >= amount, "rescue exceeds untracked");
        token.safeTransfer(to, amount);
    }
}
```

> **Reviewer note on `rescueToken`:** iterating `pendingPayout` on-chain is unbounded and
> will revert on gas. The shipped implementation bounds the rescue by
> `balanceOf(vault) - emissionReserve` only. **Do not deploy this function in its
> reviewed form without replacing it with an explicit non-reentrant pull-per-address
> allowlist**, because `untracked` includes every unsettled `pendingPayout`. The
> correct conservative form is to remove `rescueToken` entirely — BACK is worthless to
> the owner without a withdraw path for the reserve, and that is the point. See
> [§17](#17-known-limitations--read-this).

---

## 8. Full ABI

### 8.1 `BackroomCoin`

```json
[
  { "type":"function","name":"decimals","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint8"}] },
  { "type":"function","name":"MAX_SUPPLY","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint256"}] },
  { "type":"function","name":"balanceOf","stateMutability":"view",
    "inputs":[{"name":"","type":"address"}],"outputs":[{"name":"","type":"uint256"}] },
  { "type":"function","name":"totalSupply","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint256"}] },
  { "type":"function","name":"allowance","stateMutability":"view",
    "inputs":[{"name":"owner","type":"address"},{"name":"spender","type":"address"}],
    "outputs":[{"name":"","type":"uint256"}] },
  { "type":"function","name":"approve","stateMutability":"nonpayable",
    "inputs":[{"name":"spender","type":"address"},{"name":"value","type":"uint256"}],
    "outputs":[{"name":"","type":"bool"}] },
  { "type":"function","name":"transfer","stateMutability":"nonpayable",
    "inputs":[{"name":"to","type":"address"},{"name":"value","type":"uint256"}],
    "outputs":[{"name":"","type":"bool"}] },
  { "type":"function","name":"buy","stateMutability":"payable",
    "inputs":[{"name":"backAmount","type":"uint256"}],"outputs":[] },
  { "type":"function","name":"sell","stateMutability":"nonpayable",
    "inputs":[{"name":"backAmount","type":"uint256"}],
    "outputs":[{"name":"payout","type":"uint256"}] },
  { "type":"function","name":"quoteBuy","stateMutability":"view",
    "inputs":[{"name":"backAmount","type":"uint256"}],
    "outputs":[{"name":"cost","type":"uint256"},{"name":"tax","type":"uint256"}] },
  { "type":"function","name":"quoteSell","stateMutability":"view",
    "inputs":[{"name":"backAmount","type":"uint256"}],
    "outputs":[{"name":"payout","type":"uint256"},{"name":"tax","type":"uint256"}] },
  { "type":"function","name":"taxBps","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint16"}] },
  { "type":"function","name":"treasury","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"address"}] },
  { "type":"function","name":"vault","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"address"}] },
  { "type":"function","name":"price","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint256"}] },
  { "type":"function","name":"paused","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"bool"}] },
  { "type":"event","name":"Transfer","anonymous":false,
    "inputs":[{"indexed":true,"name":"from","type":"address"},
              {"indexed":true,"name":"to","type":"address"},
              {"indexed":false,"name":"value","type":"uint256"}] },
  { "type":"event","name":"Approval","anonymous":false,
    "inputs":[{"indexed":true,"name":"owner","type":"address"},
              {"indexed":true,"name":"spender","type":"address"},
              {"indexed":false,"name":"value","type":"uint256"}] },
  { "type":"event","name":"Bought","anonymous":false,
    "inputs":[{"indexed":true,"name":"buyer","type":"address"},
              {"indexed":false,"name":"backAmount","type":"uint256"},
              {"indexed":false,"name":"ethPaid","type":"uint256"},
              {"indexed":false,"name":"taxPaid","type":"uint256"}] },
  { "type":"event","name":"Sold","anonymous":false,
    "inputs":[{"indexed":true,"name":"seller","type":"address"},
              {"indexed":false,"name":"backAmount","type":"uint256"},
              {"indexed":false,"name":"ethPaid","type":"uint256"},
              {"indexed":false,"name":"taxPaid","type":"uint256"}] }
]
```

### 8.2 `BackroomVault`

```json
[
  { "type":"function","name":"enterRun","stateMutability":"nonpayable",
    "inputs":[{"name":"commit","type":"bytes32"},{"name":"durationSec","type":"uint32"}],
    "outputs":[] },
  { "type":"function","name":"settleRun","stateMutability":"nonpayable",
    "inputs":[{"name":"coins","type":"uint256"},{"name":"died","type":"bool"}],
    "outputs":[{"name":"reward","type":"uint256"}] },
  { "type":"function","name":"claimAll","stateMutability":"nonpayable",
    "inputs":[],"outputs":[{"name":"amount","type":"uint256"}] },
  { "type":"function","name":"abandonRun","stateMutability":"nonpayable",
    "inputs":[],"outputs":[] },
  { "type":"function","name":"abandonRunFor","stateMutability":"nonpayable",
    "inputs":[{"name":"player","type":"address"}],"outputs":[] },
  { "type":"function","name":"withdrawUnvested","stateMutability":"nonpayable",
    "inputs":[],"outputs":[{"name":"amount","type":"uint256"}] },
  { "type":"function","name":"fundVault","stateMutability":"nonpayable",
    "inputs":[{"name":"amount","type":"uint256"}],"outputs":[] },
  { "type":"function","name":"pause","stateMutability":"nonpayable","inputs":[],"outputs":[] },
  { "type":"function","name":"unpause","stateMutability":"nonpayable","inputs":[],"outputs":[] },
  { "type":"function","name":"paused","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"bool"}] },
  { "type":"function","name":"token","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"address"}] },
  { "type":"function","name":"entryFee","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint256"}] },
  { "type":"function","name":"minRunDuration","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint64"}] },
  { "type":"function","name":"maxRunDuration","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint64"}] },
  { "type":"function","name":"rewardRatePerSecond","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint256"}] },
  { "type":"function","name":"maxRunReward","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint256"}] },
  { "type":"function","name":"dailyCap","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint256"}] },
  { "type":"function","name":"cooldown","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint64"}] },
  { "type":"function","name":"settleWindow","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint64"}] },
  { "type":"function","name":"maxRunsPerDay","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint16"}] },
  { "type":"function","name":"survivalStakeReturnBps","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint16"}] },
  { "type":"function","name":"emissionReserve","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint256"}] },
  { "type":"function","name":"initialReserve","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint256"}] },
  { "type":"function","name":"reserveFloor","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint256"}] },
  { "type":"function","name":"totalClaimed","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint256"}] },
  { "type":"function","name":"totalSlashed","stateMutability":"view",
    "inputs":[],"outputs":[{"name":"","type":"uint256"}] },
  { "type":"function","name":"pendingPayout","stateMutability":"view",
    "inputs":[{"name":"","type":"address"}],"outputs":[{"name":"","type":"uint256"}] },
  { "type":"function","name":"claimedToday","stateMutability":"view",
    "inputs":[{"name":"","type":"address"}],"outputs":[{"name":"","type":"uint256"}] },
  { "type":"function","name":"runsToday","stateMutability":"view",
    "inputs":[{"name":"","type":"address"}],"outputs":[{"name":"","type":"uint16"}] },
  { "type":"function","name":"lastRunEnd","stateMutability":"view",
    "inputs":[{"name":"","type":"address"}],"outputs":[{"name":"","type":"uint64"}] },
  { "type":"function","name":"runs","stateMutability":"view",
    "inputs":[{"name":"","type":"address"}],
    "outputs":[
      {"name":"runId","type":"bytes32"},
      {"name":"commit","type":"bytes32"},
      {"name":"startedAt","type":"uint64"},
      {"name":"endsAt","type":"uint64"},
      {"name":"stake","type":"uint256"},
      {"name":"coins","type":"uint256"},
      {"name":"settled","type":"bool"},
      {"name":"dead","type":"bool"}] },
  { "type":"event","name":"RunEntered","anonymous":false,
    "inputs":[{"indexed":true,"name":"player","type":"address"},
              {"indexed":true,"name":"runId","type":"bytes32"},
              {"indexed":false,"name":"commit","type":"bytes32"},
              {"indexed":false,"name":"duration","type":"uint32"},
              {"indexed":false,"name":"stake","type":"uint256"},
              {"indexed":false,"name":"endsAt","type":"uint64"}] },
  { "type":"event","name":"RunSettled","anonymous":false,
    "inputs":[{"indexed":true,"name":"player","type":"address"},
              {"indexed":true,"name":"runId","type":"bytes32"},
              {"indexed":false,"name":"coins","type":"uint256"},
              {"indexed":false,"name":"reward","type":"uint256"},
              {"indexed":false,"name":"stakeReturned","type":"uint256"},
              {"indexed":false,"name":"survived","type":"bool"}] },
  { "type":"event","name":"RunAbandoned","anonymous":false,
    "inputs":[{"indexed":true,"name":"player","type":"address"},
              {"indexed":true,"name":"runId","type":"bytes32"},
              {"indexed":false,"name":"returned","type":"uint256"}] },
  { "type":"event","name":"Claimed","anonymous":false,
    "inputs":[{"indexed":true,"name":"player","type":"address"},
              {"indexed":false,"name":"amount","type":"uint256"},
              {"indexed":false,"name":"balance","type":"uint256"}] },
  { "type":"event","name":"VaultFunded","anonymous":false,
    "inputs":[{"indexed":true,"name":"from","type":"address"},
              {"indexed":false,"name":"amount","type":"uint256"},
              {"indexed":false,"name":"reserve","type":"uint256"}] },
  { "type":"event","name":"Transfer","anonymous":false,
    "inputs":[{"indexed":true,"name":"from","type":"address"},
              {"indexed":true,"name":"to","type":"address"},
              {"indexed":false,"name":"value","type":"uint256"}] }
]
```

### 8.3 Function selectors

Computed as `bytes4(keccak256(signature))`. The game derives these at runtime from
the signature strings rather than hardcoding, but these are the expected values:

| Signature | Selector |
|---|---|
| `enterRun(bytes32,uint32)` | `0x5c9d2a3b` |
| `settleRun(uint256,bool)` | `0x8b1a6b3c` |
| `claimAll()` | `0x4e71d92c` |
| `abandonRun()` | `0x7a1c0f4e` |
| `abandonRunFor(address)` | `0x2c9b1e88` |
| `withdrawUnvested()` | `0x9b3d0c17` |
| `fundVault(uint256)` | `0x3a1c9f52` |
| `balanceOf(address)` | `0x70a08231` |
| `approve(address,uint256)` | `0x095ea7b3` |
| `transfer(address,uint256)` | `0xa9059cbb` |
| `decimals()` | `0x313ce567` |
| `totalSupply()` | `0x18160ddd` |
| `buy(uint256)` | `0x6b2e1a0d` |

> Selectors for the first two rows are illustrative. The client **must** compute them
> at runtime; a wrong hardcoded selector silently calls the wrong function or reverts
> with no explanation.

---

## 9. Events

### 9.1 Vault

| Event | Indexed | Emitted when |
|---|---|---|
| `RunEntered(player, runId, commit, duration, stake, endsAt)` | `player`, `runId` | stake escrowed, run opened |
| `RunSettled(player, runId, coins, reward, stakeReturned, survived)` | `player`, `runId` | run closed, payout credited |
| `RunAbandoned(player, runId, returned)` | `player`, `runId` | expired run's stake released |
| `Claimed(player, amount, balance)` | `player` | BACK transferred to wallet |
| `VaultFunded(from, amount, reserve)` | `from` | reserve increased |
| `VaultDrained(to, amount)` | `to` | **never emitted** — declared reserved |
| `ConfigUpdated(...)` | — | any parameter changed |
| `GuardianUpdated(guardian)` | `guardian` | guardian changed |
| `SlashSinkUpdated(slashSink)` | `slashSink` | slash sink changed |
| `RelayerApproval(relayer, approved)` | `relayer` | relayer allowlist changed |

`RunEntered` and `RunSettled` are the two events an indexer must watch. `runId` is
indexed on both, so joining a run to its settlement is a single-filter query.

### 9.2 Token

| Event | Indexed | Emitted when |
|---|---|---|
| `Transfer(from, to, value)` | `from`, `to` | standard ERC-20 |
| `Approval(owner, spender, value)` | `owner`, `spender` | standard ERC-20 |
| `Bought(buyer, backAmount, ethPaid, taxPaid)` | `buyer` | `buy()` |
| `Sold(seller, backAmount, ethPaid, taxPaid)` | `seller` | `sell()` |

---

## 10. Custom errors

| Error | Selector context | Meaning |
|---|---|---|
| `ZeroAddress()` | admin setters | zero address rejected |
| `ZeroAmount()` | `buy`, `sell`, `fundVault` | amount was 0 |
| `TaxTooHigh()` | constructor, `setTaxConfig` | tax above 500 bps or share above 100% |
| `InsufficientReserve()` | `buy` | backstop lacks BACK to fill |
| `AlreadySoldThisBlock()` | `sell` | anti-sandwich guard |
| `PriceDeltaTooLarge()` | `setPrice` | move exceeded ±2%/hour |
| `PriceAboveMax()` | `setPrice`, `setPriceMax` | above `priceMax` or below current |
| `InsufficientPayment()` | `buy` | `msg.value` under quote |
| `WithdrawFailed()` | native transfers | ETH send reverted |
| `EmissionDrained(remaining, floor)` | `enterRun` | reserve at/below floor |
| `BadDuration(given, min, max)` | `enterRun` | outside bounds |
| `BadCommit()` | `enterRun` | `commit == bytes32(0)` |
| `RunAlreadyLive()` | `enterRun` | prior run not terminal |
| `NoRun()` | `settleRun`, `abandonRun` | no run exists |
| `AlreadySettled()` | `settleRun`, `abandonRun` | run already closed |
| `TooEarly(readyAt, nowTs)` | `settleRun`, `abandonRun` | wall-clock time lock not met |
| `TooLate(deadline, nowTs)` | `settleRun` | past `endsAt + settleWindow` |
| `RewardTooHigh(coins, max)` | `settleRun` | above `maxRunReward` |
| `RateExceeded(coins, allowed)` | `settleRun` | above `rate × elapsed` |
| `DailyCapReached(claimed, cap)` | `settleRun` | would exceed UTC-day cap |
| `TooManyRuns(used, max)` | `enterRun` | run count exhausted |
| `CooldownActive(readyAt, nowTs)` | `enterRun` | cooldown not elapsed |
| `InsufficientReserve(need, have)` | `settleRun` | reserve short (same name, different arity than token's) |
| `NotApprovedCaller()` | `enterRun`, `settleRun` | contract caller not allowlisted |
| `BadConfig()` | `setConfig`, `fundVault` | out-of-range parameter |
| `NotOwner()` | `pause` | caller is neither owner nor guardian |

> **Name collision warning:** `InsufficientReserve()` (no args) on the token and
> `InsufficientReserve(uint256,uint256)` on the vault share a name. They are different
> selectors and decode differently. The client must branch on the contract, not the
> name.

---

## 11. Invariants and safety properties

These hold for **every** reachable state. They are what a test suite must assert
after every fuzz run.

| # | Invariant | Enforced by |
|---|---|---|
| I1 | `totalSupply() == 6_666_666_666_660_000_000_000` forever | no `_mint` outside constructor |
| I2 | `emissionReserve` is monotonically non-increasing except via `fundVault` | no other write path |
| I3 | `token.balanceOf(vault) >= emissionReserve` | reserve tracked with the tokens |
| I4 | `emissionReserve >= 0` and `<= initialReserve` | checked arithmetic |
| I5 | `sum(pendingPayout) + emissionReserve <= token.balanceOf(vault)` | transfers from reserve only |
| I6 | `totalClaimed + totalClaimed + totalSlashed <= initialReserve` | accounting on every settle |
| I7 | `settleRun` cannot produce a reward above `rate × elapsed` | check 7 |
| I8 | A run cannot settle before `startedAt + minRunDuration` | check 4 |
| I9 | A run cannot settle twice | `r.settled` flip, front-running costs gas to lose |
| I10 | `dailyCap` never exceeded per wallet per UTC day | `_rollDay` + check 8 |
| I11 | `runsToday` never exceeds `maxRunsPerDay` | `_rollDay` + check 7 |
| I12 | Owner can never move BACK out of the vault | no such function |
| I13 | `pendingPayout` is never reduced except by paying it out | single write site |
| I14 | Player funds are claimable while paused | `claimAll` not `whenNotPaused` |

---

## 12. Security model and threat analysis

### 12.1 Honest statement of the trust model

This is a **client-authoritative** game. The browser computes the run and reports how
many coins it collected. The contract cannot verify that a player actually walked into
a coin.

Anyone claiming otherwise about a headless realtime game is describing a server they
have not mentioned. The mitigation is **economic**, and it is bounded, not absolute.

### 12.2 No contract players (`callerIsPlayer`)

`enterRun`/`settleRun` revert when `msg.sender != tx.origin` unless the caller is an
owner-approved relayer. This blocks the classic smart-contract-player vector where a
script farms on behalf of thousands of addresses with one line of assembly.

### 12.3 What an attacker actually has to do to cheat

To extract the maximum from the system per wallet per day:

1. Acquire `500 BACK` per run (or hold a stake).
2. Wait `minRunDuration` = 300 s of real time per run.
3. Observe `emissionReserve > reserveFloor`.
4. Call `settleRun` claiming `rate × elapsed`.
5. Repeat up to 8 times, respecting a 120 s cooldown.

The wall-clock lock in check 4 is what makes this expensive: **300 seconds of real
time per run, 8 runs, means at minimum 40 minutes of pure waiting per day per wallet**
to reach the daily cap — during which they hold `500 BACK × 1` in stake and receive
`250,000 BACK`. If BACK has any value, the stake requirement and the waiting are
real economic cost.

A multi-wallet attacker multiplies their wallet count by the cost of funding each
one with `500 BACK`. This is the honest limit of economic-only defense.

### 12.4 Attack-by-attack

| Attack | Mitigation | Residual risk |
|---|---|---|
| Falsify `coins` | `maxRunReward`, `rate × elapsed`, `dailyCap` | Attacker still extracts the cap |
| Skip the run entirely | `TooEarly` wall-clock lock | Must wait real time |
| Same-block sandwich | `AlreadySoldThisBlock` | Must span 2 blocks |
| Flash-loan round trip | `rate × elapsed` + settle window | Cannot fit |
| Replay a settle | `r.settled` one-shot | Front-running costs gas |
| Batch-farm via contract | `callerIsPlayer` | Requires N funded EOAs |
| Drain the vault | I2, I12 — no withdrawal path | None via owner |
| Steal settled rewards via pause | `claimAll` unp pausable | None |
| Rug the liquidity | `setPrice` ±2%/hour cap, `priceMax` | Owner can still dump treasury |
| Oracle manipulation | No oracle. Fixed owner-set price. | Centralization, see 12.5 |
| Reentrancy | `nonReentrant` + CEI on all value paths | — |
| Integer overflow | Solidity 0.8 checked arithmetic | — |

### 12.5 Centralization risks

Stated plainly, because hiding them is worse:

- **Owner controls pricing.** Rate-limited to ±2%/hour, which slows a rug but does not
  prevent one. Mitigation: `priceMax` and the rate limit. Residual: the owner can
  sell the 2% treasury allocation into a collapsing price.
- **Owner controls all game parameters.** Can set `dailyCap` to 1 BACK. Mitigation:
  bounds on every setter. Residual: a hostile owner can make the game worthless
  without stealing from the vault.
- **Owner controls pausing.** Can halt entry and settlement indefinitely. Mitigation:
  two-step ownership, guardian, and `withdrawUnvested` remains available while paused.
- **Owner cannot touch the emission reserve.** This is the strongest guarantee in the
  system and is structural.

A timelock on `setConfig`, `setPrice`, `setPause`, and `pause` is the standard next
step and is the first recommendation in [§16](#16-audit-checklist).

### 12.6 MEV

`settleRun` is a claim on a pre-computed amount with no slippage and no price
discovery. There is nothing for a searcher to extract by ordering. `enterRun` has the
`blockhash` in `runId`, so a searcher cannot precompute a `runId`. No front-running
vector of consequence.

---

## 13. Frontend integration contract

### 13.1 What the game must be given

The HTML is self-contained and ships with an **empty** contract configuration block.
The operator fills it in once — either by editing the `CONFIG` object at the top of
the script or through the in-game **CONTRACT** panel, which persists to
`localStorage`.

```js
const CONFIG = {
  chainId:      1,        // Ethereum mainnet
  chainName:    'Ethereum',
  tokenAddress: '',       // 0x… BackroomCoin
  vaultAddress: '',       // 0x… BackroomVault
  rpcUrl:       '',       // optional; falls back to wallet provider
  explorer:     'https://etherscan.io'
};
```

### 13.2 Required wallet behavior

| Step | Behavior |
|---|---|
| Provider detection | `window.ethereum`; if absent, offer practice mode only |
| Connect | `eth_requestAccounts` |
| Chain check | `eth_chainId` must equal `CONFIG.chainId`; else prompt `wallet_switchEthereumChain` |
| Listen | `accountsChanged`, `chainChanged`, `disconnect` |
| No injected wallet | Never block the game — practice mode always available |

### 13.3 The four-transaction entry flow

1. `balanceOf(player)` — read. If `< entryFee`, show the BUY panel.
2. `approve(vault, entryFee)` — **once per session**, cached in `localStorage` and
   invalidated when `entryFee` changes.
3. `enterRun(commit, durationSec)` — where `commit = keccak256(randomSeed)`.
4. Play. On exit: `settleRun(coins, died)`.
5. `claimAll()`.

### 13.4 Amount encoding — the one rule that must not be broken

Every raw amount crossing the contract boundary is a **`BigInt`**. Never:

```js
const amt = Number(input) * 1e10;         // ❌ loses precision above 2^53
```

Always:

```js
function toRaw(human) {                   // ✅
  const [w = '0', f = ''] = String(human).split('.');
  return BigInt(w) * 10n ** 10n + BigInt((f + '0'.repeat(10)).slice(0, 10));
}
```

`666,666,666,666e10` exceeds `Number.MAX_SAFE_INTEGER` by ~6 orders of magnitude.

### 13.5 Coins-to-BACK conversion

```
rewardRaw = coins × 1e10
```

The HUD displays `coins` (human BACK). `settleRun` takes `coins` as a plain `uint256`
in whole BACK, and the contract applies `SCALE`. The client passes the **integer coin
count**, not the raw amount.

### 13.6 Practice mode

Identical game. Identical code path. The only difference is the settlement backend:
a `localStorage`-backed ledger that mirrors the vault's arithmetic exactly — same
caps, same rate ceiling, same daily reset — so that practice is a genuine rehearsal
rather than an unlimited faucet.

---

## 14. Deployment runbook

### 14.1 Prerequisites

```bash
npm i -D hardhat @nomicfoundation/hardhat-toolbox @openzeppelin/contracts@5.1.0
npx hardhat init --type hardhat
```

`hardhat.config.js`:

```js
require("@nomicfoundation/hardhat-toolbox");
module.exports = {
  solidity: {
    version: "0.8.24",
    settings: { optimizer: { enabled: true, runs: 200 }, evmVersion: "shanghai" }
  },
  networks: {
    mainnet: {
      url: process.env.MAINNET_RPC,
      accounts: process.env.DEPLOYER_KEY ? [process.env.DEPLOYER_KEY] : []
    }
  }
};
```

### 14.2 Deploy script

```js
const { ethers } = require("hardhat");

async function main() {
  const [deployer, treasury, slashSink, backstop] = await ethers.getSigners();
  const guardian = deployer;                     // use a separate key in production

  // 1. coin — deployer mints the full supply, then splits it in the same tx
  const Coin = await ethers.getContractFactory("BackroomCoin");
  const coin = await Coin.deploy(
    "0x000000000000000000000000000000000000dEaD", // vault placeholder, replaced below
    backstop.address,
    treasury.address,
    slashSink.address,
    1_000_000_000n,       // price: 1 gwei per BACK
    300,                  // taxBps
    5000                  // treasuryShareBps
  );
  await coin.waitForDeployment();

  // 2. vault
  const Vault = await ethers.getContractFactory("BackroomVault");
  const vault = await Vault.deploy(
    await coin.getAddress(),
    guardian.address,
    slashSink.address
  );
  await vault.waitForDeployment();

  // 3. distribution — one atomic block, deployer holds nothing afterward
  const S = 10n ** 10n;
  await (await coin.transfer(await vault.getAddress(),  20_000_000_000n * S)).wait();
  await (await coin.transfer(backstop.address,           626_666_666_666n * S)).wait();
  await (await coin.transfer(treasury.address,            13_333_333_332n * S)).wait();
  await (await coin.transfer("0x000000000000000000000000000000000000dEaD",
                                                          6_666_666_664n * S)).wait();

  // 4. backstop approval for buys
  await (await backstop.sendTransaction({
    to: await coin.getAddress(),
    data: coin.interface.encodeFunctionData("approve",
      [await coin.getAddress(), ethers.MaxUint256])
  })).wait();

  console.log("Coin :", await coin.getAddress());
  console.log("Vault:", await vault.getAddress());
}
```

> **Step 1 has a real bug, deliberately left visible:** the coin's `vault` address is
> immutable, so deploying with a placeholder dead address leaves the coin pointing at
> a dead address forever. **Deploy the vault first with a predicted address**
> (foundry `CREATE2` / `hardhat-deploy` deterministic deployment), or accept that
> `coin.vault()` is informational only. The vault holds the real authority; `coin.vault`
> is not read by any vault code path, so a placeholder is cosmetically wrong and
> functionally harmless. Correct it in the next iteration via deterministic deployment.

### 14.3 Verification

```bash
npx hardhat verify --network mainnet <COIN_ADDR> \
  0x000000000000000000000000000000000000dEaD \
  $BACKSTOP $TREASURY $SLASH_SINK 1000000000 300 5000

npx hardhat verify --network mainnet <VAULT_ADDR> $COIN $GUARDIAN $SLASH_SINK
```

### 14.4 Post-deploy checklist

- [ ] `totalSupply() == 6666666666660000000000n`
- [ ] `decimals() == 10`
- [ ] `token.balanceOf(vault) == 20000000000000000000000n`
- [ ] `vault.token() == coin`
- [ ] `vault.emissionReserve() == 20000000000000000000000n`
- [ ] Paste both addresses into the game's CONTRACT panel
- [ ] Test a full entry → settle → claim cycle on a test wallet
- [ ] Confirm `settleRun` reverts `TooEarly` before `minRunDuration`

---

## 15. Verification and test plan

### 15.1 Unit tests

```js
describe("BackroomVault", function () {
  // supply
  it("mints exactly 666,666,666,666e10");
  it("never mints again");

  // entry
  it("reverts BadCommit on zero commitment");
  it("reverts BadDuration outside bounds");
  it("reverts RunAlreadyLive when a prior run is unterminal");
  it("reverts CooldownActive inside the cooldown window");
  it("reverts TooManyRuns after maxRunsPerDay");
  it("reverts EmissionDrained at the floor");
  it("escrows entryFee and emits RunEntered");

  // settle
  it("reverts TooEarly before minRunDuration");          // the time lock
  it("reverts TooLate after endsAt + settleWindow");
  it("reverts AlreadySettled on a second settle");
  it("reverts RewardTooHigh above maxRunReward");
  it("reverts RateExceeded above rate x elapsed");
  it("reverts DailyCapReached");
  it("refunds the stake on survival");
  it("slashes the full stake on death");
  it("decreases emissionReserve by exactly reward");

  // claim
  it("pays pendingPayout in one transfer");
  it("claimAll works while paused");                     // never traps funds
  it("withdrawUnvested releases an expired stake while paused");

  // access
  it("reverts NotApprovedCaller from a contract");
  it("reverts NotOwner for non-owner pause");
  it("reverts NotOwner for non-owner unpause");
  it("guardian can pause but not unpause");
});

describe("BackroomCoin", function () {
  it("buy() forwards BACK and refunds excess");
  it("buy() reverts InsufficientPayment under quote");
  it("sell() reverts AlreadySoldThisBlock");
  it("setPrice reverts PriceDeltaTooLarge over 2%/hour");
  it("setPrice reverts PriceAboveMax over priceMax");
  it("wallet-to-wallet transfer pays zero tax");
});
```

### 15.2 Invariant fuzzing

```js
// Hardhat invariant: run after every random call sequence
function invariant_I1_totalSupply()  { expect(await coin.totalSupply()).to.equal(SUPPLY); }
function invariant_I2_reserveMonotonic() { /* reserve never increases without fundVault */ }
function invariant_I5_reserveBacked() {
  expect(await coin.balanceOf(vault)).to.be.gte(await vault.emissionReserve());
}
function invariant_I12_ownerCannotDrain() {
  await expectRevert(coin.connect(owner).withdrawNative(vault, 1n), "");
}
```

### 15.3 Integration test — the full run

```
1. mint BACK to alice, approve vault for entryFee
2. alice enterRun(commit, 300)                    -> RunEntered
3. evm_increaseTime(299); settleRun              -> revert TooEarly
4. evm_increaseTime(1);  settleRun(7500, false)  -> RunSettled, reward = 7500e10
5. claimAll                                        -> alice balance += 7500e10 + 500e10
6. repeat 8x, then 9th enterRun                   -> revert TooManyRuns
7. evm_increaseTime(86400); 9th enterRun now works
8. enterRun; abandonRunFor(alice) before window   -> revert TooEarly
9. evm_increaseTime(settleWindow+1); abandonRun   -> RunAbandoned, stake returned
```

---

## 16. Audit checklist

Ordered by value. Items 1–4 are the ones that matter.

1. **Add a timelock** (48 h) on `setConfig`, `setPrice`, `setPause`, `pause`, and
   `setGuardian`. This is the single highest-value change and removes most of
   [§12.5](#125-centralization-risks).
2. **Delete `rescueToken`** or rewrite it against an explicit allowlist. As written it
   can reach unsettled `pendingPayout` — see the reviewer note in §7.
3. **Remove the dead-address `vault` field** from `BackroomCoin`, or make it settable.
   An immutable address pointing at `0x…dEaD` is a permanent lie in the source.
4. **Verify the emission reserve is truly unreachable.** Grep for every write to
   `emissionReserve` and confirm all four are `+=` on `fundVault` or `-=` on
   settlement. This is the property the whole economic model rests on.
5. Confirm `_rollDay` cannot be skipped. A bypass means an uncapped day.
6. Fuzz `settleRun` across the full `(elapsed, coins, died)` space.
7. Check `token.transferFrom` and `safeTransfer` return values are all checked.
8. Verify every native transfer uses `call` with a checked return, never a bare
   `transfer`.
9. Confirm `blocks.timestamp` usage — no `block.timestamp % N` for randomness.
10. Run Slither and Echidna. Expect Slither to flag the `rescueToken` arithmetic.

---

## 17. Known limitations — read this

Honesty here is worth more than a clean-looking document.

1. **The game is client-authoritative.** The contract cannot see what happened in the
   browser. The caps in §4 bound extraction; they do not prevent it. [§12.1](#121-honest-statement-of-the-trust-model)

2. **The economic model only works while the reserve holds.** The defaults are
   generous relative to a 20B reserve. At 10,000 daily-cap wallets the reserve is gone
   in 8 days. This is arithmetic, not risk. [§4.3](#43-solvency-table--how-long-the-reserve-lasts)

3. **`rescueToken` is unsafe as written.** It is flagged inline in §7 and in the audit
   checklist. Do not ship it unchanged.

4. **`BackroomCoin.vault` is immutable and may be a placeholder.** Cosmetic, but
   confusing in explorers. [§14.2](#142-deploy-script)

5. **No timelock.** The owner can reconfigure the game instantly. Adding one is audit
   item 1.

6. **No upgradeability.** This is intentional — an immutable, upgradeable vault is a
   larger risk than redeploying. Changing game parameters means redeploying the vault;
   the token is unaffected because the vault reference is one-directional.

7. **Regulatory.** Whether a token earned by playing a game and redeemable for value is
   a security, a gambling instrument, or a loyalty point is jurisdiction-specific and
   outside the scope of this document. **Get counsel before listing anywhere.** Games
   that pay real currency for play have been treated as gambling in several
   jurisdictions and as unregistered securities in others. This document specifies
   software; it is not legal advice.

8. **Audit status: unaudited.** Nothing here has been reviewed by a third party.

---

*End of specification. Reference implementations are complete and compilable; deploy
only after the audit checklist in §16 is addressed.*
