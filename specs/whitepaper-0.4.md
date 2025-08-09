# 1\. Abstract

USDM is a fully collateralised synthetic dollar issued on the Ethereum mainnet and represented on every connected rollup through LayerZero’s OFT standard. Collateral consists of USDC, USDT and USDS, each held in its vault under fixed hard caps and explicit circuit-breaker rules. Minting and redemption operate at deterministic price bands with all accounting settled on the canonical chain; bridged balances simply mirror that supply. Collateral is deployed to whitelisted L1 yield strategies (Morpho, Aave, Sky). The resulting yield is tracked on-chain and remitted pro rata to each rollup holding USDM, allowing those networks to offset incentive budgets while maintaining a single, predictable USD-pegged asset. The design minimises discretionary governance and removes cross-stable fragmentation without introducing algorithmic risk.

# 2\. Problem Statement

Stablecoin supply on Ethereum rollups is split across multiple contract variants: native USDC and bridged USDC.e, USDT, DAI, and newer algorithmic issues. Because each token has independent risk, every rollup must provision separate liquidity pools, oracle feeds, and lending parameters. This duplication creates inefficiencies:

* **Capital dilution** \- Depth is thin. A $1 million swap between “stable” tokens on most L2 DEXs can slip 20-40 bp during off-peak hours.  
* **Budget drain** \- Rollups fill the gap with perpetual incentives—often eight-figure token grants per year—yet spreads and peg gaps persist.  
* **Yield isolation** \- Collateral posted on L2 cannot directly access L1 yield streams, so networks fund rewards entirely from emissions instead of real earnings.

Together, fragmented liquidity and the absence of a yield return path drain capital efficiency, inflate subsidy budgets, and complicate composability across the entire L2 stack.

# 3\. Protocol Overview

USDM is a hard-pegged dollar stablecoin that consolidates USDC, USDT, and USDS into a single ERC-20 issued on the Ethereum mainnet (“canonical chain”). All supply operations—mint, redeem, collateral allocation, oracle checks—execute on this chain; every other network only holds a representation minted via LayerZero’s OFT. Supply coherence is therefore guaranteed by a single source of truth.

### 3.1 Collateral vaults

* Three isolated vault contracts, one per base asset.  
* Each vault has a fixed hard-cap percentage of total collateral (configurable but static during regular operation).  
* Deposits that breach a hard cap are rejected; redemptions from an over-cap vault are serviced first.

### 3.2 Mint / redeem state machine

* 1:1 mint or redeem when oracle price ≥ 0.998 and vault below cap.  
* Price between 0.995–0.998: mint at oracle rate; single-asset redeem allowed.  
* Price between 0.985–0.995: mint via on-chain swap into a healthy asset; redeem proportional only.  
* Price \< 0.985 or oracle failure: global pause for three hours.

Circuit-breaker conditions are enforced at both the vault and system levels, halting interaction rather than allowing gradual drift.

### 3.3 Yield handling

Collateral in each vault is deposited into whitelisted main-net strategies (Morpho, Aave, Sky). Net earnings follow a deterministic weekly pipeline:

1. Harvest: Strategy rewards are autocompounded into the vault itself.   
2. Mint-to-Yield: An equal amount of new USDM is minted on Ethereum; the realised yield fully backs the additional supply, so the backing ratio remains 1:1.  
3. Distribution: The net freshly minted USDM is bridged to every connected rollup in proportion to that chain’s share of total outstanding USDM.

A time-weighted balance model, which rewards chains for sustained liquidity rather than point-in-time snapshots, will be implemented in v2 (see Roadmap & Future Extensions).

### 3.4 Bridged representation

LayerZero messages lock canonical USDM and mint an equivalent OFT on the destination chain. Burn-and-redeem reverses the flow. Bridged tokens cannot influence supply or collateral composition—only the canonical chain can.

Taken together, these components create a deterministic, minimally governed stablecoin whose behaviour is identical across every connected rollup, while consolidating risk management on Ethereum.

