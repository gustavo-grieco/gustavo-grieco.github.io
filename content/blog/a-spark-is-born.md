+++
title = "Adventures in Automated Smart Contract Testing: A Spark Is Born"
date = "2026-07-03"
description = "On already-audited code, prove the core math first and fuzz the rest: verifying a DeFi contract's exact conversion properties against the real bytecode with hevm's arithmetic abstraction, and proving weaker but still-useful properties where the exact ones are out of reach."
tags = [
  "echidna",
  "formal-verification",
  "symbolic-execution",
  "fuzzing",
  "hevm",
]
+++

<div class="tldr">

## TL;DR

We *proved* the core accounting math of [Spark's PSM3](https://docs.spark.fi/dev/savings/spark-psm) (share price, conversions, swap quotes) against the real bytecode: sixty properties, verified for every input up to `uint128`, with all the code in [our spark-psm branch](https://github.com/gustavo-grieco/spark-psm/tree/symbolic-conversion-proofs). The proofs run in Echidna's verification mode on top of hevm's arithmetic abstraction, still **experimental** and in review as [argotorg/hevm#1075](https://github.com/argotorg/hevm/pull/1075). What resisted exact proof got monotonicity and rounding bounds that still rule out the attacks; everything else went to a fuzzing campaign: some 80 hours and roughly 700 million executions against the repo's own fuzz tests and eight stateful invariants. All clean, except one small bug in the invariant harness itself. Whatever your seat: if you build protocols, your fuzz suite is closer to a proof harness than you think; if you hunt bugs, every verified property is code you no longer have to re-review. **Prove the core, fuzz the rest.**

</div>


It is Monday morning, and the contract in front of you has cleared three audits, ships a green invariant suite, and has held real money without incident. The easy bugs are long gone, so what do you run *now* to raise confidence beyond "a few billion random inputs didn't break it"? Two techniques answer that, with opposite blind spots. **Fuzzing** runs the real, unmodified contract. It is assumption-free, but it samples; absence of failure is not proof. **Formal verification** proves a property for *all* inputs, but only under a model, and a wrong model proves something about a contract subtly unlike the deployed one. Together, each covers the other: the proof settles the core math for every input; the fuzzer explores what the proof left out and checks that its assumptions didn't define the bug away.


## The contract: Spark's PSM


[`PSM3`](https://github.com/sparkdotfi/spark-psm/blob/master/src/PSM3.sol) is the Peg Stability Module from [Spark](https://spark.fi), in the Sky (formerly MakerDAO) ecosystem. It holds three related stable assets (USDC, USDS, and yield-bearing sUSDS), lets anyone swap between them at deterministic prices, and lets liquidity providers deposit for shares whose price tracks the pool's value as sUSDS accrues yield. Audits read the code and fuzzers sample it; neither *proves* the conversion math, and we know of no published formal verification of it.

![How PSM3 is used: swappers and liquidity providers call it on one side; it holds USDS and sUSDS, keeps its USDC at the pocket address, and prices sUSDS through the Savings Rate Oracle.](https://github.com/user-attachments/assets/a58f70bc-fe07-423d-af7c-abfaef99f444)


## Proving the core math


Strip PSM to its arithmetic and one shape appears everywhere: an amount times one quantity, divided by another, `a * b / c`. The share price is that shape; so are the swap quotes and deposit previews:


```solidity
// src/PSM3.sol (simplified)
function convertToShares(uint256 assetValue) public view returns (uint256) {
  uint256 totalAssets_ = totalAssets();
  if (totalAssets_ != 0) return assetValue * totalShares / totalAssets_;
  return assetValue;
}
```


To a prover all three variables are unknown **256-bit** values, and that width is the problem. An SMT solver decides bitvector arithmetic by *bitblasting*, expanding each operation into a Boolean circuit. Multiplying by a constant stays cheap, but multiplying two *unknowns* expands to a 256×256 multiplier (tens of thousands of gates), and dividing by an unknown is worse. Do both, as `a * b / c` does, and the solver runs forever, returns `unknown`, or exhausts memory. (Bounding the *values* doesn't help: a `require(x < 2**32)` leaves the terms 256 bits wide, and the same query still times out.)


**hevm's abstract arithmetic** refuses to expand exactly those operations. Instead of computing what `a * b` *is*, it treats the result as a sealed box and gives the solver a short list of facts true of *every* product and quotient: a product never shrinks when an input grows; `a * b = b * a`; a quotient never grows when the divisor does; anything times zero is zero.


That is the whole trick: trade the exact value you can't compute for a few relationships you can. The difference is dramatic. On a typical property (depositing more never mints fewer shares), native solving comes back `unknown` after a two-minute timeout; the abstraction proves it in a quarter of a second. This ships as one hevm change, [#1075](https://github.com/argotorg/hevm/pull/1075), a two-phase SMT abstraction for symbolic multiplication, division, and modulo, which echidna's verification mode turns on. Once that PR merges, the integration is on its way to echidna `master`; today it takes a custom build of both.


The trade has a precise soundness contract. **A proof is real:** every fact holds for ordinary arithmetic, so the proof covers a more general contract, of which the deployed one is a special case. **A counterexample is not:** it may hinge on a sealed value real arithmetic could never produce. So a clean run is `verified`, while a counterexample is downgraded to `passed`, never reported as a bug.


Relationships prove orderings, not magnitudes. Exact results come from *cancellation* lemmas tuned to the PSM's **precision scaling**: `x * 1e18 / 1e6` is pinned to `x * 1e12`, a two-step `/1e9 /1e18` folds into one `/1e27`, and one lemma relates two sealed products differing by a constant, enough to prove that a rate move changes the pool's value by *exactly* the sUSDS revaluation.


Most of the math we verify completely, against the repo's own tests. Each property inherits the contract's *unmodified* `testFuzz_*`; the only addition is one uniform assumption, the abstraction's operand budget:


```solidity
function prove_convertToAssets_susds(uint256 rate, uint256 amount) public {
   require(rate <= type(uint128).max && amount <= type(uint128).max);
   testFuzz_convertToAssets_susds(rate, amount);   // the repo's own fuzz test, unmodified
}
```


**Spark's own fuzz tests become theorems: exact values, proven for every `uint128` input, against the deployed bytecode**.


Where the exact value is out of reach, a weaker property still rules out the attack. A few quotes are a `ceilDiv` over a rate-scaled product, or divide by the symbolic pool ratio; there we prove **monotonicity**, the *direction* of the result, which is what an attacker probes:


- *more in never yields less out*: no underpaying by splitting or reordering a swap;
- *a higher rate never inverts pricing*: a rate update can't flip a quote;
- *value → shares → value never grows*: the first-depositor inflation attack, ruled out.


A separate bound catches what monotonicity would miss (monotone but mis-rounded): round-up and round-down never differ by more than one wei.


Despite our best efforts, eleven properties stay open, for two reasons. Three `previewSwapExactOut` legs resist the *abstraction*: their round-up (`ceilDiv`) quotes come back `unknown` within the time budget, and we have not pinned down exactly why. Eight fail on *setup*: they deposit through a real `MockERC20` before asserting, and the mock's keccak-mapping balance reads are intractable inside the abstracted arithmetic. Our own relational properties sidestep that wall with a modelling workaround: the symbolic harness deploys [minimal tokens whose `balanceOf` is a single storage slot](https://github.com/gustavo-grieco/spark-psm/blob/symbolic-conversion-proofs/test/symbolic/ProveRealPSM3.t.sol#L34) rather than a mapping, so each balance read is one clean symbolic value. **That is sound here because every property reads each token at exactly one holder and never moves tokens**; the deposit-driven tests as written do move them, so they go to the fuzzer.


**Result: sixty properties verified, nothing falsified.** Most discharge in about a second, and the whole set runs in about half an hour. In the table below, each property is either ✅ **verified** or ⏳ **out of reach**.


> **Limits**
>
> **① Bounded to `uint128`.** The no-overflow facts hold while products fit in 256 bits, so every proof assumes each input (and a few derived sums like `totalAssets()`) is at most `type(uint128).max`. That is about 3×10²⁰ tokens at 18 decimals, far past any real supply, but it is a real bound, so we name it.
>
> **② Quotes are proven; the swap acting on them is not.** The property that matters most, **an executed swap never pays out more than its quote**, ranges over transfers and balances the symbolic engine can't reach. The fuzzing campaign carries it, via the *value-conservation* and *rounds-in-the-protocol's-favour* invariants.


<details>
<summary>Full per-property results: 60 verified, 11 out of reach</summary>


| Result | Property | Contract | Notes |
| :---: | --- | --- | --- |
| ✅ | `convertToAssets_susds` | `PSMConvertToAssetsTests` |  |
| ✅ | `convertToAssets_usdc` | `PSMConvertToAssetsTests` |  |
| ✅ | `convertToAssets_usds` | `PSMConvertToAssetsTests` |  |
| ✅ | `convertToAssetValue_noValue` | `PSMConvertToAssetValueTests` |  |
| ✅ | `convertToShares_noValue` | `PSMConvertToSharesTests` |  |
| ✅ | `convertToShares_susds_noValue` | `PSMConvertToSharesWithSUsdsTests` |  |
| ✅ | `convertToShares_usdc_noValue` | `PSMConvertToSharesWithUsdcTests` |  |
| ✅ | `convertToShares_usds_noValue` | `PSMConvertToSharesWithUsdsTests` |  |
| ✅ | `getSUsdsValue_roundDown_rate_monotonic` | `ProveGetters` |  |
| ✅ | `getSUsdsValue_roundUp_monotonic` | `ProveGetters` |  |
| ✅ | `getSUsdsValue_roundUp_rate_monotonic` | `ProveGetters` |  |
| ✅ | `getAssetValue` | `PSMHarnessTests` |  |
| ✅ | `getSUsdsValue` | `PSMHarnessTests` |  |
| ✅ | `getUsdcValue` | `PSMHarnessTests` |  |
| ✅ | `getUsdsValue` | `PSMHarnessTests` |  |
| ✅ | `previewDeposit_susds_firstDeposit` | `PSMPreviewDeposit_SuccessTests` |  |
| ✅ | `previewDeposit_usdc_firstDeposit` | `PSMPreviewDeposit_SuccessTests` |  |
| ✅ | `previewDeposit_usds_firstDeposit` | `PSMPreviewDeposit_SuccessTests` |  |
| ✅ | `assetValue_backing_monotonic` | `ProveRealPSM3` |  |
| ✅ | `assetValue_monotonic` | `ProveRealPSM3` |  |
| ✅ | `convertToAssets_susds_monotonic` | `ProveRealPSM3` |  |
| ✅ | `convertToAssets_usdc_monotonic` | `ProveRealPSM3` |  |
| ✅ | `convertToAssets_usds_eq_value` | `ProveRealPSM3` |  |
| ✅ | `pd_susds_firstdeposit_mono` | `ProveRealPSM3` |  |
| ✅ | `pd_usdc_firstdeposit_mono` | `ProveRealPSM3` |  |
| ✅ | `pd_usds_firstdeposit_mono` | `ProveRealPSM3` |  |
| ✅ | `roundtrip_no_inflation` | `ProveRealPSM3` |  |
| ✅ | `roundtrip_shares` | `ProveRealPSM3` |  |
| ✅ | `shares_anti_monotonic_assets` | `ProveRealPSM3` |  |
| ✅ | `shares_monotonic` | `ProveRealPSM3` |  |
| ✅ | `shares_no_inflation` | `ProveRealPSM3` |  |
| ✅ | `totalAssets_exact` | `ProveRealPSM3` |  |
| ✅ | `convertToShares_totalValue_eq_totalShares` | `ProveRealPSM3` | `convertToShares(totalAssets()) == totalShares` |
| ✅ | `convertToAssetValue_totalShares_eq_totalAssets` | `ProveRealPSM3` | dual of the above |
| ✅ | `totalAssets_rateIncrease_valueChange` | `ProveRealPSM3` | value-change identity (telescoping lemma) |
| ✅ | `rounding_susdsValue_tight` | `ProveRounding` |  |
| ✅ | `previewSwapExactIn_susdsToUsdc` | `PSMPreviewSwapExactIn_SUsdsAssetInTests` |  |
| ✅ | `previewSwapExactIn_susdsToUsds` | `PSMPreviewSwapExactIn_SUsdsAssetInTests` |  |
| ✅ | `previewSwapExactIn_usdcToSUsds` | `PSMPreviewSwapExactIn_USDCAssetInTests` |  |
| ✅ | `previewSwapExactIn_usdcToUsds` | `PSMPreviewSwapExactIn_USDCAssetInTests` |  |
| ✅ | `previewSwapExactIn_usdsToSUsds` | `PSMPreviewSwapExactIn_UsdsAssetInTests` |  |
| ✅ | `previewSwapExactIn_usdsToUsdc` | `PSMPreviewSwapExactIn_UsdsAssetInTests` |  |
| ✅ | `previewSwapExactOut_usdsToUsdc` | `PSMPreviewSwapExactOut_UsdsAssetInTests` |  |
| ✅ | `previewSwapExactOut_usdsToSUsds` | `PSMPreviewSwapExactOut_UsdsAssetInTests` | round-up within 1 wei |
| ✅ | `previewSwapExactOut_susdsToUsds` | `PSMPreviewSwapExactOut_SUsdsAssetInTests` | round-up within 1 wei |
| ⏳ | `previewSwapExactOut_usdcToUsds` | `PSMPreviewSwapExactOut_USDCAssetInTests` | `ceilDiv` round-up quote: `unknown` within the time budget |
| ⏳ | `previewSwapExactOut_usdcToSUsds` | `PSMPreviewSwapExactOut_USDCAssetInTests` | `ceilDiv` round-up quote: `unknown` within the time budget |
| ⏳ | `previewSwapExactOut_susdsToUsdc` | `PSMPreviewSwapExactOut_SUsdsAssetInTests` | nested `ceilDiv` quote: `unknown` within the time budget |
| ✅ | `roundtrip_usds_susds` | `ProveSwapPreviews` |  |
| ✅ | `swapExactIn_susdsToUsdc_rate_increasing` | `ProveSwapPreviews` |  |
| ✅ | `swapExactIn_susdsToUsds_rate_increasing` | `ProveSwapPreviews` |  |
| ✅ | `swapExactIn_usdcToSUsds_rate_decreasing` | `ProveSwapPreviews` |  |
| ✅ | `swapExactIn_usdsToSUsds_rate_decreasing` | `ProveSwapPreviews` |  |
| ✅ | `swapExactOut_susdsToUsdc` | `ProveSwapPreviews` |  |
| ✅ | `swapExactOut_susdsToUsdc_rate_decreasing` | `ProveSwapPreviews` |  |
| ✅ | `swapExactOut_susdsToUsds` | `ProveSwapPreviews` |  |
| ✅ | `swapExactOut_susdsToUsds_rate_decreasing` | `ProveSwapPreviews` |  |
| ✅ | `swapExactOut_usdcToSUsds` | `ProveSwapPreviews` |  |
| ✅ | `swapExactOut_usdcToSUsds_rate_increasing` | `ProveSwapPreviews` |  |
| ✅ | `swapExactOut_usdcToUsds` | `ProveSwapPreviews` |  |
| ✅ | `swapExactOut_usdsToSUsds` | `ProveSwapPreviews` |  |
| ✅ | `swapExactOut_usdsToSUsds_rate_increasing` | `ProveSwapPreviews` |  |
| ✅ | `swapExactOut_usdsToUsdc` | `ProveSwapPreviews` |  |
| ⏳ | `convertToShares_conversionRateIncrease` | `PSMConvertToShares*Tests` (×4) | deposit-setup wall; identity proved as `convertToShares_totalValue_eq_totalShares` |
| ⏳ | `convertToAssetValue_conversionRateIncrease` | `PSMConvertToAssetValueTests` | deposit-setup wall; dual identity proved directly |
| ⏳ | `convertToAssetValue_conversionRateDecrease` | `PSMConvertToAssetValueTests` (+4 reuse) | deposit-setup wall |
| ⏳ | `previewWithdraw` | `PSMPreviewWithdraw_SuccessFuzzTests` | deposit-setup wall |
| ⏳ | `previewDeposit_afterDepositsAndExchangeRateIncrease` | `PSMPreviewDeposit_SuccessTests` | deposit-setup wall |
| ⏳ | `roundingAgainstUser_multiUser_usds` | `RoundingTests` | deposit-setup wall |
| ⏳ | `roundingAgainstUser_multiUser_usdc` | `RoundingTests` | deposit-setup wall |
| ⏳ | `roundingAgainstUser_multiUser_susds` | `RoundingTests` | deposit-setup wall |
</details>


## Then fuzz the rest


A proof of the stateless math is narrow by design; the fuzzer takes everything around it.


The same properties come first, on the real arithmetic: echidna runs the repo's `testFuzz_*` directly, with real `mul` and `div` on full `uint256`, covering the ⏳ rows and the whole swap/deposit/withdraw *execution* surface, plus nine single-transaction properties of our own (swap value-conservation, both conversion round-trips, preview monotonicity) written for what the unit suite doesn't assert. It also independently checks the proofs: if a lemma were unsound, the prover would still say `verified`, but the fuzzer on the same code would not stay quiet. Across two campaigns (an eight-hour sweep of every stateless target, then a further 24 hours on the surfaces the proofs leave open) that is **over 600 million single-transaction executions with zero falsifications**, exercising the conversion math in both rounding directions.


Beyond the stateless properties, PSM3's real risk is emergent: long sequences of deposits, swaps, withdrawals, and rate updates by many actors. We drive the protocol's own handler suite and check [**eight invariants**](https://github.com/gustavo-grieco/spark-psm/blob/symbolic-conversion-proofs/test/invariant/InvariantsEchidna.t.sol) after every step: shares sum to the supply, the pool stays solvent, value is neither created nor destroyed, swaps round in the protocol's favour, and at teardown every position can still be withdrawn in full. That fourth invariant is exactly the end-to-end safety the proofs left open (caveat ②). Two 24-hour campaigns, roughly 90 million transactions in all, and every invariant held.


The audits picked one invariant for us: re-reading the reports surfaced exactly one property the suite didn't already enforce: a Cantina finding where a withdraw could round the **share price** down, diluting LPs. It was fixed years ago, but a fixed bug is exactly what a regression invariant should pin forever. Now it is one: [the share price never falls by more than a rounding wei after any action](https://github.com/gustavo-grieco/spark-psm/blob/symbolic-conversion-proofs/test/invariant/InvariantsEchidna.t.sol#L425), with a [companion invariant that the sUSDS rate only accrues upward](https://github.com/gustavo-grieco/spark-psm/blob/symbolic-conversion-proofs/test/invariant/InvariantsEchidna.t.sol#L436).


Building the harness caught a bug of its own: point `setPocket` at an address that already belongs to a liquidity provider and the two become one account: USDC withdrawals turn into no-op self-transfers while the ghost accounting still books the outflow, breaking `invariant_E` (pocket balance == tracked inflows − outflows). A test-setup flaw, not a contract vulnerability, but a genuine result, reported upstream as [sparkdotfi/spark-psm#53](https://github.com/sparkdotfi/spark-psm/issues/53).


Each leg of the argument is one command in the repo's [Makefile](https://github.com/gustavo-grieco/spark-psm/blob/symbolic-conversion-proofs/Makefile): `make verify`, `make fuzz T=<Contract>`, `make fuzz-invariant`.


The same result reads differently depending on your seat. If you build protocols, your existing fuzz suite is closer to a proof harness than you think; a uniform `require` plus echidna's verification mode on hevm's abstract arithmetic turned Spark's own tests into theorems. If you hunt bugs, verification is a filter: **a verified property is code you no longer have to re-review by hand**, and running the proofs yourself on a target leaves you a shorter, sharper list of things worth reading, starting with the out-of-reach rows and the executed-swap gap above.

**Prove the core; fuzz the rest.** The conversion math is now a set of theorems about the deployed bytecode, everything else is fuzzed on top, and the one property an audit once found broken is an invariant that can't quietly come back. Neither technique reaches that on its own.

## Oh, Just One More Thing

If you notice anything in this blog post that is incorrect, imprecise, or potentially misleading, please feel free to [contact me](https://forms.gle/V3jt7C2JQgZhoXfe9) for clarification. And a thank-you: this post was funded by the recent donation round on [Giveth for Echidna](https://giveth.io/project/echidna:-a-fast-smart-contract-fuzzer); work like this is possible because of everyone who contributed.
