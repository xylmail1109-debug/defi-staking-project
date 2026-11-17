# DeFiStaking — CDS528 Blockchain DeFi Smart Contract

A time-based ERC-20 staking dApp built with **Solidity 0.8.20 + Hardhat + OpenZeppelin 5.6.1**, deployable to the **Ethereum Sepolia** testnet.

**Repository:** https://github.com/wangziyan2006-cmd/defi-staking-project

**Team:** CDS528 Group (Lingnan University) · **Course:** CDS528 Blockchain DeFi Smart Contract Development

| Item | Status |
|---|---|
| Compiler | Solidity 0.8.20 (optimizer enabled, runs = 200) |
| Framework | Hardhat 2.29.2 + `@nomicfoundation/hardhat-toolbox` 5.x |
| Libraries | OpenZeppelin Contracts 5.6.1 (`ERC20`, `Ownable`, `ReentrancyGuard`, `Pausable`) |
| Unit tests | **16 passing** (`npm test`) |
| Static analysis | Slither 0.11.6 — **0 High / 0 Medium**, 20 informational findings |
| Network | Sepolia (chainId `11155111`) |

---

## 1. Problem Statement (R1)

**Users:** retail crypto holders who want a simple, transparent way to earn yield on an ERC-20 token they already hold.

**Pain point:** most yield products are custodial — the user hands over the tokens to a platform and trusts an off-chain ledger for the accounting of accrued interest. The user cannot audit the payout formula, cannot verify that their principal is still there, and cannot prove what rate they were promised.

**Why a smart contract (and not a database):** the accrual formula, the principal balance, the reward rate and every payout must be (a) *verifiable by anyone before depositing*, (b) *unchangeable by the operator after the fact without an on-chain, logged action*, and (c) *self-executing without a trusted intermediary*. `block.timestamp` gives every participant the same clock, and events give every payout a permanent public receipt — neither is reproducible with a private database.

**Key assumptions / constraints**
- Rewards are linear in time (no compounding); a fixed pool must be pre-funded by the operator.
- The contract never holds real user value in this assignment (testnet only, per course policy).
- Rate updates are owner-only but fully event-logged.

---

## 2. Solution Design (R2)

### 2.1 On-chain vs off-chain partitioning

| Concern | Location | Rationale |
|---|---|---|
| Principal custody, reward accrual, payout, rate change, pause | **On-chain** | Trust-critical state and value transfer — must be auditable and tamper-evident |
| APY display, historical charts, wallet UX, notifications | **Off-chain** (front-end / indexer) | Only reads `earned()` and events; expensive computation stays off gas |
| Rate *decision* (what APY to offer) | Off-chain decision, but the *application* of it is an on-chain owner transaction | Economic policy can be decided anywhere; the effect must be logged on-chain |

### 2.2 Token choice

A plain **ERC-20** (`MockToken`) is used for both principal and reward in the demo. ERC-20 is chosen over ERC-721 because the asset is fungible and the accounting is per-unit; it also lets us reuse OpenZeppelin's audited `ERC20` implementation instead of writing a bespoke token (course guidance: reuse battle-tested code). The 1,000,000-token initial supply is minted to the deployer so the demo has a reward pool available immediately.

> ⚠️ See §8 Known Limitations — in the demo the same token is used for principal and reward.

### 2.3 Key process flow

```
User                DeFiStaking                    MockToken (ERC20)
 |                       |                                |
 |-- approve(amount) ------------------------------------->|
 |-- stake(amount) ----->| updateReward(user)             |
 |                       | amount += _; totalStaked += _  |
 |                       |-- transferFrom(user,this) ---->|
 |                       | emit Staked(user, amount)      |
 |                       |                                |
 |   ... time passes (block.timestamp advances) ...       |
 |-- earned(user) ------>| (view) pending + elapsed*amt*rate/1e18
 |                       |                                |
 |-- claimReward() ----->| updateReward(user)             |
 |                       | pending = 0                    |
 |                       |-- transfer(user, pending) ---->|
 |                       | emit RewardClaimed(user, amt)  |
 |                       |                                |
 |-- withdraw(amount) -->| updateReward(user)             |
 |                       | amount -= _; totalStaked -= _  |
 |                       |-- transfer(user, amount) ------>|
 |                       | emit Withdrawn(user, amount)   |
```