# 4\. Architecture

MetaFi is built around isolated collateral vaults, deterministic mint/redeem logic, and controlled rebalancing infrastructure. USDM is issued on a canonical chain and deployed omnichain using LayerZero’s OFT standard. All supply and settlement logic occurs on the canonical chain to maintain system-wide consistency.

### 4.1 Collateral Vaults

* Three isolated ERC-4626 vault contracts hold USDC, USDT, and USDS, respectively.   
* A vault-manager sets and maintains a target collateral ratio (default: equal thirds). In v1 the vault manager role is held by Steakhouse Financial  
* Each vault has a fixed hard cap expressed as a share of total collateral (default: 50% per asset; changeable, but not dynamic).  
* Deposits that would push a vault above its cap are rejected; redemptions from an over-cap vault are serviced preferentially until the cap is cleared.  
* Vault contracts do not transfer assets across one another.

4.2 Mint / Redeem State Machine

* All mint-redeem calls execute on Ethereum mainnet.  
* Inputs: selected base asset, amount, current Chainlink oracle price p, vault utilisation u.  
* Transition table:

| Trigger Range | Mint/Redeeem |
| :---- | :---- |
| p ≥ 0.998 ∧ u \< cap | mint or redeem 1:1 |
| 0.995 ≤ p \< 0.998 | mint at oracle rate (discount),  allow single-asset redeem if u \< cap |
| 0.985 ≤ p \< 0.995 | mint only via on-chain swap into a healthy asset;  redeem proportional basket |
| p \< 0.985 ∨ oracle stale/deviant | enter paused state (§4.5) |

Price source and redundancy: *p* is the Chainlink median price for the asset. Its value is validated against a 30-minute TWAP from the primary Curve stableswap pool; if the deviation exceeds 0.5 % or the feed is stale for more than 10 minutes, the state machine jumps directly to the paused state defined in § 4.5

Swap paths are executed through pre-approved DEX routers with a maximum 30 bp slippage bound.

### 4.3 Bridging & Supply Coherence

* Canonical USDM balances are held in a supply manager contract.  
* LayerZero “send” locks canonical tokens and emits a message to mint the same amount on the destination chain; “receive” burns and unlocks.  
* Total bridged supply is the sum of OFT balances across destinations and always matches the locked amount on Ethereum, ensuring a single source of truth.  
* Non-canonical chains cannot mint, burn, or otherwise modify collateral state.

### 4.4 Rebalancing Engine

* The vault manager monitors live weights versus the configured target ratio.  
* When any vault reaches 95% of its hard cap or another falls below its target range, the manager may trigger a sealed-bid auction.  
* Bidders offer to exchange the over-represented asset for the under-represented one; the clearing price is the best discount (from oracle) that restores the target ratio inside tolerance.  
* Auctions are manual-triggered in v1 (multi-sig) and run off-chain for order collection, with on-chain settlement.  
* No cross-vault asset conversion occurs outside these events.

### 4.5 Circuit Breakers 

The entire mint/redeem interface is disabled for three hours if *any* of the following holds:

* Chainlink price \< 0.985.  
* Chainlink diverges from 30-min TWAP by \> 0.5 % for ≥ 10 min.  
* Oracle heartbeat \> 10 min.  
* More than one vault above hard-cap post-rebalancing attempt.

During the pause, collateral remains static.

# 5\. Economic Parameters

This section explains how value is generated and routed in **v1**, why key limits are chosen, and which constants governance can tune.

*Note – a dedicated “Safety Buffer” account is planned for v2; in v1 all surplus remains inside the originating vault (see Roadmap).*

### 5.1 Value Flow Overview

Sources

* Mint-price spreads – when the oracle price *p* is 0.995 – 0.998 (§ 4.2) the difference (1 – *p*) accrues in the vault.  
* Strategy yield – weekly harvest from Morpho, Aave and Sky positions.

