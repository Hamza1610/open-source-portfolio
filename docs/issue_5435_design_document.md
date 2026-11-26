# Design Document: Forced Conflicting Coin Selection Optimization

**Author:** Contributor  
**Target Repository:** `lightningdevkit/rust-lightning`  
**Tracking Issue:** [#5435 (Forgejo)](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/issues/5435)  
**Affected Component:** [`lightning/src/util/wallet_utils.rs`](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/src/branch/main/lightning/src/util/wallet_utils.rs)  
**Classification:** Policy Enhancement / On-Chain Resilience (P3 Hardening)  

---

## 1. Engineering Background & Skill Alignment

Resolving Issue #5435 requires a convergence of systems-level Rust programming, Bitcoin transaction construction mechanics, and algorithmic optimization under strict state invariants.

### 1.1 Systems-Level Rust Engineering
* **Data Structure Manipulation:** The implementation demands low-overhead collection processing within `lightning/src/util/wallet_utils.rs`. This involves nested hashing (`HashMap`), double-ended queues (`VecDeque`), composite ordering keys with lexicographical comparison (`sort_unstable_by_key`), and `core::cmp::Reverse` wrappers.
* **Invariant & State Safety:** LDK maintains a locked-coin tracking table in memory (`locked_utxos: Mutex<HashMap<OutPoint, Option<ClaimId>>>`). Manipulating these entries during coin selection requires clean lock management, avoiding deadlocks, and maintaining rollback integrity if downstream lookups fail (`get_prevtx`).
* **Zero Allocations & `no-std` Compliance:** Core crates in LDK target constrained environments. Modifications must avoid unbounded allocations, unnecessary cloning, and external runtime dependencies.

### 1.2 Bitcoin Protocol & Transaction Construction
* **UTXO Effective Value:** Accurate selection depends on calculating the *effective value* of an output (`value - fee_for_weight`), properly factoring in witness satisfaction weights (`P2WPKH`, `P2TR`, `P2WSH`), legacy script overhead, and dust limits.
* **Mempool Replacement (RBF) & Claim Conflict Dynamics:** When a wallet's liquid UTXOs are exhausted, choosing which locked UTXO to borrow effectively initiates an in-protocol double-spend. Understanding how package claiming (`lightning/src/chain/package.rs`) handles displaced claim transactions—and how RBF fee escalation cascades—is essential to evaluating displacement policies.

### 1.3 Algorithmic Design & Independent Research
* **Heuristic vs. Combinatorial Trade-offs:** The problem can be framed as a constrained knapsack problem. Addressing it practically in a synchronous loop requires evaluating greedy approximations against exact combinatorial solvers, balancing runtime complexity ($\mathcal{O}(N \log N)$) against displacement optimality.
* **Codebase Navigation:** Investigating the broader context of derived claim IDs across batching logic ([`lightning/src/events/bump_transaction/mod.rs#L727`](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/src/branch/main/lightning/src/events/bump_transaction/mod.rs#L727)) and height-based timer re-bumps ([`lightning/src/chain/package.rs#L1438-L1446`](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/src/branch/main/lightning/src/chain/package.rs#L1438-L1446)) informs the architectural boundary between the wallet interface and the package state machine.

---

## 2. Problem Context & Root Cause Analysis

### 2.1 The Invariant
[`CoinSelectionSource`](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/src/branch/main/lightning/src/util/wallet_utils.rs#L352) defines the protocol for funding on-chain claims (e.g., anchor transactions, HTLC resolutions). Requirement 4 and lines 383–385 explicitly dictate:
> *"If all of the available UTXOs have already been assigned to other claims, implementations must be willing to double spend their UTXOs. The choice of which UTXOs to double spend is left to the implementation, but it must strive to keep the set of other claims being double spent to a minimum."*

### 2.2 The Failure Mode
[`Wallet::select_confirmed_utxos_internal`](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/src/branch/main/lightning/src/util/wallet_utils.rs#L531) falls back to `force_conflicting_utxo_spend = true` only after attempts using unreserved coins have failed. 

In this fallback state:
1. Every locked UTXO in `self.locked_utxos` becomes eligible.
2. The candidates are sorted globally by effective value alone:
   ```rust
   eligible_utxos.sort_unstable_by_key(|(utxo, fee_to_spend_utxo)| {
       utxo.output.value - *fee_to_spend_utxo
   });
   ```
3. A greedy walk picks UTXOs starting from the smallest viable coin until the target amount and fees are satisfied.

Because the sort is completely oblivious to the owning `ClaimId`, coins from multiple unrelated claims are greedily selected in fragmented amounts.

### 2.3 Concrete Example
* **Claim A** holds two UTXOs: `1,000 sat` and `3,000 sat`.
* **Claim B** holds one UTXO: `1,100 sat`.
* **Claim C** requires `1,800 sat`.

* **Current Implementation:** Sorts as `[1,000 (A), 1,100 (B), 3,000 (A)]`. The greedy pass takes `1,000` then `1,100`, totaling `2,100 sat`.  
  **Outcome:** Displaces **both** Claim A and Claim B.
* **Optimal Implementation:** Takes the `3,000 sat` coin from Claim A.  
  **Outcome:** Displaces **only** Claim A.

---

## 3. Comparative Analysis of Architectural Approaches

There are three primary heuristics for resolving which claims to displace. Each introduces distinct trade-offs regarding computational complexity, wallet isolation, and on-chain fee efficiency.

```
+----------------------------------------------------------------------------------------------------+
|                                      Comparative Trade-Off Matrix                                  |
+----------------------+--------------------------+-----------------------+--------------------------+
| Metric               | Approach 1: Richest Claim| Approach 2: Fewest    | Approach 3: Deadline /   |
|                      | First                    | Coins / Cardinality   | Urgency-Aware            |
+----------------------+--------------------------+-----------------------+--------------------------+
| Optimality Goal      | Minimize claim count via | Minimize claim count  | Protect near-deadline    |
|                      | value concentration      | via subset search     | claims from timeout      |
+----------------------+--------------------------+-----------------------+--------------------------+
| Runtime Complexity   | O(N log N)               | O(2^K) or O(K * W)    | O(N log N)               |
+----------------------+--------------------------+-----------------------+--------------------------+
| Architectural Layer  | Self-contained in Wallet | Self-contained in     | Leaks Package/Chain state|
|                      |                          | Wallet                | into Wallet interface    |
+----------------------+--------------------------+-----------------------+--------------------------+
| Memory Overhead      | O(N) map entries         | O(N) auxiliary        | O(N) + external state    |
+----------------------+--------------------------+-----------------------+--------------------------+
| Downstream Impact    | High-value claim must    | May select multiple   | Urgent claims preserved; |
|                      | re-bump on next timer    | smaller claims        | high architectural churn |
+----------------------+--------------------------+-----------------------+--------------------------+
```

---

### Approach 1: Richest Conflicting Claim First (Value-Group Greedy Sort)

#### System Design & Mechanics
1. Aggregate the total effective value of all available conflicting coins owned by each distinct `ClaimId`.
2. Construct a composite sorting key:
   $$\text{Key} = (\text{Reverse}(\text{Total Claim Group Value}),\ \text{ClaimId},\ \text{Individual Effective Value})$$
3. Unlocked coins (owned by `None`) retain the highest priority ($\text{Group Value} = \infty$).
4. The greedy pass executes sequentially on this sorted list.

#### Advantages
* **Single-Claim Resolution:** A single claim that possesses enough aggregated value to satisfy the target is evaluated first, inherently containing displacement to one claim.
* **Self-Contained Modularity:** Operates entirely within [`Wallet::select_confirmed_utxos_internal`](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/src/branch/main/lightning/src/util/wallet_utils.rs#L531). No changes are required to trait interfaces, public signatures, or package monitors.
* **Predictable Performance:** $\mathcal{O}(N \log N)$ execution time with negligible allocation overhead (`new_hash_map()`).

#### Disadvantages & Trade-offs
* **Disproportionate Displacement:** If Claim A holds `500,000 sat` and Claim B holds `2,000 sat`, an incoming claim needing `1,900 sat` will displace Claim A, forcing a large fee-bump recalculation for Claim A on its next timer tick.

---

### Approach 2: Minimum Displaced Claim Cardinality (Exact Subset Minimization)

#### System Design & Mechanics
1. Treat each conflicting `ClaimId` as an atomic set of funds.
2. Evaluate which single claim covers the target. If none can, find the minimal subset of claims ($\min |S|$) whose combined value meets the threshold.
3. Once the target claim subset is selected, draw individual coins from those claims.

#### Advantages
* **Strict Adherence to Specification:** Directly minimizes the mathematical cardinality of displaced claims under all input configurations.
* **Prevents Unnecessary High-Value Eviction:** If a claim with `2,000 sat` can satisfy an incoming requirement of `1,800 sat`, it is chosen over a claim with `500,000 sat`.

#### Disadvantages & Trade-offs
* **Algorithmic Overhead:** Exact subset minimization reduces to the NP-hard Set Cover / Knapsack decision problem. While $N$ (the number of conflicting claims) is bounded by active package claims, branch-and-bound or dynamic programming introduces complexity.
* **Increased Code Surface:** Significant branching logic is added to an otherwise linear scanning routine.

---

### Approach 3: Urgency / Deadline-Aware Displacement

#### System Design & Mechanics
1. Correlate each `ClaimId` with its confirmation deadline (e.g., HTLC expiry block height).
2. Sort conflicting groups by deadline descending: claims with deadlines far in the future are displaced first; claims close to expiration are protected.

#### Advantages
* **Optimal Safety for Channel State:** HTLCs near expiration are at risk of counterparty settlement or penalty timeouts. Protecting urgent claims prevents fund loss.

#### Disadvantages & Trade-offs
* **Interface Boundary Leak:** [`WalletSource`](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/src/branch/main/lightning/src/util/wallet_utils.rs#L434) and [`CoinSelectionSource`](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/src/branch/main/lightning/src/util/wallet_utils.rs#L352) are designed as generic wallet abstractions (intended to integrate with Bitcoin Core, BDK, or external signers). Injecting Lightning-specific deadline timers into the wallet trait violates abstraction boundaries.
* **Transience of Lock Table:** The lock table is in-memory only and re-initialized across restarts. Maintaining deadline metadata across derived batch IDs adds state synchronization fragility.

---

## 4. Proposed Implementation Architecture

Based on LDK's modularity requirements and the localized nature of `wallet_utils.rs`, **Approach 1** offers the highest reliability-to-complexity ratio, with the option to introduce cardinality tie-breaking (Approach 2 refinement) if multiple single claims can each satisfy the target.

### 4.1 Data Pipeline within `select_confirmed_utxos_internal`

1. **Phase 1: Aggregate Claim Effective Values** — Compute total effective value per conflicting `ClaimId`.
2. **Phase 2: Hierarchical Sorting** — Sort by `(IsConflicting, Reverse(GroupTotalValue), ClaimId, EffectiveValue)`.
3. **Phase 3: Greedy Selection** — Accumulate coins; evict smaller coins if weight budget exceeded, update lock table with new `ClaimId` allocations.

### 4.2 Algorithm Specification

```rust
// 1. Identify conflicting claim ownership without allocating extra state for unreserved coins
let conflicting_claim_id = |utxo: &Utxo| {
    locked_utxos
        .get(&utxo.outpoint)
        .and_then(|utxo_claim_id| *utxo_claim_id)
        .filter(|utxo_claim_id| Some(*utxo_claim_id) != claim_id)
};

// 2. Pre-compute total effective purchasing power per conflicting claim
let mut conflicting_claim_values = new_hash_map();
for (utxo, fee_to_spend_utxo) in &eligible_utxos {
    if let Some(c_id) = conflicting_claim_id(utxo) {
        let effective_value = utxo.output.value - *fee_to_spend_utxo;
        conflicting_claim_values
            .entry(c_id)
            .and_modify(|val| *val += effective_value)
            .or_insert(effective_value);
    }
}

// 3. Composite Sort:
//    - Primary: Unlocked coins first (None maps before Some)
//    - Secondary: Conflicting claims with highest total effective value first
//    - Tertiary: Stable tie-breaking via ClaimId bytes
//    - Quaternary: Ascending order of individual coin effective value within group
eligible_utxos.sort_unstable_by_key(|(utxo, fee_to_spend_utxo)| {
    let effective_value = utxo.output.value - *fee_to_spend_utxo;
    let conflicting_claim = conflicting_claim_id(utxo).map(|c_id| {
        (
            core::cmp::Reverse(*conflicting_claim_values.get(&c_id).unwrap()),
            c_id.0,
        )
    });
    (conflicting_claim, effective_value)
});
```

---

## 5. Verification & Testing Strategy

### 5.1 Unit Reproduction Test
A dedicated unit test module (`conflicting_claim_selection_tests`) in [`lightning/src/util/wallet_utils.rs`](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/src/branch/main/lightning/src/util/wallet_utils.rs) validates the displacement boundary:
* **Topology:**
  * Outpoint 0: `1,000 sat` (Claim A)
  * Outpoint 1: `1,100 sat` (Claim B)
  * Outpoint 2: `3,000 sat` (Claim A)
* **Target:** `1,800 sat` (Claim C).
* **Validation:** Asserts `selection.confirmed_utxos.len() == 1` and matches Outpoint 2.

### 5.2 Regression Coverage
* Execute full workspace verification:
  ```bash
  cargo +1.75.0 test -p lightning --lib util::wallet_utils
  cargo +1.75.0 test -p lightning --lib events::bump_transaction
  ```
* Ensure no regressions across existing coin selection tests:
  * `failed_selection_releases_newly_locked_utxos`
  * `failed_forced_selection_restores_previous_claim`
* Enforce formatting standard:
  ```bash
  cargo +1.85.0 fmt --all
  ```

---

## 6. Open Review Points for Discussion

1. **Threshold Sizing (Tie-Breaking Refinement):**  
   If multiple single claims can each satisfy the incoming target independently (e.g., Claim A has `5,000 sat` and Claim B has `2,500 sat` for a `2,000 sat` target), should the selection prefer the claim closest to the target to preserve the larger reservoir, or remain strictly greedy?
2. **Batch Claim ID Granularity:**  
   Confirm that keying on derived claim IDs from `bump_transaction` remains the desired granularity, as opposed to aggregating across parent channel identifiers.