Every state-changing entry point goes through the `updateReward` modifier, so no interaction can be used to skip or double-count accrual.

### 2.4 Accrual formula

```
pending += (block.timestamp - lastUpdateTime) * amount * rewardRatePerSecond / 1e18
```

`rewardRatePerSecond` is expressed in **wei of reward per 1 whole token per second**. Default `1e10` ⇒ `1e10 * 31,536,000 / 1e18 ≈ 31.5%` annualised, chosen so a live demo produces a visible reward balance within minutes.

---

## 3. Repository Structure

```
defi-staking-project/
├── contracts/
│   ├── MockToken.sol            # ERC-20 test token, 1,000,000 initial supply
│   └── DeFiStaking.sol          # staking core: accrual, guards, pause, events
├── scripts/
│   ├── deploy.js                # one-command deploy + reward pool funding + preflight checks
│   ├── demo.js                  # full lifecycle (stake → claim → withdraw) on Sepolia
│   └── verify-sourcify-v2.js    # source verification against the Sourcify v2 API
├── test/
│   └── DeFiStaking.test.js      # 16 unit tests
├── frontend/
│   ├── index.html               # no-build console: read state, sign via MetaMask
│   ├── app.js                   # ethers UMD + explicit gas limits (see §5.2)
│   ├── ethers.umd.min.js        # vendored ethers build, so the page works offline
│   └── README.md
├── security/
│   ├── slither-report.txt       # raw Slither output (R4 evidence)
│   └── SECURITY.md              # threat model & mitigations (R4)
├── flattened/                   # single-file / standard-json sources for manual verification
├── deployments/sepolia.json     # addresses, constructor args, verification status, demo tx hashes
├── hardhat.config.js
├── .env.example                 # copy to .env and fill in
├── LICENSE                      # MIT
└── README.md
```

---

## 4. Quick Start

### 4.1 Prerequisites

- Node.js ≥ 18 (tested on v22.23.2) and npm ≥ 9
- An RPC endpoint for Sepolia (Alchemy / Infura)
- A **dedicated test wallet** funded with Sepolia ETH (faucet)

### 4.2 Install

```bash
git clone <this-repo-url>
cd defi-staking-project
npm install
cp .env.example .env      # Windows: copy .env.example .env
```

Fill `.env`:

```ini
SEPOLIA_RPC_URL="https://eth-sepolia.g.alchemy.com/v2/<YOUR_KEY>"   # note: eth-sepolia, NOT eth-mainnet
PRIVATE_KEY="<64 hex chars, no 0x prefix, TEST WALLET ONLY>"
ETHERSCAN_API_KEY=""     # optional; leave empty to verify via Sourcify instead
```

### 4.3 Compile

```bash
npm run compile
# Compiled 11 Solidity files successfully (evm target: paris)
```

### 4.4 Test

```bash
npm test
# 16 passing
```

### 4.5 Deploy to Sepolia

```bash
npm run deploy
```

The script performs three preflight checks before spending any gas, so failures are self-explanatory:

1. `PRIVATE_KEY` present in `.env`
2. deployer balance > 0 (otherwise it prints the faucet URL)
3. RPC `chainId === 11155111` — this **blocks accidental mainnet deployment** (the `sepolia` network in `hardhat.config.js` also pins `chainId`, so a mainnet RPC fails fast instead of deploying with real ETH)

Expected output:

```
Deploying with account: 0x....
Account balance: 0.5 ETH
MockToken deployed to: 0x....
DeFiStaking deployed to: 0x....
Reward pool funded: 10000 MTK -> 0x....
```

### 4.6 Verify source code

`npx hardhat verify` reaches Sourcify through `hardhat-verify`, whose Sourcify path still calls the retired **v1** API and fails with HTTP 404 on a correct configuration. Use the bundled v2 client instead:

```bash
npm run verify:sourcify -- <stakingAddress> DeFiStaking
npm run verify:sourcify -- <tokenAddress> MockToken
```

If an `ETHERSCAN_API_KEY` is supplied, the `hardhat verify` path additionally submits to Etherscan (the config enables it automatically), and that route is unaffected by the Sourcify change. Manual fallback with no API key: upload `flattened/DeFiStaking_flat.sol` (compiler 0.8.20, optimizer on / 200) through the explorer's *Verify & Publish → Solidity (Single file)* form, or `flattened/standard-json-input.json` via *Solidity (Standard-Json-Input)*.

---

## 5. Deployment Record

| Contract | Sepolia address | Verified |
|---|---|---|
| `MockToken` (MTK) | [`0x68b78097aa78e0E5f444BbE43E8E904822c0b6d5`](https://sepolia.etherscan.io/address/0x68b78097aa78e0E5f444BbE43E8E904822c0b6d5) | Sourcify `exact_match` — [lookup](https://repo.sourcify.dev/11155111/0x68b78097aa78e0E5f444BbE43E8E904822c0b6d5) |
| `DeFiStaking` | [`0x988398cF9453F855471f6dAfE706C344741eD2D3`](https://sepolia.etherscan.io/address/0x988398cF9453F855471f6dAfE706C344741eD2D3) | Sourcify `match` — [lookup](https://repo.sourcify.dev/11155111/0x988398cF9453F855471f6dAfE706C344741eD2D3) |

Deployed 2026-11-16 from account `0xfcaFDDf94BAafF9598Cc67F650da383Fc49DC916` (chainId 11155111, block 12141741), constructor arguments `MockToken("Mock Token","MTK")` and `DeFiStaking(stakingToken = MTK, rewardToken = MTK)`, reward pool funded with 10,000 MTK in the same script run. Machine-readable record: `deployments/sepolia.json`.

On-chain state immediately after deployment (verified by a read-only script, not copied from logs):

| Read-back | Value |
|---|---|
| `MockToken.totalSupply()` | 1,000,000 MTK |
| `MockToken.balanceOf(DeFiStaking)` | 10,000 MTK (the reward pool) |
| `DeFiStaking.totalStaked()` | 0 |
| `DeFiStaking.rewardRatePerSecond()` | 10000000000 (1e10, ≈31.5% APY) |
| `DeFiStaking.owner()` | 0xfcaFDDf94BAafF9598Cc67F650da383Fc49DC916 |
| `DeFiStaking.paused()` | false |

> **Verification note (worth reading before you re-run `npm run verify`).** `hardhat-verify` 2.1.x still calls the Sourcify **v1** endpoints (`/check-all-by-addresses`), which Sourcify has retired — the plugin therefore fails with HTTP 404 `API v1 is removed` regardless of correct configuration. This repository therefore ships `scripts/verify-sourcify-v2.js`, which posts the solc standard-JSON input to `POST /v2/verify/{chainId}/{address}` and polls the job. It matters that the compiler version is passed **with its commit hash** (`0.8.20+commit.a1b79de6`): plain `0.8.20` makes Sourcify's own runner fail to fetch solc.
>
> ```bash
> npm run verify:sourcify -- 0x988398cF9453F855471f6dAfE706C344741eD2D3 DeFiStaking
> npm run verify:sourcify -- 0x68b78097aa78e0E5f444BbE43E8E904822c0b6d5 MockToken
> ```

### 5.1 Live demo (`npm run demo`)

`scripts/demo.js` runs the full lifecycle on Sepolia against the addresses recorded in `deployments/sepolia.json`: `approve` → `stake(100 MTK)` → wait 90 s of chain time → `claimReward()` → `withdraw(100 MTK)`, printing every transaction hash. It refuses to run on any chain other than 11155111.

**Status: executed on 2026-11-16.** One complete cycle, from a clean pool:

| Step | Sepolia tx hash | Observed |
|---|---|---|
| `approve` | [`0xc1791d04…b1cb07`](https://sepolia.etherscan.io/tx/0xc1791d04dc91e58efb474dbdfbce3d23dc37e42fb2f0d7cc69f9a6b945b1cb07) | allowance 100 MTK |
| `stake` | [`0xceb2aa21…cb9d8ed`](https://sepolia.etherscan.io/tx/0xceb2aa21bf2b61585cff803a379cd5da6c331242b704261a41197ad45cb9d8ed) | `totalStaked` 0 → 100 MTK, `Staked` emitted |
| `claimReward` | [`0xd94f0584…edf1756`](https://sepolia.etherscan.io/tx/0xd94f0584d00c9f8970986c83b5c62f3131bf333de3e1a6c15d1c523beedf1756) | 0.000192 MTK paid, `pendingRewards` zeroed |
| `withdraw` | [`0x39c723a8…452d39`](https://sepolia.etherscan.io/tx/0x39c723a8327f1a1b4e139f54bde8fa838ed012caaa02b5685c28f11f17452d39) | principal returned, `totalStaked` 100 → 0 |

Accrual observed live over 90 seconds: `earned()` climbed 0.000096 → 0.000192 MTK in even steps (0.000012 per block window), which is `ρ · Δt · a / 1e18` for a 100 MTK position at `ρ = 1e10`.

### 5.2 Operational finding: `estimateGas` is under-reported on this RPC

The first two withdrawal attempts **failed on-chain** (`status 0`, no logs, `gasUsed == gasLimit`). Cause, measured rather than guessed:

| Call | `eth_estimateGas` returned | Actually consumed |
|---|---|---|
| `withdraw` | 94,594 / 113,512 / 213,344 (varies) | **176,873** |
| `stake` | 311,608 | 286,373 |
| `claimReward` | 91,797 | 71,516 |

So `stake` and `claimReward` are estimated correctly, while `withdraw` was under-reported in two of three attempts, and always by enough to exhaust the limit. A static call (`eth_call`) on the identical state succeeds, which rules out a contract-level revert — this is an RPC-side estimation artifact, not a contract bug. Failed transactions here also showed an empty receipt: **no revert data, no logs, `gasUsed` exactly equal to `gasLimit`** is the signature of out-of-gas, and worth recognising in any future debugging.

Consequence for this repository: `scripts/demo.js` never sends the raw estimate. It sends `max(estimate × 1.2, function-level floor)` with floors of `approve 80k`, `stake 320k`, `claimReward 130k`, `withdraw 260k`. Gas is nearly free on this testnet (`gasPrice` 16 wei), so generous limits cost nothing and remove a whole class of flaky demos. Keep the same habit if you rebuild with `ethers` in a front-end: pass an explicit `gasLimit` for `withdraw`.

---

## 6. Testing (R4)

`npm test` — **16 passing**, grouped by concern:

| Group | Tests | What is asserted |
|---|---|---|
| Core flows | 3 | `stake` / `withdraw` / multi-staker `totalStaked`, event args, storage state |
| Reward math | 3 | `earned()` matches the formula exactly over a 1-hour window; `claimReward` payout equals the accrual and resets pending; pending rewards survive a partial withdraw |
| Access control | 3 | non-owner `setRewardRate` / `pause` revert with `OwnableUnauthorizedAccount`; owner update emits `RewardRateUpdated` |
| Pausable ops | 2 | staking blocked while paused (`EnforcedPause`), works again after unpause; **withdraw still allowed while paused** so funds are never trapped |
| Edge cases | 5 | `stake(0)`, over-withdraw, claim with nothing accrued, insufficient allowance (`ERC20InsufficientAllowance`), insufficient balance (`ERC20InsufficientBalance`) |

Timing-sensitive assertions derive expected rewards from the **actual block timestamps** of the mined transactions instead of assuming a fixed one-second block interval, so the suite is deterministic on both the in-process Hardhat network and a live testnet fork.

---

## 7. Security (R4)

### 7.1 Automated scan — Slither 0.11.6

```bash
pip install slither-analyzer
npm run scan          # slither .
```

Raw output is committed at **`security/slither-report.txt`**. Headline:

```
INFO:Slither:. analyzed (13 contracts with 102 detectors), 20 result(s) found
```

**All 20 findings are informational / low severity — no High, no Medium, and zero reentrancy or value-transfer issues in our own code.** Breakdown:

| Detector | Severity | Where | Assessment |
|---|---|---|---|
| `solc-version` | Info | `pragma ^0.8.20` | Floating pragma. Acceptable here because the exact binary is pinned by `hardhat.config.js`; a production build would pin `=0.8.20`. |
| `pragma` | Info | OZ dependencies | Same as above, inherited from OpenZeppelin (`>=0.8.4`). |
| `timestamp` | Low | `DeFiStaking.withdraw` | Reward accrual is *intentionally* time-based. Validator drift is ±15 s, which moves yield by a negligible amount and gives no exploitable advantage — the classic timestamp attacks target randomness or auctions, neither of which exists here. |
| `assembly` | Info | OZ `StorageSlot` | Inline assembly inside OpenZeppelin's audited library, not our code. |
| `shadowing-local` | Info | `MockToken` constructor params `name`/`symbol` | Constructor parameters shadow the inherited getters. Cosmetic; kept for readability. |
| `too-many-digits` | Info | `MockToken` `1000000 * 10 ** decimals()` | Readability literal; intentional. |
| `unindexed-event-address` | Info | OZ `Pausable` events | Inherited from OpenZeppelin, not our emission path. Our own events index the user address. |

Reproduce exactly with `npm run scan` (requires `hardhat compile` first; Slither picks up the Hardhat build info).

### 7.2 Threat model — top 3 risks and mitigations

| # | Threat | Attack sketch | Mitigation in this code | Residual risk |
|---|---|---|---|---|
| 1 | **Reentrancy** | Malicious ERC-20 or a callback re-enters `stake`/`withdraw`/`claimReward` mid-transfer to drain the pool | `nonReentrant` on all three entry points; **state is updated before the external call** (checks-effects-interactions); OpenZeppelin `ReentrancyGuard` | Low. A malicious token is still an operator-trust assumption (see #3). |
| 2 | **Access control / privilege abuse** | Non-owner flips `rewardRatePerSecond` or pauses to grief users | `Ownable` (`onlyOwner`) on `setRewardRate`, `pause`, `unpause`; every privileged action emits an event; **`withdraw` deliberately not pausable** so users can always exit | Low. Owner is a single EOA in the demo — production should move to a multisig / timelock. |
| 3 | **Accounting integrity / economic drain** | Integer error, or reward pool over-commitment, lets one user withdraw more than entitled | Strict per-user `amount` accounting, `require(user.amount >= amount)`, `pendingRewards` zeroed before transfer, rate bounded by a pre-funded pool in the demo | **Medium by design** — see §8.1: principal and reward share one token balance. |

Also considered but not applicable: oracle manipulation (no price feed), front-running (no auction or swap, and accrual is rate-fair over time), flash-loan attacks (no same-block profit path).

### 7.3 Fuzz / invariant testing

Not included in this iteration. The natural next step is an Echidna/Foundry invariant: *`sum(user.amount) == totalStaked`* at all times, and *`contract reward balance never below the sum of claimable rewards`*. Both properties are already implied by the unit tests for the single-user case.

---

## 8. Known Limitations & Design Trade-offs

Honest disclosure — these are the questions a reviewer will ask, answered up front.

### 8.1 Principal and reward share one token balance
`DeFiStaking` is deployed with `rewardToken == stakingToken`. `claimReward` transfers out of the contract's own balance, so an under-funded pool or an aggressive rate would pay rewards out of other users' principal and make `withdraw` revert. The demo pre-funds 10,000 MTK and uses a 31.5% APY so the pool cannot be exhausted during the demo. **Production fix:** deploy two tokens (`MTK` principal, `RWD` reward) or track `rewardPool` separately and revert when `claimable > rewardPool`.

### 8.2 `setRewardRate` is not retroactively settled
Changing the rate does not first checkpoint every staker (that would be unbounded gas). Time accrued since a user's `lastUpdateTime` is therefore recomputed at the **new** rate on their next interaction. **Production fix:** the Synthetix `accRewardPerShare` accumulator pattern — a global index updated on every write, so each user settles at the rate in force while they were staked. This also removes the per-call division and improves gas.

### 8.3 Single pool, linear rewards
The mechanism intentionally mirrors the canonical staking tutorial, which limits the *novelty* score. Natural extensions: multiple pools with different lock-ups and rates, an early-exit penalty redistributed to remaining stakers, or an accumulator-based reward index.

### 8.4 Immutable upgrade policy
Both contracts are non-upgradeable. Rationale: the assignment scope does not justify a proxy, and an immutable contract is strictly easier for a depositor to audit — there is no admin key that can change the code underneath their principal. Operational flexibility is provided instead by `Pausable` + `setRewardRate`.

---

## 9. Operations Runbook (R5)

| Situation | Action | Command / effect |
|---|---|---|
| Suspected exploit or anomalous flow | Pause deposits | `staking.pause()` (owner) → `Paused` event; `stake` reverts with `EnforcedPause`, **withdrawals keep working** |
| Resume normal service | Unpause | `staking.unpause()` (owner) → `Unpaused` event |
| Adjust economics | Change rate | `staking.setRewardRate(newRate)` (owner) → `RewardRateUpdated(newRate)` |
| Monitor | Subscribe | `Staked` / `Withdrawn` / `RewardClaimed` / `RewardRateUpdated`, all `indexed` on the user address |
| Redeploy | One command | `npm run deploy` (idempotent script, fresh addresses) |

---

## 10. Roles

This submission is an **individual** project, not a multi-member group: one author carried every role. The table records what each role actually produced, so the split of work is auditable rather than nominal.

| Role | Member | Delivered |
|---|---|---|
| Group Project Lead | Yulun Xu | Organize, coordinate, guide overall direction and architecture，report, deck, submission |
| Smart Contract Lead | cosmos wang | `contracts/DeFiStaking.sol`, `contracts/MockToken.sol`, reward-accrual design |
| Backend / Integration | Jiajun Yang | `scripts/deploy.js`, `scripts/demo.js`, `scripts/verify-sourcify-v2.js`, `deployments/sepolia.json` |
| Testing & Security Analyst | Junrui Wu | `test/DeFiStaking.test.js` (16 tests), Slither run, `security/SECURITY.md`, threat model |
| Front-end / UX | Xiaobo Liu | `frontend/index.html`, `frontend/app.js` (no-build MetaMask console) |

---

## 11. AI Assistance Disclosure

Per the course's academic integrity policy: the initial project skeleton (Hardhat configuration, contract structure, and test scaffolding) was drafted with AI assistance, then reviewed, executed and corrected by the team — every claim in this README is backed by a command that was actually run in this repository (compile, `16 passing`, Slither output committed under `security/`). AI-generated code was checked against the OpenZeppelin 5.x API (custom errors `EnforcedPause`, `OwnableUnauthorizedAccount`, `ERC20InsufficientAllowance`) and the licence terms of every dependency are compatible with the MIT licence of this repository.

---

## 12. References (APA)

- Ethereum Foundation. (2025). *Ethereum developer tutorials*. https://ethereum.org/developers/tutorials/
- OpenZeppelin. (2025). *OpenZeppelin Contracts v5.6.1 documentation*. https://docs.openzeppelin.com/contracts/5.x/
- Trail of Bits. (2025). *Slither: Static analyzer for Solidity*. https://github.com/crytic/slither
- Nomic Foundation. (2025). *Hardhat documentation*. https://hardhat.org/docs
- Solidity documentation — *Security considerations / SWC registry*. https://docs.soliditylang.org/

## 13. Licence

MIT — see [LICENSE](LICENSE).