Routing

mint-price spread ───────────► vault surplus (implicit over-collateralisation)

strategy yield  ───┐

                	 ├─► protocol fee α  → new USDM minted on ETH to treasury

            		     	│

                 		└─► (1 – α) → new USDM minted on ETH → bridged pro-rata to rollups

### 5.2 Parameter Rationale

| Parameter | Default | Justification |
| ----- | ----- | ----- |
| Hard-cap ceiling *C* | 45 % per vault | Limits blast radius if an asset fails. |
| Target ratio *wᵢ* | (0.333, 0.333, 0.333) | Neutral starting point; manager can tilt to higher-grade collateral (yield or security). |
| Rebalance tolerance *τ* | ±20 % of *wᵢ* | Avoids auction spam yet corrects real drift. |
| Harvest interval *E* | 7 days | Balances gas cost with timely distribution. |
| Protocol fee α | 10 % of harvest | Funds treasury, adjustable. |

*Note: Parameter setting will evolve in v2 (see Roadmap).*

# 6\. Risks 

This section enumerates the principal failure modes for USDM v1 and the specific controls in place to contain each one. The list is scoped to risks that can affect peg integrity or collateral solvency; commercial or regulatory risk is addressed elsewhere.

### 6.1 Smart-contract surface

Scope: Vault implementations (ERC-4626), mint/redeem state machine, rebalancing auction contracts, LayerZero OFT adapter.

Mitigations:

* Minimal upgrade surface: proxy admin can only change logic after a 48-hour time-lock.  
* Internal accounting uses 18-decimals fixed-point; no rounding shortcuts.  
* Continuous test-suite \> 95 % branch coverage; differential fuzzing between reference and production builds.

### 6.2 Oracle & price-feed dependency

* Primary feed: Chainlink USD price for each base asset.  
* Redundancy: 30-minute Curve stableswap TWAP; deviations \> 0.5 % or heartbeat \> 10 min route all transactions to the paused state (§ 4.5).  
* Fallback: Manual pause via authorised multisig if both feeds fail concurrently.

### 6.3 Cross-chain / bridge risk

Scope: LayerZero endpoint or relayer compromise could inflate bridged supply.

Mitigations:

* Canonical chain is single source of truth; non-canonical chains can only mint if canonical tokens are locked.  
* Daily supply-coherence check: script verifies bridged balances \= locked amount and emits alert on mismatch.  
* Rollups integrate USDM via OFT only—no custom wrappers.

### 6.4 Collateral de-peg

* Trigger: Base-asset price \< 0.985 USD.  
* Response: Global mint/redeem pause (3 h) plus forced proportional redemption once resumed; vault capped at 45 % limits system-wide shortfall.  
* Loop-prevention: Price-aware minting and proportional redemption block the classic “deposit cheap, redeem expensive” loop.

### 6.5 Strategy / yield risk

Scope: Morpho, Aave, Sky smart-contract failure or unexpected insolvency.

Controls:

* Strategies may be fully invested; small redemptions may require partial unwinds, increasing gas but not jeopardising solvency.  
* Weekly harvest pulls rewards and checks the underlying balance ≧ principal; mismatch \> 5 bp halts further deposits.  
* Any realised loss first consumes surplus collateral (mint-spread accruals) before touching base backing.

### 6.6 Liquidity-layer risk

* DEX slippage: Fallback minting path relies on Curve / Uniswap routes with 30 bp cap; if route fails transaction reverts.  
* Auction liquidity: Rebalancing auction can be cancelled if clearing discount \> 0.5 % (governance abort).

### 6.7 Operational & governance risk

* Key roles: Vault-manager multisig (5/9), Protocol-guardian (pause), Timelock (48 h).  
* Fail-safe: Any signatory can call the global-pause circuit breaker; unpause requires ≥ 60 % multisig plus 12-h delay.

### 6.8 Audit & formal-verification plan

