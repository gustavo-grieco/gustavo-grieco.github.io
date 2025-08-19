+++
title = "Echidna Enters a New Era of Symbolic Execution"
date = "2025-08-19"
description = "A blog post describing the new symbolic execution capabilities of Echidna include preliminary results and immediate future"
tags = [
    "echidna",
    "symbolic-execution",
    "fuzzing"
]
+++

In this post, we will see a bit of the new Echidna capabilities using the enhanced symbolic execution from hevm. In a nutshell, **symbolic execution works in the same way as fuzzing, checking whether a program has specific issues, like assertion failures**. However, unlike fuzzing, **it either confirms the program works correctly by showing no paths lead to these issues, or it finds examples that prove such issues exist**. An initial implementation Echidna's symbolic execution capabilities was [added last year](https://github.com/crytic/echidna/pull/1216), but it was recently consolidated after [PR 1349](https://github.com/crytic/echidna/pull/1394) was merged, which implemented symbolic execution in two "flavors": 


* **Verification mode for stateless tests**: This mode aims to prove the absence of bugs, aligning with tools like [hevm](https://hevm.dev/), [Halmos](https://github.com/a16z/halmos), and [Certora](https://www.certora.com/). 
* **Exploration mode for stateful tests**: This mode combines symbolic execution with fuzzing campaigns to identify assertion failures in scenarios involving state changes. 


The integration of symbolic execution into Echidna is guided by a key principle: **vulnerability research tools should minimize the requirements placed on users**. In the context of smart contract security, this means no new cheat codes or specialized tests are needed, just the same codebase used in traditional fuzzing campaigns.  


Before diving deeper, we would like to acknowledge the **hevm team** for their exceptional work over the past year. Their improvements to symbolic execution performance and capabilities, detailed in the [recent release](https://github.com/ethereum/hevm/releases/tag/release%2F0.55.1), have been critical to enabling seamless integration with tools like Echidna.  


## Verification Mode


Verification mode mirrors formal verification techniques applied to stateless (or single-transaction) code. In Echidna’s new verification mode, the tool analyzes a contract by first executing its constructor and then symbolically testing each method. This approach is ideal for fully stateless functions (e.g., mathematical logic) but can also be used strategically in stateful code, similar to [Halmos' methodology](https://github.com/a16z/halmos/blob/main/docs/getting-started.md). 


When a test is explored using the symbolic engine in verification mode, there a few of possible results:


* Verified ✅. The code was fully explored, without any issues on the translation, solving. As expected, no counterexamples.
* Passed  👍. The code was fully explored without detecting any counter examples, but the SMT solver cannot solve some of the queries (e.g. it timed out), so the assertion could still fail.
* Failed 💥. The exploration revealed a counterexample that was successfully replayed in concrete mode.
* Error ❌. A bug or a missing feature blocks the exploration or solving of some paths.
* Timeout ⏳. There are scalability issues preventing the creation of the model to explore all the program paths. 


So when a transaction checks out during verification, here's what's happening under the hood:


* **Full Coverage Check**: the tool basically reviewed through every possible path the contract could take after the constructor finishes using a single transaction. If you ever tweak the contract code, this whole check needs to restart from scratch.
* **No counterexample was found**: your assertions should hold!
* **External Calls**: If your contract can call arbitrary addresses, we only look at the ones you've actually deployed in your tests. This means we can't catch sneaky reentrancy attacks from "unknown" contracts you haven't tested with. 
* **The Fineprint**: All calldata parameters, msg.sender, msg.value, block.timestamp, and block.number are treated as symbolic variables, allowing the analysis to explore all possible values. 
   * The sender addresses are limited to the ones you specified in your test config (like owner and non-owner accounts). If you need to test with other addresses, make sure they're in your config!
   * Transaction value can't go above the maximum specified in your config file.
   * The largest timestamp and block number jumps it will consider are exactly the ones you set in your config. If your test needs bigger changes, you'll need to update those values
   * To simplify analysis, hevm assumes infinite gas by default. This means solutions might include scenarios requiring unrealistic gas limits, which may not be practical. However, this overapproximation ensures no valid findings are missed, even if some theoretical results are overly optimistic.


We decided to do a test drive with a recent campaign from [Algebra](https://github.com/cryptoalgebra/Algebra/tree/integral-v1.2.2/src/core/contracts/test/echidna), which contains a number of stateless tests. All the tests were performed in a MacBook Pro M4 Max using 10 solvers (bitwuzla 0.7):


| Test Name | Contract | Result | Approximate time required | Notes |
| ----- | ----- | ----- | ----- | ----- |
| checkMulDivRounding | FullMathEchidnaTest | Passed 👍 | Minutes | 5 SMT queries are timing out |
| checkMulDiv | FullMathEchidnaTest | Passed 👍 | Minutes | 18 SMT queries are timing out |
| checkMulDivRoundingUp | FullMathEchidnaTest | Passed 👍 | Minutes | 40 SMT queries are timing out |
| addDelta | LiquidityMathEchidnaTest | Verified ✅ | Seconds |  |
| checkGetAmountsForLiquidity | LiquidityMathEchidnaTest | Timeout ⏳ | Unknown | Exploration is somehow stuck |
| checkAdd | LowGasSafeMathEchidnaTest | Verified ✅ | Seconds |  |
| checkSub | LowGasSafeMathEchidnaTest | Verified ✅ | Seconds |  |
| checkMul | LowGasSafeMathEchidnaTest | Verified ✅ | Seconds |  |
| checkAddi | LowGasSafeMathEchidnaTest | Verified ✅ | Seconds |  |
| checkSubi | LowGasSafeMathEchidnaTest | Verified ✅ | Seconds |  |
| mulDivRoundingUpInvariants | TokenDeltaMathEchidnaTest | Passed 👍 | Minutes | 7 SMT queries are timing out |
| getNextSqrtPriceFromInputInvariants | TokenDeltaMathEchidnaTest | Passed 👍 | Minutes | 49 SMT queries are timing out |
| getNextSqrtPriceFromOutputInvariants | TokenDeltaMathEchidnaTest | Passed 👍 | Minutes | 21 SMT queries are timing out |
| getNextSqrtPriceFromAmount0RoundingUpInvariants | TokenDeltaMathEchidnaTest | Passed 👍 | Minutes | 12 SMT queries are timing out |
| getNextSqrtPriceFromAmount1RoundingDownInvariants | TokenDeltaMathEchidnaTest | verified ✅ | Minutes |  |
| getToken0DeltaInvariants | TokenDeltaMathEchidnaTest | Passed 👍 | Minutes | 18 SMT queries are timing out |
| getToken0DeltaEquivalency | TokenDeltaMathEchidnaTest | Passed 👍 | Minutes  | 19 SMT queries are timing out |
| getToken1DeltaInvariants | TokenDeltaMathEchidnaTest | Verified ✅ | Minutes |  |
| getToken0DeltaSignedInvariants | TokenDeltaMathEchidnaTest | Passed 👍 | Minutes | 2 SMT queries are timing out |
| getToken1DeltaSignedInvariants | TokenDeltaMathEchidnaTest | Verified ✅ | Seconds |  |
| getOutOfRangeMintInvariants | TokenDeltaMathEchidnaTest | Passed 👍 | Minutes | 7 SMT queries are timing out |
| getInRangeMintInvariants | TokenDeltaMathEchidnaTest | Passed 👍 | Minutes | 1 SMT query is timing out |
| checkDivRoundingUp | UnsafeMathEchidnaTest | Verified ✅ | Seconds |  |
| checkGetNewPriceAfterInputInvariantOtZ | PriceMovementMathEchidnaTest | Passed 👍 | Hours | 7 SMT queries are timing out |
| checkMovePriceTowardsTargetInvariants | PriceMovementMathEchidnaTest | Timeout ⏳ | Unknown | At least 24 hours exploring |
| checkGetNewPriceAfterInputInvariantZtO | PriceMovementMathEchidnaTest | Passed 👍 | Minutes | 35 SMT queries are timing out |
| leastSignificantBitInvariant | BitMathEchidnaTest | Error ❌ | Seconds | Pow exponent is symbolic, not supported in SMTLIB2 |


As you can see, there is still a long way to go, but it is clear to say symbolic execution tools are starting to mature enough to tackle real code, even if they take a bit of time. Keep in mind that there is a large variance in the time needed, as it ranges from seconds for simple code to hours or days in the most complex cases.


Is it possible to actually prove most of the invariants that are passing now? There is a good chance to do it when hevm switches to using multiple SMT solvers at same time with tools like ["just solve it"](https://github.com/a16z/jsi) especially for the invariants that have a small number of queries timing out. On top of that, while we found no failing properties during these experiments, there is more than meets the eye: essentially we're able to provide guarantees beyond fuzzing but (without changing the tests in any way) and that’s very valuable.


### Symbolic Execution Disclaimer


It is important to have a discussion on what verification means in practice. As you probably know, **formal verification is rarely completely bulletproof: there are a number of shortcomings and limitations that need to be taken into consideration** coming either from theory (e.g. keccak256 hashes are hard to invert) and practice (e.g. SMT solvers can get stuck). That's why it is extremely important that the output of a tool using symbolic executions reflects any potential shortcoming or limitations, as the developer could incorrectly interpret it as "there are no bugs here". There is an interesting discussion in the recent Galois blog post ["What works and doesn't selling formal methods"] (https://www.galois.com/articles/what-works-and-doesnt-selling-formal-methods). The [hevm](https://hevm.dev/limitations-and-workarounds.html) and the [halmos](https://github.com/a16z/halmos/wiki/warnings) documentation describe a few things to need to be considered when doing symbolic execution in EVM.


We also wanted to highlight that trying to verify code with an unbounded loop, including any function that takes dynamic data structures as inputs (e.g. `bytes`) is not recommended given the current state of tooling. In fact, Echidna will not perform symbolic execution on any function with dynamic data structures as inputs. While there are techniques to concretize the size of them, they need to be used with care to avoid having blind spots in the code verification procedure.


## Exploration Mode


This mode is kind of a new thing for blockchain. This mode combines traditional fuzzing and symbolic execution to try to discover new inputs that trigger assertion failures, inspired by seminal research such as [Driller (2016)](https://sites.cs.ucsb.edu/~vigna/publications/2016_NDSS_Driller.pdf). 
Typically, FV tools will work from the deployment state of a contract and try to do a number of symbolic execution transactions in order to incrementally reach deeper and deeper states:

![imaga3](https://github.com/user-attachments/assets/a2944ae7-0b52-4802-b1db-a247419f57a3 "400px")

The immediate issue with this approach is that the number of states tends to grow for each symbolic transaction, making each new set of constraints harder and harder to solve. FV tools use a number of tricks to reduce the possible transaction combinations (e.g. analyzing how slots are read/write)


On the opposite side, fuzzing tools explore the contract state using concrete transactions. Here we consider a corpus that can contain many transactions, but only show the first one:


  
[b]


The immediate issue with fuzzing is that there is a good amount of "luck" involved in reaching an assertion failure, depending on how the state is built for the current corpus. Echidna's exploration mode using symbolic execution allows combining both approaches relying on the corpus accumulated over time but executing a single symbolic executing transaction on top of it.


  
[c]


The effect is executing a potentially more scalable transaction, but we are relying less on luck, since the symbolic executive will explore that particular state more exhaustively. 


As an example of the symbolic execution capabilities, let's take a look at this code showing an example of a ERC4626 vault that aggregates other three vaults: 


```
contract MultiVault {
    uint256 internal vaultNumber;
    uint256[3] internal vaultBalance;
    uint256 internal minSupply = 100 ** 18;

    function sqrt(uint x) internal returns (uint y) {
        uint z = (x + 1) / 2;
        y = x;
        while (z < y) {
            y = z;
            z = (x / z + z) / 2;
        }
    }
 
    function addVault(uint128 supply, uint8 decimals) public {
        require(vaultNumber < 3);
        require(decimals >= 8);
        require(decimals <= 18);
        uint256 normalizedSupply;
        if (decimals <= 18)
            normalizedSupply = supply * (10 ** (18 - decimals));
        else 
            normalizedSupply = supply / (10 ** (decimals - 18));

        require(normalizedSupply > 100 ** 18); // min supply
        vaultBalance[vaultNumber] = sqrt(normalizedSupply);
        vaultNumber++;
    }

    function previewWithdraw(uint256 assets) external returns (uint256 shares) {
        if (vaultNumber < 3)
            return 0;

        uint256 totalSupply = assets - (vaultBalance[0] + vaultBalance[1] + vaultBalance[2]);
        assert(totalSupply > 0);
    }
}
```


Note that the process of adding a vault involves calculating the square root using a loop, an operation that symbolic execution tools have a hard time dealing with, since it needs to either unroll the loop a maximum number of times (which we don't know how many since depends on the input) or use a suitable loop invariant (there is [a series of great blog post about loop invariants](https://runtimeverification.com/blog/formally-verifying-loops-part-1) by the Runtime Verification folks).


For the actual invariant, we added a simple assertion that states that total supply cannot be empty after a withdrawal. For sure it is not a very interesting invariant, but something simple enough to show how this mode works. To run this, we execute: 


```
echidna multiVault.sol --test-limit 1000000000000 --sym-exec true 
```


Breaking this assertion is hard for a fuzzer since it needs to create the three vaults and then select the exact amount to withdraw. Typically, if you have to deal with this pattern in the code, you will need to introduce a "ghost variable" that accumulates the normalized balances and make sure the fuzzer can access it (e.g. it should be public). These constraints are easy to solve for the symbolic worker, which finds a counter example that trigger the assertion failure in seconds and minimizes the sequence:


```
 Call sequence:
    addVault(187437275246714327976927632528824801846,17)
    addVault(212381979343444824622226965412329656662,13)
    addVault(340282366920938463463374607431768211452,9)
    previewWithdraw(340303606993245560412980109897585769705914218460)
```


It is worth mentioning that in this example, we know that we are going to reach a certain part of the code after three transactions, so we could create a test to simulate these three transactions, perhaps making some of the inputs symbolic. However, it is faster not to do it, as we do not really care about which transactions we used to reach that state, that's the power of random testing!


A summary of what happens during this fuzzing campaign:


1. Before starting, a static analysis tool detected important constants and identified functions with assertions.
2. The fuzzing engine initially explored the contract state and found a number of interesting states (using coverage).
3. The symbolic engine asked for one of these states, selected a function with an assertion (e.g. `previewWithdraw`) and started to run fully symbolic transactions from there.
4. The symbolic execution created a model, solved the constraints and produced an input for a transaction that triggers an assertion failure.
5. The fuzzing engine validated the transaction produced by the symbolic engine and confirmed an assertion failure.
6. The fuzzing engine reduced the number of transactions needed for triggering the assertion failure.


We just saw a tool that combined all the major [program analysis techniques](​​https://en.wikipedia.org/wiki/Program_analysis) to automatically discover and simplify an input that breaks an invariant and that's very cool!


## How to Use it?


Currently, symbolic execution modes operate **exclusively in assertion mode** and are limited to detecting assertion failures. If you’re testing these features, keep that in mind. 


You will need to install a SMT solver. It is strongly recommended to use [bitwuzla](https://bitwuzla.github.io/), especially compared to other alternatives such as Z3. Once it is installed, you can turn symbolic execution using `--sym-exec true`. In order to know which mode to use, Echidna uses other parameters:


* In verification mode, we turn off fuzzing workers and use a single transaction, so this needs `--workers 0 --seq-len 1`. 
* In exploration mode, we can use a positive number of workers and any number of transactions. 


As you can see, this interface is not very intuitive, and it should be eventually replaced by something better like dedicated mode (e.g. `--test-mode verification` or something like that).


Once Echidna starts, we can check the symbolic workers logs to see if some parameters need to be tweaked. It is often useful to use `--format text` to know what the symbolic worker is doing (but the same information is available in the TUI). In particular, there are a few important warnings to pay attention to:


* `Partial explored path(s) during symbolic verification of method …: Branches too deep at program counter: …`, then you must increase `symExecMaxExplore`. This is probably the most important symbolic execution parameter, which determines the number of maximum branches explored. Keep in mind that when you use a value such as 20, it means it can reach up to 2**20 states, so go easy on it!
* `Max Iterations Reached in contract: …` then you must increase `symExecMaxIters` : this parameter signals the symbolic engine how many times the exploration is allowed to visit a given instruction. Typically this applies to loops, however, it can also be useful when the same function is called multiple times or when the solc optimizer reduces the size of the contract finding duplicated snippets of code. 
* `Error(s) during symbolic exploration: "Result unknown by SMT solver"` then you need to increase the `symExecTimeout` parameter. By default it is 30 seconds, you can try with something like 120 or even 300.


## When to use it?


So, when should a smart contract developer or security researcher use symbolic execution? From our limited experience testing this new feature, **it should not be the default choice**. In most cases, fuzzers are already very effective at exploring states and uncovering counterexamples. Instead, symbolic execution should be applied in contexts where we want to provide **stronger security guarantees**. In particular:  
* When dealing with math-heavy functions that have no state (verification mode).  
* When adding an extra layer of assurance after an extensive fuzzing campaign.  


There is still an interesting challenge worth discussing, one related to common patterns often used in fuzzing campaigns. Consider this example:


```
contract TestTarget {
   function callOperation(uint256 p1, uint256 p2, address p3) { 
     Target.operation(p1, p2, p3);
     … 
   }


   function checkAsserts() { 
        assert(..);
   }
}
```


With the current implementation, symbolic execution is not very effective for this pattern. On the one hand, `callOperation` usually contains no assertions, so the symbolic engine skips it. On the other hand, `checkAsserts` has no symbolic inputs (besides block timestamp/number and msg.sender), so it is also skipped, since most of its data is concrete.


A common workaround is to combine these two functions into a single call (e.g., `callOperationAndCheckAsserts`). While this works for experimentation, it’s not an ideal long-term solution. Traditional symbolic execution tools face this challenge all the time when trying to efficiently explore beyond a single transaction.


Fortunately, **Slither, our static analysis engine**, can help. It reveals data dependency relationships between functions, such as:
```
"functions_relations": {
        "TestTarget": {
            "callOperation(uint256,uint256,address)": {
                "impacts": [
                    "checkAsserts()",
                    …
                ],
             }
        }
}
```
This information allows us to identify meaningful chains of transactions for symbolic execution, focusing exploration only on the most relevant combinations.


## Upcoming features and issues to solve
### Power to users!


It’s important to clearly show users the results of symbolic execution, even when no counterexamples are found or when some states remain unexplored. One interesting proposal is to extend fuzzing campaign coverage reports beyond concrete execution, adding a layer of *"symbolic coverage"* to highlight which lines were reached by the symbolic engine. For example, in this case we could mark lines reached symbolically with `s`:


```
*rs |     function liquidate(address user, uint256 amount) public {
*rs |         require(amount > 0, "Invalid amount");
    |
* s |         uint256 debt_user = debt[user];
* s |         uint256 collateralETH_user = collateralETH[user];
*rs |         require(debt_user > 0, "No debt");
    |
* s |         uint collateralUSD = (collateralETH_user * ethPrice) / ETH_UNIT;
* s |         uint ratio = (collateralUSD * 100) / debt_user;
    |
*rs |         require(ratio < MIN_COLLATERAL_RATIO, "Healthy position");
    |
*   |         if (ratio >= EMERGENCY_MODE_RATIO) {
    |             require(amount <= debt_user, "Amount exceeds debt");
    |         } else {
*r  |             revert("emergency mode active");
    |         }
    |         … 
```




This kind of reporting would help developers and auditors better understand both the strengths and the limitations of symbolic execution. It could also encourage them to simplify their code to make it more "tool-friendly."  


Note that this isn’t a brand-new idea: Halmos recently [introduced support for a similar feature](https://a16zcrypto.com/posts/article/halmos-v0-3-0-release-highlights/).


### Automatic Bounds Extraction


Many smart contract developers use `require` statements at the beginning of a function to define preconditions, especially for given parameters. For example:


```
 function f(uint256 x, uint256 y) external {
      require(x <= balance[msg.sender]);
      require(y + 1 >= 0x99);
      …
  }
```
What if we could extract valid ranges for `x` and `y`, so Echidna knows which inputs are "valid" and can explore contract states more efficiently? **This is actually possible with a bit of symbolic execution**: a basic prototype of automatic bound extraction is [available here](https://github.com/crytic/echidna/pull/1409). You can already inspect the bounds it produces. For example, for the function above, one possible set of extracted constraints could be:
```
Constraints resolved:
arg1 <= 0x100000000
arg2 >= 0x98
```
In this case, one constraint depends on the contract’s state. In this example, Echidna produces a constraint where `balance[msg.sender]` is `0x100000000`.


The initial prototype was surprisingly easy to build. However, integrating it into the fuzzing engine’s workflow is challenging: we need to synchronize the concrete and symbolic workers to propagate bounds when a specific state is explored. The campaign code needs to be completely refactored before we can do that.


This code was developed after a discussion started by [@GalloDaSballo](https://github.com/GalloDaSballo), who asked whether symbolic execution could be used to extract simple input constraints that prevent reverts, thereby speeding up fuzzing campaigns (similar to what [pyrometer](https://github.com/nascentxyz/pyrometer/) computes). The answer is yes, and **this idea will eventually be integrated into the fuzzing engine to enable faster exploration**.




### Benchmark, Benchmark, Benchmark


A benchmark using a set of real smart contracts, designed for testing both symbolic and concrete exploration, is on the way. It will serve as a testbed for different symbolic execution frameworks and tools, allowing us to evaluate how well they handle complex code. Stay tuned!


## Oh, Just One More Thing


If you notice anything in this blog post that is incorrect, imprecise, or potentially misleading, please feel free to [contact me](https://forms.gle/V3jt7C2JQgZhoXfe9) for clarification. If you’ve tested this new Echidna feature and encountered any issues, please [open an issue](https://github.com/crytic/echidna).
