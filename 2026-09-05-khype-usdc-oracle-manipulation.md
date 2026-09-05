# Post-Mortem: KHYPE-USDC Pool Oracle Manipulation Exploit

**Date of incident:** September 5, 2026
**Affected contract:** LiquidCore KHYPE-USDC pool — `0x158F5919A3C65C201A02CB2fEE7421F7B78F3b1e` (HyperEVM)
**Total loss:** ~$155,000 (25,622.41 USDC + 1,473.58 kHYPE shortfall across 5 LP positions)
**Attacker profit:** 146,577.47 USDC (net, after fees), bridged to Ethereum and converted to DAI
**Status:** Pool halted. HYPE-KHYPE pool paused as a precaution (not exploited). All other pools unaffected.

---

## 1. Summary

On September 5, 2026 at 13:38 UTC, an attacker exploited the LiquidCore KHYPE-USDC pool using a flash-loan price manipulation attack against the pool's kHYPE price oracle (`KHypeOracle` at `0xB446554dC3f7AA203d78e827bE69c51EfFC15fcF`).

The oracle derived the kHYPE/HYPE exchange rate from the live spot prices (`slot0` / `globalState`) of four on-chain kHYPE/WHYPE DEX pools. It enforced a maximum 0.25% divergence *between* those four sources and capped the rate *above* at the Kinetiq staking NAV, but it had **no lower bound and no time-weighting**. By flash-loaning 150,000 kHYPE (~$13.1M) from Morpho Blue and dumping it into all four source pools simultaneously, the attacker moved every source in lockstep — passing the divergence check — and crashed the oracle's reported kHYPE price by 88.9% (from $87.53 to $9.73) within a single transaction.

The attacker then swapped USDC into the LiquidCore pool at the manipulated price, draining its entire kHYPE reserve, unwound the DEX positions, repaid the flash loan, and exited. The attack was repeated three more times over the following hour, harvesting kHYPE that arbitrage bots had restocked into the pool between attacks. The pool was halted at approximately 15:23 UTC.

## 2. Timeline (all times UTC, September 5, 2026)

| Time | Block | Event |
|---|---|---|
| ~07:00 | — | Attacker EOA receives pre-attack funding on Ethereum (1,427 DAI via bridge route) |
| 13:38:00 | 45098506 | **Attack 1** — 1,496.65 kHYPE drained for 15,428.58 USDC; net profit 113,620.29 USDC |
| 13:40:28 | 45098656 | 113,620.29 USDC bridged to Ethereum via Circle CCTP v2 (swapped to DAI on arrival) |
| 13:47:00 | 45099055 | **Attack 2** — 342.52 kHYPE drained for 3,653.47 USDC; profit 24,338.66 USDC |
| 13:47:52 | 45099107 | 24,338.66 USDC bridged via CCTP v2 |
| 13:59:00 | 45099787 | **Attack 3** — 96.48 kHYPE drained for 1,141.57 USDC; profit 5,284.54 USDC |
| 13:59:53 | 45099840 | 5,284.54 USDC bridged via secondary route |
| 14:32:00 | 45101800 | **Attack 4** — 71.26 kHYPE drained for 887.18 USDC; profit 3,333.98 USDC |
| 14:33:05 | 45101866 | 3,333.98 USDC bridged via secondary route |
| ~15:23 | — | Team halts KHYPE-USDC pool and pauses HYPE-KHYPE pool (`poolShutdown = true` deployed) |

## 3. Root cause

### 3.1 Oracle design

`KHypeOracle._computeRates()` computed the kHYPE/HYPE rate as the **median of four live DEX spot prices**:

- PRJX: `0xbe352daF66af94ccF2012a154a67DAEF95FAcB91`
- Nest (Algebra): `0xA83D60b1a9CA6Dd1d0D2d9275c700114F2F3a8d6`
- Ramses: `0x705D5dDa03D170384Eb43eB1aA692a6FC548306f`
- Hyperswap: `0x5Cbe810071DE393de35e574Fb2830E16dA794bab`

with two safety checks:

1. **Cross-source divergence:** revert if the max/min spread across the four pools exceeds 0.25% (`MAX_POOL_DEVIATION_BPS = 25`).
2. **NAV cap:** clamp the rate to the Kinetiq `StakingAccountant` redemption rate (`kHYPEToHYPE`) — an **upper bound only**.

The final kHYPE/USD price multiplied this rate by the HYPE/USDC spot price from the HyperCore precompile (not manipulable and not at issue).

### 3.2 The flaws