* Code-complete freeze T – 6 weeks before main-net deploy.  
* External audits: Two firms, sequential:  
  1. \[Auditor 1\]  
  2. \[Auditor 2\]  
* Bug bounty: \[Platform\] programme, max payout \[xxx\], active from audit hand-off onward.  
* Formal spec Mint / redeem state machine translated to Scribble assertions; model-checked against invariant “backing ≥ supply”.

These layered controls aim to ensure that any single failure degrades the system in a predictable, containable way rather than cascading into insolvency or un-backed supply.

# 7\. Roadmap & Future Extensions

The v1 of MetaFi described in this whitepaper is expected to be fully functional in early Q4 of 2025\. It will allow for USDM to deliver the core benefits to rollups from day one. However the team has envisioned a lot of improvements and upgrades will improve scalability, and reduce friction and dependencies further, while maintaining first-in-class security.  This section lists the following planned releases and the corresponding improvements Timelines are indicative only.

### 7.1 v1.1 — Usability and Automation Upgrades 

Target timeline: Q1 2026

* **Automated auctions**: A controller contract opens a 15-minute commit–reveal window, settles bids on-chain, and re-weights vaults without multisig intervention.  
* **Gas-aware routing**: The mint / redeem state machine queries real-time L1 gas and relay fees; if “penalty \+ mint” beats “DEX swap \+ mint” by ≥ 5 bp it chooses the cheaper path automatically.  
* **Public monitoring**: A dashboard surfacing vault utilisation, oracle heartbeat, and cross-chain supply so rollups can track risk without running their own indexers.

### 7.2 v1.2 — Capital-efficiency Upgrades ** **

Target timeline: Q2 2026

* **Time-weighted yield distribution**: Instead of a single snapshot, a new contract integrates each rollup’s balance over time; harvests are distributed by “liquidity-days,” discouraging flash-loan farming.  
* **Dynamic allocation bands**: Once per week the vault-manager reads net APY for each base asset (Morpho \+ Aave \+ Sky) and writes a new target ratio to storage, shifting up to ±0.5 % per epoch toward the highest-yielding collateral. Auctions that follow will rebalance toward this freshly-set target without human input.  
* **Mint-proxy upgrade** — users can initiate USDM mints directly on integrated rollups. The proxy locks the base asset, relays the deposit to L1, and releases bridged USDM once the canonical mint is finalised. Collateral remains on Ethereum; only UX latency is improved.

### 7.3 v2.0 — Risk & Treasury Upgrades

Target timeline: Q3 2026

* **Dedicated Safety Buffer**: All mint-spread accruals plus 0.5 % of every harvest (β) flow into a new `SafetyBuffer` ERC-4626 vault.  
  *The buffer is non-redeemable:* funds can leave only to cover strategy losses, de-pegs, or to insure partner protocols. A public “buffer-to-TVL” ratio is streamed on-chain so integrators can monitor first-loss depth in real time.  
* **Adaptive redemption buffer**: `Adaptive redemption buffer` keeps **1–3 %** of each vault’s TVL in hot liquidity. Because v1 holds no dedicated idle slice, this buffer offers instant withdrawals without locking productive capital.   
* **Exponential penalty curve**: A utilisation-based formula of a penalty is introduced to reduce the rebalancing frequency:  
   `fee = Fmax · (u / cap)^n`, with *n* ≈ 4 and `Fmax` starting at 200 bp. Fees begin near zero at 90 % utilisation and rise smoothly toward the hard cap (mirrored near the hard floor on redemption). This creates a price signal that nudges markets back into balance without abrupt rejections and converts late-stage congestion into protocol revenue.

### 7.4 v2.x — Strategic Extensions

Target timeline: TBC

