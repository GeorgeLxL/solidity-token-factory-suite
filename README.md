# Ktv2 Smart Contract Suite

A set of EVM / Solidity smart contracts implementing a **staking, donation, and
committee-governed reward protocol**, together with a deployment factory, a
Uniswap-based price oracle, and an ownership timelock.

> ⚠️ **Status:** Reference / educational code. These contracts are **unaudited**
> and have **not** been deployed to mainnet. Do not use in production without a
> full independent security audit and a comprehensive test suite.

---

## Contracts

| Contract | Solidity | Purpose |
|---|---|---|
| [`Ktv2.sol`](Ktv2.sol) | `0.8.16` | Core protocol: staking, donations, price-linked burning, and an operator-committee (OC) reward mechanism with on-chain consensus voting. |
| [`Ktv2Factory.sol`](Ktv2Factory.sol) | `0.8.16` | Factory that deploys new `Ktv2` instances and transfers ownership to the caller. |
| [`Ktv2OwnershipTimelock.sol`](Ktv2OwnershipTimelock.sol) | `0.8.20` | Time-bound freeze of a `Ktv2` contract's ownership, with register / freeze / extend / restore. |
| [`TokenPrice.sol`](TokenPrice.sol) | `0.7.6` | Price oracle reading spot price from Uniswap **V3** (`slot0` sqrtPrice) and **V2** (reserves). |

### `Ktv2` — core protocol

- **Staking** — users `stake` / `withdraw` an ERC-20 token tracked per-account and in aggregate.
- **Donations (`give`)** — ETH donations are forwarded to a destination address; a price-linked portion of staked tokens is burned via a tiered burn curve (driven by the live token price from `TokenPrice`).
- **Operator Committee (OC)** — a set of authorized reward nodes that:
  - distribute epoch-based ETH rewards (`rwd`) gated by a configurable consensus threshold;
  - `vote` / `resetVote` on reward destinations per epoch;
  - `voteToAdd` / `voteToRemove` committee members, with membership changes auto-executed once a majority is reached;
  - accrue and withdraw OC fees, with fee migration across epochs.
- **Owner controls** — epoch interval, consensus requirement, OC fee, burn parameters, donation percentage, pool, V2/V3 mode, and emergency rescue of non-protocol tokens.

### `Ktv2OwnershipTimelock` — ownership freeze

A standalone timelock that temporarily locks ownership of a specific `Ktv2`
contract for a chosen duration (1 hour – 365 days):

1. `registerOriginalOwner()` — the current Ktv2 owner registers.
2. Transfer Ktv2 ownership to the timelock.
3. `freezeOwnership(duration)` — starts the lock.
4. `extendFreeze(additionalDuration)` — optionally extends, capped at the max duration.
5. `restoreOwnership()` — after expiry, the original owner reclaims ownership; lock state is cleared.

Uses OpenZeppelin `ReentrancyGuard` and follows checks-effects-interactions
(state is cleared before the external `transferOwnership` call). View helpers
(`timeUntilRestore`, `canRestore`, `getLockStatus`) report lock status.

---

## Architecture

```
                 ┌────────────────┐
   deploys       │  Ktv2Factory   │
  ──────────────▶│   create()     │
                 └───────┬────────┘
                         │ new Ktv2(...)
                         ▼
        ┌─────────────────────────────────┐        price()/priceV2()
        │              Ktv2               │ ───────────────────────────▶ ┌──────────────┐
        │  staking · donations · burning  │                              │  TokenPrice  │
        │  OC consensus rewards & voting  │ ◀─── Uniswap V2/V3 pools ──── │  (oracle)    │
        └───────────────┬─────────────────┘                              └──────────────┘
                        │ transferOwnership()
                        ▼
        ┌─────────────────────────────────┐
        │     Ktv2OwnershipTimelock       │
        │  freeze / extend / restore      │
        └─────────────────────────────────┘
```

---

## Dependencies

- **OpenZeppelin Contracts `4.x`** — `Ownable`, `IERC20`, `security/ReentrancyGuard`
  (the `security/` import path is v4; in v5 it moved to `utils/`).
- **Uniswap** — `@uniswap/v3-core` (`IUniswapV3Pool`, `FullMath`) and
  `@uniswap/v2-core` (`IUniswapV2Pair`) for `TokenPrice.sol`.

> Note: the suite spans **three compiler versions** (`0.7.6`, `0.8.16`, `0.8.20`).
> A build setup (Hardhat or Foundry) must configure all three and resolve the
> OpenZeppelin and Uniswap dependencies.

---

## Build & Test (not yet included)

No toolchain is committed in this repo yet. A recommended setup:

- **Foundry** for unit + fuzz/invariant testing of the contracts, or
- **Hardhat** (+ `@nomicfoundation/hardhat-toolbox`) for JS/TS tests and coverage.

Suggested next steps before any deployment:

- [ ] Add a build config covering solc `0.7.6` / `0.8.16` / `0.8.20`.
- [ ] Unit + invariant/fuzz tests (staking, donation burn curve, OC consensus, timelock lifecycle).
- [ ] Static analysis (Slither) and an independent security audit.
- [ ] CI pipeline (compile + test + analysis on every PR).

---

## License

MIT — see SPDX headers in each source file.