1. **No lower bound.** The NAV cap correctly reflects that kHYPE cannot trade *above* redemption value, but nothing constrained how far *below* NAV the reported price could fall. A kHYPE price 89% under redemption value is economically impossible in any real market, yet the oracle accepted it.
2. **The divergence check compares sources only to each other.** All four source pools share the same (thin) liquidity profile, so a single actor with a flash loan can move all four in lockstep and keep them within 0.25% of one another.
3. **No time-weighting.** The rate was pure same-block spot, so a manipulate-consume-restore sequence inside one transaction was invisible to any observer before or after the block.
4. **Amplifier in the pool:** `_calculateSwapAmount` clamps `amountOutBeforeFee` to available reserves instead of reverting when a swap requests more than the pool holds. This made "drain the entire reserve" the default outcome of any oversized swap at a manipulated price.

## 4. Attack mechanics (from `debug_traceTransaction` of attack 1)

1. Deploy two throwaway contracts (deployer + executor; both self-destruct at the end).
2. Flash-loan **150,000 kHYPE** from Morpho Blue (`0x68e37dE8d93d3496ae143F2E900490f6280C57cD`).
3. Read oracle: kHYPE/USD = **$87.527539**.
4. **Mint concentrated-liquidity positions** in all four source pools at price ranges *below* the market. This is the cost-minimizing trick: the attacker's dump lands mostly in their own liquidity, so unwinding recovers almost all of the flash-loaned kHYPE, leaving only DEX fees as cost.
5. **Dump ~85,000 kHYPE across all four pools.** Each pool's tick moved from −234 to ~+21,740 — a ~9x price move — while staying within 0.25% of each other, passing the divergence check.
6. Read oracle: kHYPE/USD = **$9.725282** (−88.9%). The NAV cap is upper-only; no check failed.
7. **Swap 15,428.58 USDC into the LiquidCore pool.** At the manipulated price, the computed output exceeded the pool's entire kHYPE reserve, so the reserve-clamp handed over everything: **1,496.65 kHYPE** (~$131,000 at the true price) minus a 3.81 kHYPE fee.
8. Burn/collect the concentrated LP positions, swap back through the four pools to restore prices, repay the flash loan.
9. Sell the surplus WHYPE (proceeds of the stolen kHYPE) through the Uniswap V3 WHYPE/USDC pool `0x6c9a33e3b592c0d65b3ba59355d5be0d38259285` (a third-party pool, priced normally — not exploited).
10. Net **113,620.29 USDC** to the attacker EOA. End-of-block oracle rate was fully restored (1.0234), leaving no visible price anomaly.

### Repeat attacks and the arb feedback loop

After attack 1 the pool held only USDC. Because swaps were still enabled and priced at the (now-restored) fair oracle rate, arbitrage bots sold kHYPE into the imbalanced pool for USDC — mechanically restocking the asset the attacker wanted. Attacks 2–4 repeated the manipulation and drained the restocked kHYPE each time. This loop also explains why the pool's USDC balance *fell* between attacks (80,424 → 41,327): the pool was buying kHYPE from arbitrageurs at fair value, and the attacker was then taking that kHYPE at ~1/9th of fair value.

## 5. Transactions and fund flow

### Exploit transactions (HyperEVM)

| # | Block | Tx hash | kHYPE taken | USDC paid | Profit (USDC) |
|---|---|---|---|---|---|
| 1 | 45098506 | `0x89b3d70198093d47ae4cd61409163279b44bc5e270c8ba707186b2710f0326a8` | 1,496.65 | 15,428.58 | 113,620.29 |
| 2 | 45099055 | `0x1e33661400bdb952a4d8c5379807d21535adfc3845dbb03b85c485b61cd1b50f` | 342.52 | 3,653.47 | 24,338.66 |
| 3 | 45099787 | `0x979833c79d1a45266b3a8ce0c9c1bd2d31710c9aa7326a084e3ada823cfe08a5` | 96.48 | 1,141.57 | 5,284.54 |
| 4 | 45101800 | `0xdcb97f11c2b5b08adb5a3a3afc5888036d2fc47bd1f357361fc2f6e8e723e9bb` | 71.26 | 887.18 | 3,333.98 |

### Attacker addresses

- **Primary EOA (HyperEVM and Ethereum):** `0xfeAf3B83dfF6DB0785DF190435FF669878A1667D`
- Throwaway exploit contracts (self-destructed): `0x84cf0b693bae57188751caa45b45ae753754a3c1`, `0xC77F2211DbE883067291E81158f9dc0b0893e86f` (attack 1); executors `0xc07e9bc7c730f44ac6ef46c3dff66b9da1e2dbc7` (2), `0x78a9e48ad6521b3e0e855071583e3bdf6226a2c2` (3), `0x2bc14d21a9451472763f187e88e5fcff2b988f3b` (4)

### Exit path