* **Direct issuer lanes**: The protocol establishes secured Signet / Tether Treasury accounts. When an on-chain de-peg of USDC or USDT ≥ 1 % persists for a defined window, a batch process can redeem up to N million tokens directly for fiat (or mint the reverse). Collateral then re-enters the vault via an institutional bridge. This “off-chain escape hatch” short-circuits extreme market stress and allows tighter hard-cap settings.  
* **L2 minting:** Light-weight `RollupVault` contracts on supported rollups accept *native* USDC-L2 or USDT-L2 and issue USDM instantly. Deposits are batch-bridged to L1 once per epoch (e.g., every 2 h), where the canonical vault mints backing USDM. A per-rollup mint ceiling (e.g., 0.5 % of TVL per epoch) and a zero-knowledge proof that the batch really arrived on L1 are used as safety guardrails. An emergency kill-switch pauses L2 minting if the bridge stalls.   
* **Solver / intent integration**: USDM becomes a first-class asset in intent networks such as CoW Protocol and Anoma. Solvers can atomically lock canonical USDM on Ethereum and unlock bridged USDM on any rollup, enabling zero-hop cross-chain swaps and rebalancing auctions that never touch a user-facing bridge. Integrations rely on the existing LayerZero proof path, so no new trust assumptions are introduced.

# 8\. Regulatory, Compliance & Disclaimers

### 8.1 Jurisdictional scope

USDM is issued and redeemed exclusively through Ethereum smart-contracts; it holds no fiat accounts and uses no off-chain custody. Upgrades and parameter changes pass through an on-chain DAO timelock (48 h). No single jurisdiction can compel minting, redemption, or asset seizure.

### 8.2 Collateral legal status

The underlying assets—USDC, USDT and USDS—are fiat-backed stablecoins issued by entities licensed as money transmitters or trusts in the United States or the EU. Because USDM remains 100 % collateralised by those tokens and introduces no algorithmic expansion, it is expected to inherit the same “payment-token” classification rather than securities status, provided the issuers maintain 1:1 reserves.

### 8.3 Yield classification

In v1 the only yield distribution is protocol-level: after the protocol fee α, new USDM is bridged to rollups, not directly to end-users. This structure is intended to fall outside the Howey test for investment contracts. Rollups that choose to on-distribute yield to users must evaluate their own securities and tax regimes.

### 8.4 Sanctions & AML controls

* A block-list module references the Chainalysis Sanctions Oracle every four hours; addresses on the list cannot call mint or redeem.  
* Front-end operators are expected to enforce additional IP and geo-fencing for restricted jurisdictions.  
* Plans for a zero-knowledge AML attestation layer are being researched (see § 7.4).

### 8.5 Legislative outlook

Draft U.S. legislation (e.g., the Genius Act) proposes disclosure and liquidity-window rules for fiat-backed stablecoins. Compliance for USDM depends on Circle, Tether and Stably meeting any new requirements. Governance retains the right to remove non-compliant collateral without altering USDM’s core contracts.

### 8.6 Legal wrapper & audit process

A Siwss-base foundation publishes protocol documentation, signs audit agreements, and licenses code under MIT. The foundation controls **no** upgrade keys; those remain with the on-chain governance timelock.

Audit cadence:

* \[Auditor 1\] — deep-spec review (complete before v1 launch)  
* \[Auditor 2\] — differential fuzzing (complete before v1 launch)  
* New audit round required for every major roadmap tier (§ 7), plus Immunefi bug-bounty programme with a $XXX k max payout.

### 8.7 Disclaimers

* **No solicitation**: This document is for information only; it does not constitute an offer to sell or the solicitation of an offer to buy any security or financial instrument.  
* **Forward-looking statements**: Roadmap items are targets, not guarantees. Delivery dates, features and priorities may change.  
* **Risk notice**: Smart-contract exploits, oracle failures, collateral de-pegs or adverse regulation could result in partial or total loss of value. Prospective integrators must conduct independent due diligence.  
* **Jurisdictional limits**: USDM may be unavailable in regions where its distribution or use would violate local law. Each participant is responsible for ensuring compliance with applicable regulations.