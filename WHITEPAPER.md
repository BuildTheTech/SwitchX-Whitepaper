# SwitchX Protocol Whitepaper

**Version 1.0 — February 2026**

---

## Table of Contents

1. [Abstract](#1-abstract)
2. [Introduction](#2-introduction)
3. [Protocol Architecture Overview](#3-protocol-architecture-overview)
4. [Concentrated Liquidity Engine](#4-concentrated-liquidity-engine)
5. [Adaptive Fee System](#5-adaptive-fee-system)
6. [SWITCH Token Economics](#6-switch-token-economics)
7. [Vote-Escrowed Governance](#7-vote-escrowed-governance)
8. [Gauge Voting & Fee Distribution](#8-gauge-voting--fee-distribution)
9. [Farming & Auto-Lock Mechanism](#9-farming--auto-lock-mechanism)
10. [MEV Recapture System](#10-mev-recapture-system)
11. [Automated Liquidity Management](#11-automated-liquidity-management)
12. [Security Model](#12-security-model)
13. [What Sets SwitchX Apart](#13-what-sets-switchx-apart)
14. [Emission Math Appendix](#14-emission-math-appendix)
15. [References](#15-references)

---

## 1. Abstract

SwitchX is a concentrated liquidity decentralized exchange protocol built on PulseChain that combines a custom ve(3,3) governance model with novel mechanisms for supply scarcity, MEV recapture, and sustainable rewards. Unlike existing ve(3,3) implementations that suffer from perpetual inflation and mercenary capital, SwitchX introduces a hard-capped token supply with a 2.5-year emission schedule, permanent burn locks with a 2x voting power multiplier, automatic farming reward locking, protocol fee buybacks, drip-based reward smoothing, and an integrated MEV recapture system that redistributes extracted value back to governance participants. Together, these mechanisms create a self-reinforcing flywheel where long-term commitment is rewarded, supply progressively contracts, and protocol value accrues to those most aligned with the system's success.

---

## 2. Introduction

### 2.1 The Problem

Existing ve(3,3) decentralized exchanges — pioneered by Solidly and iterated upon by Velodrome and Aerodrome — introduced a powerful alignment mechanism: voters direct emissions to pools, earn trading fees from those pools, and lock tokens to gain voting power. This creates a feedback loop between liquidity provision and governance participation.

However, these protocols share several structural weaknesses:

- **Perpetual inflation.** Emissions never end. Token supply grows indefinitely, diluting existing holders and creating persistent sell pressure. The system depends on continuous demand growth outpacing supply expansion — a condition that rarely holds long-term.

- **Mercenary capital.** Without meaningful lock commitment mechanisms, large holders can lock briefly, extract maximum emissions, and exit. The absence of early exit options means locked capital becomes an illiquid liability, discouraging genuine long-term participation.

- **Reward volatility.** Trading fees are distributed immediately and entirely in the period they accrue. This creates volatile, unpredictable reward streams that make governance yields difficult to underwrite and incentivize short-term vote-hopping.

- **MEV extraction.** Automated market makers are prime targets for sandwich attacks and arbitrage extraction. In standard ve(3,3) systems, this value is captured by external bots rather than being returned to the protocol and its participants.

- **Static fees.** Fixed fee tiers fail to adapt to market conditions. During high volatility, fees may be too low (leaving value on the table for MEV actors); during low volatility, fees may be too high (repelling volume).

- **Manual liquidity management.** Concentrated liquidity requires active position management. Without automation, liquidity fragments across stale ranges, reducing capital efficiency and increasing impermanent loss.

### 2.2 The SwitchX Protocol Thesis

SwitchX addresses these problems through a coordinated set of mechanisms:

1. **Fixed supply with terminal emissions.** A hard cap of 1 billion SWITCH tokens with all emissions completing within 2.5 years. After that, no new tokens are ever minted.

2. **Burn locks for permanent alignment.** Holders can irreversibly burn their tokens in exchange for permanent, non-decaying voting power at a 2x multiplier — creating genuine skin in the game.

3. **Automatic reward locking.** A configurable percentage of farming rewards are automatically locked into veNFTs at maximum duration, compounding governance alignment without requiring manual action.

4. **Protocol fee buybacks.** A portion of all trading fees is used to buy SWITCH on the open market, creating consistent demand-side pressure.

5. **Drip-based reward smoothing.** Rather than distributing all fees immediately, rewards are pooled and released at a configurable rate per period, reducing volatility and extending reward duration.

6. **MEV recapture.** An integrated afterSwap hook system detects arbitrage opportunities and executes backrun trades, returning the profit to ve(3,3) voters.

7. **Adaptive fees.** Volatility-responsive fee curves, same-block MEV surcharges, and cross-DEX oracle-based LVR fees ensure the protocol captures fair value across all market conditions.

8. **Automated liquidity management.** Protocol-native ALM vaults manage concentrated liquidity positions using TWAP-driven rebalancing with volatility-adaptive range widths.

---

## 3. Protocol Architecture Overview

SwitchX is built on a modular architecture where each layer composes with the next through well-defined interfaces. This design allows independent upgrades to individual components without disrupting the broader system.

```
┌─────────────────────────────────────────────────────────────────┐
│                        VOTING LAYER                             │
│  VotingEscrow (veNFT) · Voter · Minter · Gauges · Rewards      │
│  ProtocolFeeManager · DripVotingReward                          │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────┐
│                      ALM VAULT LAYER                            │
│  ALMVault (ERC20) · HybridRebalanceManager · ALMKeeper          │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────┐
│                       FARMING LAYER                             │
│  FarmingCenter · V4EternalFarming · EternalVirtualPool           │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────┐
│                       PLUGIN LAYER                              │
│  SwitchXBasePlugin: DynamicFee · BackrunFee · CrossDexOracleFee  │
│  FeeDiscount · SecurityPlugin · LimitOrderPlugin                │
│  MevBackrunPlugin · VolatilityOraclePlugin · FarmingProxyPlugin │
│  AlmPlugin                                                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────┐
│                      PERIPHERY LAYER                            │
│  NonfungiblePositionManager · SwapRouter · Quoter               │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────┐
│                        CORE LAYER                               │
│  V4Pool · V4Factory · V4PoolDeployer · V4CommunityVault         │
└─────────────────────────────────────────────────────────────────┘
```

### Layer Descriptions

**Core** — The AMM engine. V4Pool implements concentrated liquidity with tick-based pricing, callback-secured token transfers, and a three-tier fee architecture. V4Factory deploys pools deterministically via CREATE2 with role-based access control.

**Periphery** — User-facing contracts. NonfungiblePositionManager wraps liquidity positions as ERC721 tokens. SwapRouter provides stateless swap execution. Quoter enables off-chain swap simulation.

**Plugin** — An extensible hook system attached to each pool. Plugins use a bitmap-driven configuration (uint8 `pluginConfig`) to selectively enable lifecycle hooks: `beforeInitialize`, `afterInitialize`, `beforeModifyPosition`, `afterModifyPosition`, `beforeSwap`, `afterSwap`, `beforeFlash`, `afterFlash`, and `handlePluginFee`. The `SwitchXBasePlugin` composes ten sub-plugins into a single contract that handles adaptive fees, MEV protection, cross-DEX LVR capture, oracle tracking, farming integration, ALM rebalancing, limit orders, security states, fee discounts, and MEV backrun execution.

**Farming** — Perpetual reward distribution. V4EternalFarming manages incentive programs with virtual tick-based reward accounting. FarmingCenter coordinates position enrollment and reward collection, including the auto-lock mechanism.

**ALM Vault** — Automated liquidity management. ALMVault wraps concentrated liquidity positions as fungible ERC20 shares using a two-position strategy. HybridRebalanceManager implements a volatility-adaptive state machine that adjusts position ranges based on TWAP signals.

**Voting** — The ve(3,3) governance stack. VotingEscrow manages veNFTs with time-locked and permanent burn lock variants. Voter coordinates gauge voting, emission distribution, and fee collection. Minter controls the fixed emission schedule with halving phases. ProtocolFeeManager handles fee-to-SWITCH buybacks. DripVotingReward smooths fee distribution over time.

---

## 4. Concentrated Liquidity Engine

### 4.1 V4Pool Mechanics

At the core of SwitchX is the V4Pool contract — a concentrated liquidity AMM where liquidity providers allocate capital to specific price ranges (ticks). This is a ground-up recreation of Uniswap V4, itself derived from the Uniswap V3 concentrated liquidity design with significant modifications.

Each pool tracks:
- A global price (`sqrtPriceX96`) and current tick
- Per-tick liquidity boundaries
- Fee accumulators for both tokens
- A plugin attachment point for lifecycle hooks

The tick-based system allows liquidity to be concentrated where it matters most, improving capital efficiency by orders of magnitude compared to constant-product AMMs.

### 4.2 Three-Tier Fee Architecture

Total swap fee = **Base Fee** + **Override Fee** + **Plugin Fee**

| Tier | Description | Set By |
|------|-------------|--------|
| **Base Fee** | Static fee set per pool (configurable, 100–10,000 bps) | Pool owner / factory |
| **Override Fee** | Dynamic fee calculated by the plugin's `beforeSwap` hook | DynamicFeePlugin |
| **Plugin Fee** | Additional revenue accumulated by the plugin contract | handlePluginFee callback |

The DynamicFeePlugin calculates the override fee based on recent price volatility, ensuring fees adapt to market conditions in real time. During calm markets, fees decrease to attract volume; during volatile periods, fees increase to compensate liquidity providers for impermanent loss.

### 4.3 Community Fee

A configurable portion of all swap fees is diverted to the ve(3,3) system through the community fee mechanism. At launch, this is set to **75%** (`communityFee = 750`), meaning 75% of every swap fee collected flows to the Voter contract for distribution to veNFT holders who vote on that pool's gauge. The remaining 25% accrues directly to liquidity providers (including ALM vault depositors).

This split maximizes voter revenue — strengthening the ve(3,3) flywheel — while still leaving meaningful fee income for ALM vault depositors alongside their farming emissions. Industry benchmarks (Aerodrome, Velodrome) route 100% to voters; SwitchX's 75% ensures ALM vault depositors retain a direct fee incentive even after emissions end.

This creates the core ve(3,3) feedback loop: voters direct emissions to pools → pools generate fees → fees flow to voters → voters are incentivized to direct emissions to the highest-yielding pools.

### 4.4 Callback Security Pattern

All value transfers in V4Pool use a callback-based validation pattern to prevent flash loan attacks and ensure atomic execution:

```
User calls pool.swap()
  → Pool calculates amounts
  → Pool calls user.v4SwapCallback(amount0Delta, amount1Delta, data)
  → User transfers tokens to pool inside callback
  → Pool validates received amounts
```

The same pattern applies to `v4MintCallback` (adding liquidity) and `v4FlashCallback` (flash loans). This ensures that tokens are always received before any state changes are finalized, preventing reentrancy and manipulation attacks.

---

## 5. Adaptive Fee System

### 5.1 Volatility-Based Dynamic Fees

The `DynamicFeePlugin` calculates swap fees using a sigmoid-like adaptive fee curve parameterized by:

| Parameter | Description |
|-----------|-------------|
| `alpha1`, `alpha2` | Fee curve amplitudes |
| `beta1`, `beta2` | Volatility inflection points |
| `gamma1`, `gamma2` | Curve steepness factors |
| `baseFee` | Minimum fee floor |

The fee is computed as a function of the average volatility observed through the TWAP oracle:

```
fee = baseFee + f(volatilityAverage, alpha1, alpha2, beta1, beta2, gamma1, gamma2)
```

When `alpha1` and `alpha2` are both zero, the plugin returns a flat `baseFee` — allowing pools to opt out of dynamic pricing.

**Tiered Fee Configuration:**

Each pool has its adaptive fee curve configured independently based on competitive dynamics and asset characteristics:

| Pool | Base Fee | Max Fee | Strategy |
|------|----------|---------|----------|
| **WPLS/DAI** | 0.20% | 1.00% | Competitive — undercuts PulseX's fixed 0.26% at low volatility |
| **SWITCH/DAI** | 0.30% | 1.00% | Monopoly — SwitchX is the only venue for SWITCH trading |
| **SWITCH/WPLS** | 0.30% | 1.00% | Monopoly — same rationale as SWITCH/DAI |
| **USDC/DAI** | 0.20% | 0.30% | Stablecoin — narrow range, fee stays near base during normal conditions |

The WPLS/DAI base fee is set to capture more value per swap than the default while remaining competitive with PulseX (0.26% fixed). At low volatility, the adaptive fee sits near the 0.20% base — a meaningful undercut. During volatile periods, fees rise toward 1.0% to compensate LPs for impermanent loss.

SWITCH token pairs use higher base fees because SwitchX holds a monopoly on SWITCH liquidity. Every SWITCH trade must route through SwitchX, so there is no competitive pressure to minimize fees. The 0.30% base captures more value for voters on every SWITCH trade while the 1.0% max provides IL protection during volatile markets.

The USDC/DAI stablecoin pair uses a narrow adaptive range (0.20%–0.30%) because stablecoins exhibit minimal volatility. The fee sits near 0.20% under normal conditions and only rises to 0.30% during depeg/stress events.

### 5.2 MEV Protection (Bot-Proof Backrun Fee)

A **sandwich attack** is the most common form of MEV extraction on DEXs: an attacker front-runs a user's swap to move the price against them, then back-runs (reverses direction) immediately after to capture the difference as profit. The victim receives worse execution, effectively paying a hidden tax to the attacker. On many DEXs this occurs silently and at scale.

The `BackrunFeePlugin` eliminates this attack vector by making the back-run leg — the profitable reversal — prohibitively expensive:

- **Trigger condition**: A swap occurs in the same block as a previous swap, and the new swap's direction would profit from the prior price movement (i.e., it reverses the direction of the previous swap's tick change).
- **Surcharge formula**: `updatedFee = baseFee + (baseFee × backrunFeeFactor / 1000)`
- **Default factor**: `5000` (i.e., +500%, resulting in 6x the base fee)
- **Maximum factor**: `10000` (10x surcharge, 11x total)

**Why regular users are unaffected**: The surcharge only triggers on same-block direction reversals — a pattern that characterizes sandwich back-runs, not ordinary trading. Users swapping in different blocks, or in the same direction as the prior price movement, always pay the standard fee. In practice, this means normal traders never see the surcharge while sandwich bots face fees that exceed their expected profit, making the attack unprofitable.

### 5.3 Cross-DEX LVR Protection (Oracle Fee)

The BackrunFeePlugin (Section 5.2) protects against sandwich attacks that originate within SwitchX — but a significant class of MEV occurs across DEXs. When a user swaps on PulseX, the resulting price movement creates an arbitrage opportunity against SwitchX. External bots exploit this by trading on SwitchX at the normal fee to correct the price discrepancy. This is known as **Loss-Versus-Rebalancing (LVR)** — the value that liquidity providers lose to informed arbitrageurs correcting stale prices.

The `CrossDexOracleFeePlugin` addresses this by reading PulseX pair reserves on every swap and applying a proportional surcharge to corrective (arbitrage-direction) trades:

**How it works:**

1. The plugin reads the external PulseX pair's reserves to derive its current price
2. It compares this against the SwitchX pool's `sqrtPriceX96`
3. If the swap moves the SwitchX price **toward** the external price (corrective / arb direction), a surcharge is applied
4. The surcharge is proportional to the price deviation, minus the existing dynamic + backrun fee (no double-charging)
5. Safety guard: same-block reserve updates and stale oracle reserves (>30 minutes old) are ignored to reduce reserve-skew manipulation risk

**Surcharge formula:**

```
targetFee = deviationBps × captureRate
surcharge = max(0, targetFee − existingFee)
totalFee  = existingFee + min(surcharge, cap)
```

**Parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `stalenessFeeFactor` | 5,000 (50%) | Fraction of deviation targeted as total fee |
| `stalenessFeeCapHBips` | 30,000 (3%) | Maximum surcharge cap |
| `minDeviationBps` | 20 (0.2%) | Noise filter — deviations below this are ignored |
| `maxOracleReserveAge` | 1,800s (30 min) | Fail-open if external reserve timestamp is stale |

**Direction detection:**

The plugin only surcharges swaps that correct the price discrepancy:
- If SwitchX price is **higher** than PulseX → only `zeroToOne` swaps (pushing price down) are surcharged
- If SwitchX price is **lower** than PulseX → only `oneToZero` swaps (pushing price up) are surcharged
- Swaps in the opposite (non-corrective) direction always pay the normal fee

**Interaction with other fee components:**

The `max(0, targetFee − existingFee)` formula ensures that the backrun fee (Section 5.2) and cross-DEX oracle fee never double-charge. If the backrun fee already exceeds the cross-DEX target, no additional surcharge is applied. The total fee is effectively `max(backrunFee, crossDexTargetFee)`.

**PulseX pair auto-selection:**

The `setPulseXPairFromFactories()` function queries both PulseX V1 and V2 factories, scores each pair by `reserve0 × reserve1` (liquidity depth), and selects the deepest-liquidity pair. Token ordering alignment is auto-detected. If neither factory has the pair, the feature gracefully degrades (no surcharge applied).

**Why this matters for LPs:**

Without LVR protection, liquidity providers subsidize every arbitrage trade — each time an external bot corrects a stale SwitchX price, the LP effectively sells at the old (worse) price. The cross-DEX oracle fee ensures that a portion of this correction value is captured as fee revenue for the pool, making LP positions more profitable and reducing the hidden cost of providing liquidity.

### 5.4 Fee Discount Plugin

The `FeeDiscountPlugin` allows per-user fee reductions, enabling loyalty programs, partner integrations, or volume-based discounts. Discounts are applied after the base fee and backrun surcharge are calculated.

### 5.5 Security Plugin

The `SecurityPlugin` controls pool access through configurable security states:

| State | Swaps | Adds | Burns | Flash |
|-------|-------|------|-------|-------|
| **Enabled** | Yes | Yes | Yes | Yes |
| **Disabled** | No | No | No | No |
| **BurnOnly** | No | No | Yes | No |
| **FlashDisabled** | Yes | Yes | Yes | No |

This allows pool operators to pause trading during emergencies, enable graceful wind-downs (burn-only mode), or disable flash loans while keeping swaps active.

### 5.6 Limit Order Plugin

The `LimitOrderPlugin` enables on-chain limit order execution through the `afterSwap` hook — no off-chain keepers or relayers required.

**Mechanism:**

1. **Placement**: Users deposit single-sided liquidity at a specific tick via `LimitOrderManager.place()`. Orders at the same tick and direction are grouped into a shared **epoch** for gas-efficient batch settlement.
2. **Execution**: When a swap crosses the order's tick, the `afterSwap` hook detects the crossing, removes the epoch's liquidity, and marks it as filled. Execution is atomic with the triggering swap — no MEV opportunity exists between detection and fill.
3. **Withdrawal**: After an epoch is filled, users withdraw their pro-rata share of the converted tokens via `withdraw()`.

**Properties:**
- **No external dependencies**: Orders execute as part of normal swap flow, eliminating reliance on off-chain bots
- **Gas efficient**: Epoch batching amortizes gas costs across all orders at the same price point
- **Composable**: Works alongside dynamic fees, farming, and ALM without conflicts

Limit orders are disabled for pools with narrow tick spacing (e.g., SWITCH token pools) where large price movements could cause excessive tick iteration in the `afterSwap` hook.

---

## 6. SWITCH Token Economics

### 6.1 Supply Parameters

| Parameter | Value |
|-----------|-------|
| **Hard Cap** | 1,000,000,000 SWITCH |
| **Premint** | 23,000,000 SWITCH (2.3% of cap) |
| **Emission Budget** | 977,000,000 SWITCH (97.7% of cap) |
| **Emission Duration** | 2.5 years |

The SWITCH token has a **hard cap** enforced at the smart contract level. No more than 1 billion tokens can ever exist. This stands in contrast to standard ve(3,3) implementations where emissions continue indefinitely.

### 6.2 Premint Allocation

The 23M premint is allocated as follows:

| Allocation | Amount | % of Premint | Purpose |
|-----------|--------|-------------|---------|
| Governance Burn Lock | 8,000,000 | 34.8% | Permanent 2x voting power (16M vePower) |
| Liquidity Bootstrapping | 12,000,000 | 52.2% | Initial pool liquidity (SWITCH/DAI, SWITCH/WPLS) |
| Bribe Budget | 2,000,000 | 8.7% | Epochs 1-12 voting incentives |
| Operational Buffer | 1,000,000 | 4.3% | Team locks, gas, contingencies |

The 8M burn lock creates an initial governance position with 16M vePower (via the 2x burn multiplier), giving the protocol treasury >60% voting control at launch — sufficient to defend against hostile governance attacks during the bootstrapping phase.

### 6.3 Emission Schedule

Emissions follow a three-phase halving schedule over 2.5 years:

| Phase | Duration | Rate | Daily Emissions | Total Emitted |
|-------|----------|------|-----------------|---------------|
| Year 1 | 12 months | Base rate | ~1,644,000 SWITCH/day | ~600,060,000 |
| Year 2 | 12 months | Half rate | ~822,000 SWITCH/day | ~300,030,000 |
| Year 2.5 | 6 months | Quarter rate | ~411,000 SWITCH/day | ~75,007,500 |
| **Total** | **2.5 years** | — | — | **~977,000,000** |

**After 2.5 years, emissions stop permanently.** No tail emissions, no governance vote to extend them. The emission budget is fully consumed and the token supply is fixed forever.

### 6.4 Base Rate Formula

The base rate per second is derived from the total budget using the halving constraint:

```
baseRatePerSecond = (totalBudget × 8) / (13 × YEAR)
```

Where `YEAR = 365 days` in seconds. The factor `8/13` ensures the three halving phases sum correctly:

```
Year 1: base × YEAR                 = budget × 8/13
Year 2: (base/2) × YEAR             = budget × 4/13
Year 2.5: (base/4) × (YEAR/2)       = budget × 1/13
                                       ─────────────
Total:                               = budget × 13/13 = budget
```

This produces a base rate of approximately 19.03 SWITCH per second (≈1,644,000/day).

### 6.5 Treasury Allocation

**10%** of all emissions (`TREASURY_RATE = 1000`) are directed to the protocol treasury to maintain governance power in the early days & prevent hostile takeover. This will be scaled down over time as the protocol matures. The remaining 90% flows to the Voter contract for distribution to gauges based on veNFT votes.

### 6.6 Why Fixed Supply Matters

In perpetual-emission ve(3,3) systems, token holders face a coordination problem: they must continuously lock and vote just to avoid dilution. New emissions create selling pressure that existing holders must absorb. The system works only as long as demand for emissions exceeds their supply — a condition that eventually fails.

SWITCH eliminates this dynamic. After 2.5 years:
- No new tokens are minted, so there is no dilution
- The only sources of sell pressure are unlocking veNFTs and farming rewards
- Burn locks permanently remove tokens from circulation
- Fee buybacks create consistent buy pressure
- Early exit penalties burn a portion of exiting tokens

The result is a token with progressively decreasing circulating supply and sustained demand from fee buybacks — a fundamentally different economic trajectory than perpetual-emission alternatives.

---

## 7. Vote-Escrowed Governance

### 7.1 Standard veNFT Locks

Users lock SWITCH tokens for a chosen duration (up to 2 years / `MAXTIME`) to receive a veNFT — an ERC721 token representing their voting power.

- **Voting power** decays linearly over time: `power = lockedAmount × (timeRemaining / MAXTIME)`
- **Lock duration** is rounded down to the nearest week
- **Maximum lock**: 2 years (730 days) for maximum voting power
- Users can **increase amount** or **extend duration** but cannot decrease either
- Expired locks can be **withdrawn** to recover the underlying SWITCH tokens

### 7.2 Burn Locks (Permanent Commitment)

Burn locks are SwitchX's most distinctive governance feature. Instead of locking tokens for a finite period, holders can **irreversibly burn** their SWITCH tokens in exchange for permanent, non-decaying voting power. Unlike protocols such as Blackhole, where "burn" refers to permanently locked (but not destroyed) tokens, SwitchX burn locks are true burns: the underlying SWITCH is permanently destroyed, reducing total supply. This distinction is unique among ve(3,3) implementations.

**Mechanism:**
1. User calls `burn_lock(tokenId)` on an existing veNFT, or `create_burn(amount, recipient)` for a new one
2. The underlying SWITCH tokens are **permanently destroyed** via the `burn()` function
3. The veNFT receives permanent voting power equal to `burnedAmount × burnMultiplierBps / MAX_BPS`

**Parameters:**
| Parameter | Value |
|-----------|-------|
| `burnMultiplierBps` | 20,000 (2x) |
| `MAX_BPS` | 10,000 |

This means burning 100 SWITCH tokens grants 200 units of permanent voting power — equivalent to holding a maximum-duration lock of 200 SWITCH that never decays.

**Properties of burn locks:**
- Voting power **never decays** — it remains constant regardless of time
- Tokens are **permanently removed** from circulation — they cannot be recovered
- Burn locks **cannot be split** (prevents gaming via fragment-and-reconstruct)
- Burn locks **cannot early exit** (the tokens no longer exist)
- Burn locks **can be merged** with other burn locks (combining permanent power)
- Decaying locks **can be merged into burn locks** (the decaying tokens are burned and converted to burn power)
- Burn locks **cannot be merged into decaying locks** (prevents silent loss of burn power)

**Why burn locks matter:**

Burn locks create genuine, irreversible alignment. In standard ve(3,3), a "max-locked" whale can simply wait for expiry and dump. Burn lock holders have no such exit — their incentive is permanently tied to the protocol's success. The 2x multiplier compensates for this irreversibility while creating supply contraction that benefits all remaining holders.

### 7.3 Early Exit

SwitchX introduces an early exit mechanism that allows veNFT holders to unlock before expiry — in exchange for a penalty that scales with time remaining.

**Penalty Curve:**

The default curve is **inverse quadratic** (`FeeDecayType.INVERSE_QUADRATIC`), which keeps penalties high early in the lock and allows them to decrease as the lock approaches expiry:

```
progress = elapsed / totalDuration
feeRaw = 1 - progress²
feeBps = maxFee × feeRaw
```

Three curve options are available:

| Curve | Formula | Behavior |
|-------|---------|----------|
| **Linear** | `fee = max - (max - min) × elapsed / duration` | Straight-line decay |
| **Inverse Quadratic** | `fee = max × (1 - progress²)` | Slow start, fast finish (default) |
| **Inverse Cubic** | `fee = max × (1 - progress³)` | Very slow start, steep tail |

**Fee Parameters:**
| Parameter | Value |
|-----------|-------|
| `maxEarlyExitFeeBps` | 10,000 (100%) |
| `minEarlyExitFeeBps` | 1,000 (10%) |
| `feeSplitBps` | 5,000 (50/50) |

The penalty ranges from 10% (near expiry) to 100% (just after locking). The penalty amount is split:
- **50% burned** — permanently reducing circulating supply
- **50% distributed to voters** — rewarding those who maintain their commitments

**Anti-gaming protections:**
- `lockStartTime` is **propagated on merge** — the destination lock inherits the most recent (highest) start time from either source, preventing users from merging a new lock into an old one to reduce penalties
- `lockStartTime` is **propagated on split** — split tokens inherit the parent's start time, preventing users from splitting and selectively exiting with reduced penalties
- These rules ensure that the penalty curve always reflects the actual commitment timeline

### 7.4 Merge & Split Rules

**Merge** (`merge(from, to)`):

| From → To | Allowed | Behavior |
|-----------|---------|----------|
| Decaying → Decaying | Yes | Standard merge, power and lock time aggregated (uses latest end time) |
| Burn → Burn | Yes | Combines permanent burn power |
| Decaying → Burn | Yes | Decaying tokens are burned, burn power added at 2x multiplier |
| Burn → Decaying | No | Blocked — prevents silent burn-power loss |

**Split** (`split(from, weights[])`):
- Creates multiple new veNFTs from one, allocated by weight
- Only allowed for decaying locks (burn locks cannot be split)
- Child locks inherit the parent's `lockStartTime` and unlock time

### 7.5 Governance Power Timeline

Starting from the 8M governance burn lock (16M vePower):

| Time | Governance vePower | Max External vePower* | Governance Control |
|------|-------------------|----------------------|-------------------|
| Launch | 16,000,000 | 0 | 100% |
| Week 1 | 16,000,000 | ~5,750,000 | ~74% |
| Week 2 | 16,000,000 | ~11,500,000 | ~58% |
| Month 1 | 16,000,000 | ~25,000,000 | ~39% |

*Assumes 50% of new SWITCH emissions are locked at maximum duration (2 years)*

The permanent burn lock ensures the protocol treasury retains meaningful governance influence even as emissions distribute tokens to the broader community. By month 1, the treasury still holds ~39% control — sufficient to block hostile governance proposals while community governance becomes increasingly decentralized.

---

## 8. Gauge Voting & Fee Distribution

### 8.1 Weekly Epoch System

SwitchX operates on a weekly epoch cycle:

1. **veNFT holders vote** on which pools should receive emissions
2. **Minter distributes** tokens to Voter based on the emission schedule
3. **Voter allocates** tokens to gauges proportional to votes received
4. **Gauges stream** rewards to liquidity providers in their respective pools
5. **Trading fees** from each pool flow back to voters who supported that gauge

This creates the ve(3,3) alignment: voters are incentivized to direct emissions to the most productive pools (highest fees), which in turn generates the highest returns for those voters.

### 8.2 Community Fee Collection

When a swap occurs in any pool:
1. The community fee percentage (75%) of the swap fee is collected by `V4CommunityVault`
2. The Voter contract claims these fees during the `distribute()` call
3. Fees are routed through the `ProtocolFeeManager` for processing
4. Processed fees are deposited into the pool's `DripVotingReward` contract

### 8.3 ProtocolFeeManager Buyback

The `ProtocolFeeManager` converts a portion of collected fees into SWITCH tokens before distributing them to voters.

**Mechanism:**
1. Raw fee tokens (e.g., DAI, WPLS) arrive at the ProtocolFeeManager
2. The buyback percentage (`buybackBps = 5000`, i.e., 50%) determines how much to swap
3. The manager executes a swap through the SwitchX router: `fee_token → SWITCH`
4. The resulting SWITCH tokens are sent to voters alongside any passthrough (unconverted) fee tokens
5. The remaining 50% passes through as the original fee token

**Oracle Slippage Protection:**

Before executing any buyback swap, the manager checks oracle-based slippage guards:

| Check | Parameter | Purpose |
|-------|-----------|---------|
| Short-term | `shortTermMaxTick` | Detects recent price manipulation |
| Long-term | `longTermMaxTick` + `longTermSecondsAgo` | Detects sustained price deviation |

If either check fails, the entire fee amount passes through without a swap — ensuring voters always receive their rewards even when market conditions make buybacks inadvisable.

**Graceful Fallback:**

If the buyback swap reverts for any reason (insufficient liquidity, pool not initialized, etc.), the manager catches the error and falls back to full passthrough mode. Fees are never lost or stranded in the manager contract.

### 8.4 DripVotingReward

Standard ve(3,3) implementations distribute 100% of fees immediately in the period they accrue. This creates volatile, unpredictable reward streams — a large trade generates a spike, followed by periods of minimal returns.

SwitchX replaces this with the `DripVotingReward` contract, which pools fees and releases them gradually.

**Mechanism:**
1. Fees arrive via `notifyRewardAmount()` from the Voter during `distribute()`
2. The **drip rate** (`dripRate = 1000`, i.e., 10%) determines what fraction of the pool is released each period
3. On each period transition, the contract releases `poolBalance × dripRate / 10000` to the current period
4. Voters claim rewards proportional to their vote weight in each period

**Properties:**
- **Smoothing**: A single large fee event is spread over multiple periods, reducing reward volatility
- **Compounding**: The pool decreases geometrically (90% retained each period), creating a long tail of rewards
- **Catch-up**: If drip checkpoints are missed (no one interacts for several periods), up to 52 periods can be processed in a single call
- **No-vote protection**: If no veNFTs vote for a gauge in a period, fees are not released to that period (they remain in the pool for future periods when votes exist)
- **Bribe compatibility**: Direct bribes (`incentivize()`) bypass the drip mechanism and target the next period directly, preserving standard ve(3,3) bribe semantics

**Example:**

If a pool accumulates 10,000 DAI in fees with a 10% drip rate:

| Period | Released | Pool Remaining |
|--------|----------|----------------|
| 1 | 1,000 | 9,000 |
| 2 | 900 | 8,100 |
| 3 | 810 | 7,290 |
| 4 | 729 | 6,561 |
| ... | ... | ... |
| 10 | ~387 | ~3,487 |

After 10 periods, ~65% of the original fees have been distributed, with the remainder continuing to drip over subsequent periods.

---

## 9. Farming & Auto-Lock Mechanism

### 9.1 V4EternalFarming

`V4EternalFarming` manages perpetual reward streams for liquidity providers. Unlike time-bounded farming programs, eternal farms run continuously with adjustable reward rates.

Each pool can have one active incentive at a time, consisting of:
- A **primary reward token** (typically SWITCH, funded by Minter emissions)
- An optional **bonus reward token** (any ERC20)
- A **virtual pool** that tracks reward accrual using the same tick math as the underlying AMM
- A **minimum position width** requirement to prevent gaming via dust positions

Positions are automatically enrolled in farming when minted through the FarmingCenter under an active incentive.

### 9.2 Auto-Lock Mechanism

The auto-lock system is SwitchX's mechanism for compounding governance alignment through farming rewards. When a user claims SWITCH farming rewards, a configurable percentage is automatically locked into a veNFT at maximum duration (2 years).

**How it works:**

1. As farming rewards accrue, they are split into two buckets:
   - **Free rewards**: Claimable and transferable immediately
   - **Locked rewards**: Earmarked for automatic veNFT locking

2. The split is determined by the auto-lock percentage, which is configured per-pool with a tiered approach:
   - **Native pools** (SWITCH/WPLS, SWITCH/DAI): **50%** auto-locked — balances governance compounding with farmer liquidity
   - **Non-native pools** (WPLS/DAI): **75%** auto-locked — standard non-SWITCH pool, lighter friction for routing backbone
   - **Vampire pools** (USDC/DAI): **90%** auto-locked — maximum anti-mercenary protection; zero-IL stablecoin pair where farmers have no need for liquid SWITCH
   - **On-chain cap**: `MAX_AUTO_LOCK_PERCENTAGE = 9000` (90% maximum, enforced in `V4EternalFarming`)
   - **Per-pool override**: `autoLockConfigByPool[pool]` allows governance to adjust each pool independently
   - **Default fallback**: `defaultAutoLockPercentage` applies to pools without explicit configuration

3. When the user claims locked rewards:
   - If `immediateLockOnClaim` is enabled (default: true), the contract attempts to create a veNFT immediately via `create_lock_for(amount, MAXTIME, recipient)`
   - If the lock creation fails (e.g., contract interaction issues), the amount is credited to `_lockedCredits` for later manual locking via `lockReserved()`
   - If `immediateLockOnClaim` is disabled, all locked rewards go to `_lockedCredits`

4. Users with credited locked rewards can call `lockReserved(rewardToken, amount)` to create a veNFT at any time

**Exempt accounts:**

Addresses with the `AUTO_LOCK_EXEMPT_ROLE` receive 100% of rewards as free (no auto-lock). This is used for:
- ALM vault contracts (which need liquid tokens for rebalancing)
- Protocol-owned positions
- Any address the administrator designates

**Why auto-lock matters:**

Without auto-lock, farming emissions follow the typical ve(3,3) pattern: farmers receive tokens → sell immediately → price drops → emissions become less valuable → TVL decreases. Auto-lock breaks this cycle by ensuring that a significant portion of emissions are recycled back into governance commitments, creating compounding voting power for active participants and reducing immediate sell pressure.

---

## 10. MEV Recapture System

### 10.1 The MEV Problem

In any AMM, price movements create arbitrage opportunities. When the AMM price diverges from the broader market price, bots extract the difference through arbitrage trades. In traditional systems, this value is entirely captured by external actors (searchers, block builders) rather than the protocol or its users.

On PulseChain, the primary arbitrage venue is between SwitchX pools and PulseX (V1 and V2). Every price-moving swap on SwitchX creates a potential arbitrage opportunity against the corresponding PulseX pool.

### 10.2 MEV Backrun Architecture

SWITCH implements an `afterSwap` hook-based MEV recapture system through the `MevBackrunPlugin`:

```
1. User swap executes on SwitchX pool
2. afterSwap hook fires
3. If MEV is enabled and executor is configured:
   → Plugin calls executor.onAfterSwap(pool, recipient, zeroToOne, amount0, amount1)
4. Executor evaluates arbitrage opportunity against PulseX
5. If profitable, executor performs backrun trade:
   → Borrows from SwitchX pool (internal swap at zero fee)
   → Swaps on PulseX
   → Returns borrowed amount + profit
6. Profit is distributed as ve(3,3) voting rewards
```

### 10.3 Internal Swap Detection

When the MEV executor performs its backrun trade through the SwitchX pool, the plugin recognizes it as an **internal swap**:

```solidity
function _isMevInternalSwap(address sender, bytes calldata) internal view returns (bool) {
    return sender == mevExecutor;
}
```

Internal swaps receive special treatment:
- **Zero swap fee**: The executor pays no fees on its routing leg through SwitchX (`outFee = 1` sentinel in `beforeSwap`)
- **No backrun or LVR surcharge**: Both the backrun fee plugin and the cross-DEX oracle fee plugin are bypassed for internal swaps
- **Full hook execution**: Farming virtual pool ticks and limit order state are still updated to maintain consistency

**Complementary relationship with LVR protection:**

The MEV recapture system and the cross-DEX oracle fee (Section 5.3) work together to capture value from cross-DEX arbitrage. The MEV executor performs the protocol's own backrun trades at near-zero cost, capturing the arbitrage profit for ve(3,3) voters. For any residual cross-DEX deviation that the protocol's bot doesn't capture (e.g., due to gas costs, timing, or insufficient profitability), the cross-DEX oracle fee ensures that external arb bots still pay a proportional surcharge — meaning the LP value is protected regardless of who executes the corrective trade.

### 10.4 Profit Distribution

Profits from MEV recapture are distributed as ve(3,3) voting rewards, flowing through the same DripVotingReward system as regular trading fees. This means:
- MEV profits are smoothed over time (drip mechanism)
- Voters who direct emissions to MEV-active pools earn more
- The protocol captures value that would otherwise leave the ecosystem

### 10.5 PulseX Integration

The MEV executor is configured with PulseX factory addresses for both V1 and V2:

| Parameter | Address |
|-----------|---------|
| `PULSEX_FACTORY_V1` | `0x1715a3E4A142d8b698131108995174F37aEBA10D` |
| `PULSEX_FACTORY_V2` | `0x29ea7545def87022badc76323f373ea1e707c523` |
| `PULSEX_SWAP_ROUTER` | `0xDA9aBA4eACF54E0273f56dfFee6B8F1e20B23Bba` |

This allows the executor to evaluate and execute arbitrage against PulseX's on-chain liquidity in the same transaction as the user's swap.

---

## 11. Automated Liquidity Management

### 11.1 ALMVault Architecture

Each `ALMVault` is an ERC20 contract that wraps concentrated liquidity positions into fungible vault shares. Users deposit tokens and receive shares representing their proportional claim on the vault's assets.

**Two-Position Strategy:**

Each vault manages two positions simultaneously:

| Position | Range | Purpose |
|----------|-------|---------|
| **Base Position** | Wide range (e.g., 15-50% around current price) | Passive fee earning, backstop liquidity |
| **Limit Position** | Narrow range (e.g., 5-20% around current price) | Concentrated exposure, higher fee capture |

The base position provides broad liquidity coverage and resilience to price movements. The limit position captures the majority of fees by concentrating liquidity near the current price. Together, they balance capital efficiency against impermanent loss risk.

**One-Sided Deposits:**

Each vault is configured with `allowToken0` and `allowToken1` flags. In a typical dual-vault deployment per pool:
- Vault 1 accepts token0-denominated deposits (`allowToken0 = true`)
- Vault 2 accepts token1-denominated deposits (`allowToken1 = true`)

This simplifies the user experience — depositors provide a single asset, and the vault handles the rest.

### 11.2 HybridRebalanceManager

The `HybridRebalanceManager` implements a volatility-adaptive state machine that controls when and how vault positions are rebalanced.

**State Machine:**

| State | Condition | Behavior |
|-------|-----------|----------|
| **Normal** | Inventory balanced, low volatility | Standard position widths |
| **OverInventory** | Too much of deposit token | Widen ranges, reduce limit position aggressiveness |
| **UnderInventory** | Too little of deposit token | Tighten ranges, prioritize deposit token accumulation |
| **Special** | Extreme conditions | Safety widths, potentially pause deposits |

**TWAP-Driven Rebalancing:**

The rebalance manager uses two TWAP windows to evaluate market conditions:

| Window | Duration | Purpose |
|--------|----------|---------|
| **Fast TWAP** | 15 minutes | Short-term price trend detection |
| **Slow TWAP** | 2 hours | Medium-term price trend, smooths noise |

Rebalancing is triggered through the `AlmPlugin`'s `afterSwap` hook when:
1. Sufficient TWAP history is available (`_ableToGetTimepoints(slowTwapPeriod)`)
2. The swap is not an MEV internal swap
3. The system is not already in a rebalance
4. Price deviation exceeds the configured threshold

**Volatility Tiers:**

The manager classifies market conditions into volatility tiers and adjusts position widths accordingly:

| Tier | Threshold | Effect |
|------|-----------|--------|
| **None/Low** | < 600 bps | Standard position widths |
| **High** | 600 - 2,500 bps | Wider base and limit positions |
| **Extreme** | > 2,500+ bps | Maximum width safety positions, may pause deposits |

Width configuration is pool-type-aware:

| Parameter | Volatile Pairs | Stablecoin (USDC/DAI) |
|-----------|---------------|----------------------|
| baseNormal | 15% | 3% |
| limitNormal | 5% | 1% |
| baseHigh | 50% | 10% |
| limitHigh | 20% | 4% |

Volatile pairs use wide day-0 safety margins for launch; stablecoin pairs use tight ranges for capital efficiency near the 1:1 peg. Stablecoin managers also apply tighter volatility gates (`priceChange=0.5%`, `lowVol=0.5%`, `highVol=2%`, `extremeVol=5%`) to detect and react to even small depegging events earlier.

### 11.3 Farming Integration

ALM vault positions are automatically enrolled in farming incentives when they are minted (during rebalance). Since positions are owned by the vault contract (via the FarmingCenter), the vault collects farming rewards on behalf of depositors. ALM vaults are granted the `AUTO_LOCK_EXEMPT_ROLE` so that farming rewards remain liquid for rebalancing operations.

### 11.4 TWAP Manipulation Guards

To prevent oracle manipulation attacks on deposits and withdrawals, the vault implements TWAP-based sanity checks:

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `MIN_TWAP_PERIOD` | 30 seconds | Prevents flash loan manipulation |
| `MAX_TWAP_PERIOD` | 24 hours | Prevents stale data |

Deposit and withdrawal pricing uses the TWAP price rather than the instantaneous spot price, making sandwich attacks on vault operations unprofitable.

---

## 12. Security Model

### 12.1 Transfer Security

All token transfers in the core protocol use the callback pattern (Section 4.4), ensuring:
- No tokens are credited without actual receipt
- Flash loan attacks cannot manipulate state between credit and debit
- Reentrancy is prevented through state validation

### 12.2 Plugin Security States

The SecurityPlugin (Section 5.5) provides granular control over pool operations, allowing administrators to:
- Halt all trading during emergencies
- Enable burn-only mode for graceful wind-downs
- Disable flash loans while keeping swaps active

### 12.3 Oracle Manipulation Guards

Multiple layers of TWAP-based protection prevent oracle manipulation:

1. **ALM vault deposits/withdrawals**: Use TWAP pricing (min 30s window) instead of spot
2. **Fee buyback swaps**: Oracle slippage checks (short-term + long-term TWAP) must pass before execution
3. **Rebalance triggers**: TWAP comparison (fast vs. slow) prevents rebalancing on manipulated prices
4. **DripVotingReward**: No-vote protection prevents releasing rewards into periods without claimants

### 12.4 Access Control

Role-based access control is enforced throughout the protocol:

| Role | Scope | Controls |
|------|-------|----------|
| Factory Owner | Global | Pool deployment, role grants |
| Plugin Authority | Per-pool | Fee configuration, security states |
| `INCENTIVE_MAKER_ROLE` | Farming | Creating/deactivating incentives |
| `FARMINGS_ADMINISTRATOR_ROLE` | Farming | Emergency withdraw, auto-lock config |
| `AUTO_LOCK_EXEMPT_ROLE` | Farming | Exemption from auto-lock |
| `MANAGER_ROLE` | ALM | Vault parameter updates |
| `REBALANCER_ROLE` | ALM | Triggering rebalances |
| VotingEscrow Owner | Governance | veNFT parameters, burn multiplier |
| Voter | Governance | Gauge creation, emission distribution |

### 12.5 Upgradability

Critical contracts (VotingEscrow, Voter, Minter, ProtocolFeeManager, SWITCH token) use the UUPS proxy pattern with `Ownable2StepUpgradeable` for secure ownership transfers. Upgrades require explicit acceptance by the new owner, preventing accidental or malicious ownership changes.

### 12.6 Additional Safeguards

- **Reentrancy guards**: All state-modifying functions in the voting layer use `ReentrancyGuardUpgradeable`
- **SafeERC20**: All token transfers use OpenZeppelin's SafeERC20 to handle non-standard ERC20 implementations
- **Overflow protection**: Solidity 0.8.x built-in overflow checks, supplemented by SafeCast for critical conversions
- **DripVotingReward DoS protection**: Catch-up is capped at 52 periods per call to prevent unbounded gas consumption
- **Minimum shares**: ALM vaults enforce a minimum share amount (1000 wei) to prevent share inflation attacks

### 12.7 Governance Decentralization Roadmap

At launch, the deployer address holds admin keys for operational agility during the bootstrapping phase. The protocol follows a deliberate three-phase decentralization path:

**Phase 1 — Launch (Weeks 0-4)**: The deployer retains direct ownership of upgradeable contracts to enable rapid response to any issues discovered in the live environment. All admin actions are logged and auditable on-chain.

**Phase 2 — Timelock Transfer (Post-Stabilization)**: Once the protocol is operating smoothly, ownership of all upgradeable contracts (SWITCH token, VotingEscrow, Voter, Minter, ProtocolFeeManager, ALM vaults) is transferred to the `SwitchXTimelock` — an OpenZeppelin `TimelockController` with a configurable minimum delay. This ensures that all upgrades and parameter changes are publicly queued before execution, giving the community time to review and react.

**Phase 3 — Community Governance**: Proposer and executor roles on the timelock are transitioned to a community multisig or on-chain governance module, completing the path from deployer-controlled to community-governed.

The `SwitchXTimelock` contract is deployed as part of the protocol's periphery infrastructure and is ready for ownership transfer from day one.

### 12.8 Risk Factors

Users should be aware of the following risks inherent to the SwitchX protocol and DeFi participation generally:

**Smart Contract Risk**: Despite multiple audits (Section 15), smart contracts may contain undiscovered vulnerabilities. Novel features (burn locks, auto-lock, DripVotingReward, MEV recapture) introduce complexity beyond the audited upstream Uniswap V3 engine.

**Impermanent Loss**: Concentrated liquidity positions — whether managed directly or through ALM vaults — are subject to impermanent loss when prices move significantly. ALM vault automation reduces but does not eliminate this risk.

**Oracle Manipulation**: While TWAP-based protections guard against flash loan manipulation (Section 12.3), sustained multi-block price manipulation could potentially affect fee buyback pricing, ALM rebalance triggers, or deposit/withdrawal valuations.

**Admin Key Risk**: During the launch phase (Phase 1), the deployer retains direct control over upgradeable contracts. A compromise of the deployer key could result in malicious upgrades. This risk is mitigated by the planned timelock transfer (Section 12.7).

**PulseChain Dependencies**: SwitchX is deployed exclusively on PulseChain. Users are exposed to chain-level risks including validator concentration, network congestion, RPC availability, and bridge security for assets bridged from other chains.

**Liquidity Risk**: Low-liquidity pools may experience high slippage, and MEV recapture profitability depends on sufficient liquidity depth on both SwitchX and PulseX.

**Irreversibility of Burn Locks**: Burn locks permanently destroy the underlying SWITCH tokens. This is by design, but users should understand that burned tokens can never be recovered regardless of future circumstances.

**Regulatory Uncertainty**: The regulatory status of DeFi protocols, governance tokens, and ve(3,3) mechanisms varies by jurisdiction and may change over time.

---

## 13. What Sets SwitchX Apart

### 13.1 Comparison Table

| Feature | Standard ve(3,3) | SwitchX |
|---------|-----------------|--------|
| **Token Supply** | Perpetual tail emissions | Hard cap (1B), 2.5-year schedule, then zero |
| **Lock Types** | Time-locked only | + Burn locks (permanent, 2x voting power) |
| **Early Exit** | Not possible (wait for expiry) | Penalty-based with 3 curve options (10-100%) |
| **Fee Distribution** | Immediate 100% per period | Drip-based (10%/period from pool) |
| **Fee Processing** | Direct passthrough to voters | 50% SWITCH buyback + 50% passthrough |
| **Farm Rewards** | 100% liquid | 50-90% auto-locked for 2 years (three tiers: native 50%, non-native 75%, vampire 90%) |
| **MEV** | Extracted by external bots | Recaptured via afterSwap hook, redistributed |
| **MEV Protection** | None; users vulnerable to sandwich attacks | Backrun fee surcharge makes sandwiches unprofitable |
| **LVR Protection** | None; arb bots extract LP value at standard fees | Cross-DEX oracle fee captures LVR from corrective arb swaps |
| **Community Fee** | Varies (typically 100% to voters) | 75% to voters, 25% to LPs — balances flywheel strength with ALM vault returns |
| **Swap Fees** | Static fee tiers | Tiered volatility-adaptive fees (0.2%–0.3% base) + MEV surcharge + LVR surcharge |
| **Liquidity Management** | Manual positions | Automated ALM vaults with TWAP rebalancing |
| **Exit Penalty Distribution** | N/A | 50% burned, 50% to voters |

### 13.2 The SwitchX Flywheel

These mechanisms combine into a self-reinforcing economic flywheel:

```
                    ┌──────────────────────┐
                    │   Burn Locks (2x)    │
                    │   Reduce Supply      │
                    └──────────┬───────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
┌────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ Auto-Lock (50%)│  │  Fee Buybacks    │  │ Early Exit Burns │
│ Compounds      │  │  Buy Pressure    │  │ Supply Reduction │
│ Governance     │  │  on SWITCH       │  │ + Voter Rewards  │
└────────┬───────┘  └────────┬─────────┘  └────────┬─────────┘
         │                   │                     │
         └───────────┬───────┘                     │
                     │                             │
                     ▼                             │
          ┌──────────────────┐                     │
          │  Higher Voting   │◄────────────────────┘
          │  Power → More    │
          │  Fee Revenue     │
          └────────┬─────────┘
                   │
                   ▼
          ┌──────────────────┐
          │  More Emissions  │
          │  Directed to     │
          │  Productive Pools│
          └────────┬─────────┘
                   │
                   ▼
          ┌──────────────────┐
          │  Higher TVL &    │
          │  Volume → More   │
          │  Fees Collected  │
          └────────┬─────────┘
                   │
                   └──────────────► (back to top)
```

Each mechanism reinforces the others:
- **Burn locks** reduce supply and increase per-token voting power
- **Auto-lock** ensures farming rewards compound into governance rather than selling pressure
- **Fee buybacks** create consistent SWITCH demand from protocol revenue
- **Early exit burns** remove tokens while rewarding committed voters
- **Drip rewards** smooth volatility, making governance yields more predictable
- **MEV recapture** returns leaked value to voters
- **LVR protection** ensures cross-DEX arbitrage profits are shared with LPs
- **Adaptive fees** optimize revenue capture across market conditions
- **ALM vaults** maximize capital efficiency and fee generation

### 13.3 Terminal Emissions: The Endgame

After 2.5 years, SWITCH reaches a fundamentally different equilibrium than any perpetual-emission protocol:

- **No new supply**: The only SWITCH tokens that will ever exist are already in circulation (minus those that have been burned)
- **Ongoing demand**: Fee buybacks continue indefinitely as long as the protocol generates trading volume
- **Progressive scarcity**: Burn locks, early exit penalties, and fee buybacks continuously reduce circulating supply
- **Self-sustaining governance**: veNFT holders are rewarded purely from protocol revenue (trading fees + MEV recapture), not from inflationary emissions

This creates a protocol where long-term holders benefit from a progressively favorable supply/demand dynamic, without relying on continuous growth to offset dilution.

### 13.4 Post-Emission Sustainability (Year 3+)

A common objection to fixed-supply tokenomics is: "What happens when emissions end? Won't LPs leave?"

SwitchX is designed to be self-sustaining after emissions end, through multiple revenue streams that do not depend on token inflation:

**1. Trading Fee Revenue**: Liquidity providers earn swap fees directly from trading volume. This is the fundamental LP incentive that exists independent of any emission schedule — and it is the same mechanism that sustains Uniswap, Curve, and every other fee-generating AMM.

**2. Fee Buyback Demand**: The ProtocolFeeManager continues to buy SWITCH on the open market using 50% of all trading fees, creating persistent demand-side pressure that replaces emission-driven incentives.

**3. Community-Funded Farming**: V4EternalFarming supports permissionless reward top-ups via `addRewards()`. Any party — the treasury, partner protocols, or community DAOs — can fund farming incentives for specific pools by depositing reward tokens directly. This enables targeted, sustainable incentive programs without protocol-level inflation.

**4. veNFT Revenue Compounding**: Voters who have accumulated burn locks and auto-locked positions continue earning trading fees, MEV recapture rewards, and early exit penalty distributions. The lack of dilution from new emissions means these revenue streams become relatively more valuable over time.

**5. Progressive Scarcity**: Each year after emissions end, burn locks, early exit penalties, and fee buybacks continue reducing circulating supply — making the remaining tokens scarcer while revenue generation continues.

The key insight is that emissions are a bootstrapping mechanism, not a permanent dependency. Once sufficient TVL and trading volume are established, the protocol sustains itself through the same fee-based economics that power every successful DEX.

---

## 14. Emission Math Appendix

### 14.1 Full Emission Rate Table

| Metric | Value |
|--------|-------|
| Total Cap | 1,000,000,000 SWITCH |
| Premint | 23,000,000 SWITCH (2.3%) |
| Emission Budget | 977,000,000 SWITCH (97.7%) |
| Year 1 Rate | ~19.03 SWITCH/second |
| Year 1 Daily | ~1,644,000 SWITCH/day |
| Year 2 Rate | ~9.52 SWITCH/second |
| Year 2 Daily | ~822,000 SWITCH/day |
| Year 2.5 Rate | ~4.76 SWITCH/second |
| Year 2.5 Daily | ~411,000 SWITCH/day |
| Emission Duration | 2.5 years (912.5 days) |
| Treasury Share | 10% of all emissions |

### 14.2 Base Rate Derivation

Given:
- Total budget `B = 977,000,000 × 10¹⁸` (in token units)
- Year 1 emits at rate `r` for `Y` seconds
- Year 2 emits at rate `r/2` for `Y` seconds
- Year 2.5 emits at rate `r/4` for `Y/2` seconds
- `Y = 365 × 24 × 3600 = 31,536,000` seconds

Total emissions must equal the budget:

```
B = r×Y + (r/2)×Y + (r/4)×(Y/2)
B = r×Y × (1 + 1/2 + 1/8)
B = r×Y × (8/8 + 4/8 + 1/8)
B = r×Y × 13/8
```

Solving for `r`:

```
r = (B × 8) / (13 × Y)
r = (977,000,000 × 8) / (13 × 31,536,000)
r ≈ 19.0335 SWITCH/second
```

This matches the on-chain implementation:
```solidity
baseRatePerSecond = (totalBudget * 8) / (13 * YEAR);
```

### 14.3 Verification: Phase Totals

| Phase | Calculation | SWITCH Emitted |
|-------|-------------|----------------|
| Year 1 | 19.0335 × 31,536,000 | ~600,307,692 |
| Year 2 | 9.5168 × 31,536,000 | ~300,153,846 |
| Year 2.5 | 4.7584 × 15,768,000 | ~75,038,462 |
| **Sum** | | **~975,500,000** |

The small difference from 977M is due to integer rounding in the per-second rate. The Minter contract handles this with an end-of-schedule cleanup:

```solidity
if (tEffective == tEnd && emitted + budget < totalBudget) {
    budget = totalBudget - emitted;
}
```

This ensures the full budget is eventually distributed, with any rounding remainder allocated in the final period.

### 14.4 Governance Power Decay Timeline

Assuming the 8M burn lock (16M permanent vePower) and that 50% of new emissions are locked at max duration:

| Time | Cumulative Emissions | Assumed Locked (50%) | Max New vePower | Governance vePower | Governance % |
|------|---------------------|---------------------|-----------------|-------------------|-------------|
| Day 0 | 0 | 0 | 0 | 16,000,000 | 100% |
| Week 1 | ~11,508,000 | ~5,754,000 | ~5,754,000 | 16,000,000 | ~74% |
| Week 2 | ~23,016,000 | ~11,508,000 | ~11,508,000 | 16,000,000 | ~58% |
| Month 1 | ~49,320,000 | ~24,660,000 | ~24,660,000 | 16,000,000 | ~39% |
| Month 3 | ~147,960,000 | ~73,980,000 | ~73,980,000 | 16,000,000 | ~18% |
| Month 6 | ~295,920,000 | ~147,960,000 | ~147,960,000* | 16,000,000 | ~10%* |

*Note: External vePower starts decaying for earlier locks, so actual governance share will be higher than this simplified model suggests. Burn locks by other participants would further redistribute power.*

### 14.5 Supply Contraction Dynamics

After emissions end (year 2.5+), the circulating supply of SWITCH can only decrease through:

1. **Burn locks**: Tokens permanently destroyed, converted to voting power
2. **Early exit penalties**: 50% of penalty amount burned
3. **Fee buybacks**: Purchased SWITCH distributed to voters (not burned, but locked)

Assuming conservative post-emission activity:
- 5% of circulating supply burned via burn locks per year
- 1% of circulating supply burned via early exit penalties per year
- Fee buybacks create consistent lock demand

The result is a deflationary token with increasing scarcity over time — the opposite trajectory of perpetual-emission alternatives.

---

## 15. References

### Smart Contract Sources

| Contract | Path |
|----------|------|
| V4Pool | `src/core/contracts/V4Pool.sol` |
| V4Factory | `src/core/contracts/V4Factory.sol` |
| SwitchXBasePlugin | `src/plugin/contracts/SwitchXBasePlugin.sol` |
| DynamicFeePlugin | `src/plugin/contracts/plugins/DynamicFeePlugin.sol` |
| BackrunFeePlugin | `src/plugin/contracts/plugins/BackrunFeePlugin.sol` |
| CrossDexOracleFeePlugin | `src/plugin/contracts/plugins/CrossDexOracleFeePlugin.sol` |
| MevBackrunPlugin | `src/plugin/contracts/plugins/MevBackrunPlugin.sol` |
| SecurityPlugin | `src/plugin/contracts/plugins/SecurityPlugin.sol` |
| VotingEscrow | `src/voting/contracts/VotingEscrow.sol` |
| Minter | `src/voting/contracts/Minter.sol` |
| ProtocolFeeManager | `src/voting/contracts/ProtocolFeeManager.sol` |
| DripVotingReward | `src/voting/contracts/reward/DripVotingReward.sol` |
| V4EternalFarming | `src/farming/contracts/farmings/V4EternalFarming.sol` |
| ALMVault | `src/alm-vault/contracts/ALMVault.sol` |

### Key Parameters Summary

| Parameter | Value | Source |
|-----------|-------|--------|
| SWITCH_MAX_SUPPLY | 1,000,000,000 | `scripts/deployAll.js` |
| SWITCH_PREMINT | 23,000,000 | `src/voting/scripts/deploy.js` |
| MAXTIME | 2 years (730 days) | `VotingEscrow.sol:31` |
| burnMultiplierBps | 20,000 (2x) | `VotingEscrow.sol:103` |
| maxEarlyExitFeeBps | 10,000 (100%) | `VotingEscrow.sol:104` |
| minEarlyExitFeeBps | 1,000 (10%) | `VotingEscrow.sol:105` |
| feeDecayType | INVERSE_QUADRATIC | `VotingEscrow.sol:106` |
| feeSplitBps | 5,000 (50/50) | `VotingEscrow.sol:107` |
| DEFAULT_DRIP_RATE | 1,000 (10%) | `scripts/deployAll.js:34` |
| buybackBps | 5,000 (50%) | `src/voting/scripts/deploy.js:226` |
| TREASURY_RATE | 1000 (10%) | `scripts/deployAll.js:24` |
| DEFAULT_COMMUNITY_FEE | 750 (75%) | `scripts/deployAll.js:44` |
| Emission Duration | 2.5 years | `Minter.sol:196` |
| Base Rate Formula | `(budget×8)/(13×YEAR)` | `Minter.sol:198` |
| Backrun Fee Factor | 5,000 (6x total) | `BackrunFeePlugin.sol:13` |
| Max Backrun Factor | 10,000 (11x total) | `BackrunFeePlugin.sol:12` |
| LVR Capture Rate | 5,000 (50%) | `CrossDexOracleFeePlugin.sol:44` |
| LVR Surcharge Cap | 30,000 (3%) | `CrossDexOracleFeePlugin.sol:45` |
| LVR Min Deviation | 20 (0.2%) | `CrossDexOracleFeePlugin.sol:46` |
| ALM Fast TWAP | 15 minutes | `scripts/deployAll.js:51` |
| ALM Slow TWAP | 2 hours | `scripts/deployAll.js:52` |
| Min Rebalance Interval | 600 seconds | `scripts/deployAll.js:54` |
| Drip Catchup Limit | 52 periods | `DripVotingReward.sol:34` |

### Audits

The protocol has been audited at multiple layers. All reports are available in the [`audits/`](audits/) directory.

**SwitchX Protocol Audits** (covering SwitchX-specific features: ve(3,3) governance, burn locks, auto-lock, MEV recapture, DripVotingReward, ProtocolFeeManager, early exit, tiered emissions):
- **SpyWolf**: [`switchx-audit-report-spywolf.pdf`](audits/switchx-audit-report-spywolf.pdf)
- **33Audits**: [`switchx-audit-report-33audits.pdf`](audits/switchx-audit-report-33audits.pdf)

**Underlying Uniswap V3/V4 Engine Audits** (covering the core AMM, plugin system, farming, and periphery contracts that SwitchX is built on):
- Core: MixBytes, Bailsec (v1.2, v1.2.1)
- Farming & Base Plugin: MixBytes
- Entire protocol: Riley Holterhus, Paladin
- Custom Pools: Bailsec

### Prior Art

- Solidly: [solidlyexchange/solidly](https://github.com/solidlyexchange/solidly) — Original ve(3,3) implementation
- Velodrome: [velodrome-finance](https://velodrome.finance) — ve(3,3) on Optimism
- Aerodrome: [aerodrome-finance](https://aerodrome.finance) — ve(3,3) on Base
- Curve: [curve-dao-contracts](https://github.com/curvefi/curve-dao-contracts) — Original vote-escrowed governance
- Uniswap V4: [uniswap/v4-core](https://github.com/Uniswap/v4-core) — Concentrated liquidity with hook/plugin system
- Uniswap V3: [uniswap/v3-core](https://github.com/Uniswap/v3-core) — Concentrated liquidity AMM

---

*This document describes the SwitchX protocol as implemented in the smart contracts at the time of writing. Parameters are configurable through governance and may be adjusted post-launch. All values cited are from the deployed contract code and deployment scripts.*