The two large tranches were burned via **Circle CCTP v2** (`TokenMessengerV2` `0x28b5a0e9c621a5badaa536219b3a228c8168cf5d`, HyperEVM domain 19 → Ethereum domain 0) using `depositForBurnWithHook`; the post-mint hook swapped the USDC to **DAI on arrival** (defeating any USDC freeze) and delivered it to the same EOA on Ethereum. The two small tranches used a secondary bridge route with the same USDC→DAI outcome.

| HyperEVM burn tx | Ethereum arrival tx | Amount |
|---|---|---|
| `0x0290611a00021c148ec87c309e5aef5fe7e53e72bf00cb5f22b60e90ca9f2016` | `0xde5327fa38f66476028207dd083ca25e95ee2cc7a67f8da5da78c76101081c0f` | 113,620.29 → DAI |
| `0x580adcade716e3ce6b3c559c53f9ecc967d293da8bc423d1aa9eb3b7219ad1bd` | `0x6a1bae6c22b9cca40df18c5fcb052bea86692514a74fa129b0761396cb436bc5` | 24,338.66 → DAI |
| `0xc2c5ab21a0d1bfb33e96cff77ff2de0e3b4b63c9eff48ff5c3a4470cdb1e1402` | `0xf3da82fbf646cf6f5ffed83e966df45bb88d3d563cd7dc93a0bc7bc836bb69e6` | 5,284.54 → 5,283.93 DAI |
| `0xb96261321622e067898b13121e093f8b51bcba45ce7679a6cf0a0807ed0d09b2` | `0x4a1807cbbb4dd2d4d4578cfb76a52b9b096e094f21923b822884de850172caa3` | 3,333.98 → 3,333.55 DAI |

Total received on Ethereum: **146,576.43 DAI**. The EOA also received 1,427 DAI on Ethereum ~6.5 hours before the attack (funding trail: `0x5aefd4a9c14ba04ce1115d6bd4d64175d08052fb793216405db8cedb0ea83b03`).

## 6. Impact on LPs

Pool state before attack 1 (block 45098505): 64,995.42 USDC + 1,500.46 kHYPE (~$196,330).
Pool state after halt: 39,372.90 USDC + 22.43 kHYPE (~$41,340).

Five LP positions were affected. Per-holder shortfall (pre-hack entitlement incl. unclaimed fees, minus current withdrawable value):

| Holder | Share | USDC shortfall | kHYPE shortfall |
|---|---|---|---|
| `0xbfb849931dc8cfa08a690c435fb4e4f47906ec8d` | 52.15% | 13,362.88 | 768.518 |
| `0x18774e536505f7f78f0597ab371a385511fffafc` | 20.32% | 5,206.67 | 299.443 |
| `0x428a22eede833fcb0a0c2d0e16a760a92f4a6112` | 12.25% | 3,138.65 | 180.508 |
| `0x7763bb0de57d485166cac224174ce93d1513886a` | 12.01% | 3,078.18 | 177.031 |
| `0x4263fa4976f2838181a295b6f98a9ccd7962be99` | 3.26% | 836.03 | 48.081 |
| **Total** | | **25,622.41** | **1,473.58** |

Withdrawals from the pool remain enabled; only deposits and swaps are halted.

## 7. Remediation

Completed:

- KHYPE-USDC pool halted (`poolShutdown = true`); HYPE-KHYPE pool (same oracle) paused as a precaution.
- Full on-chain trace and fund-flow analysis completed (this document).

Required before kHYPE pools can reopen:

1. **NAV-anchored lower bound in `KHypeOracle`.** Reject or clamp any DEX-derived rate more than a small tolerance (e.g., 1–2%) below the Kinetiq `StakingAccountant` redemption rate, mirroring the existing upper cap. This single change makes the observed attack impossible: the manipulated rate (−89%) would have been clamped to ~NAV.
2. **TWAP or multi-block observation.** Same-block spot from manipulable AMM pools should never be the sole price input for swap execution.
3. **Revert instead of clamping output to reserves** in `_calculateSwapAmount`, so an oversized swap fails rather than silently emptying the pool.
4. **Divergence check against an external anchor**, not only between correlated sources: the four source pools share liquidity dynamics and can be moved together; comparing them to NAV × HYPE spot catches lockstep manipulation.
5. Review all other oracle-priced pools for the same pattern (spot-only sources, one-sided bounds, reserve clamping) before considering the incident closed.

## 8. Recovery actions

- Attacker EOA, exploit contracts, and all bridge transactions identified (Section 5); reports to be filed with chain-analytics providers and relevant law enforcement.
- Circle notified of the CCTP exit path; the immediate USDC→DAI conversion on arrival limits freeze options for the bridged funds, but the pre-attack funding trail on Ethereum is actionable for attribution.
- Affected LPs have been identified and will be contacted directly regarding reimbursement.
