# SnarkSide | Encrypted Perpetual Futures Exchange

SnarkSide is a zero-knowledge (ZK) intent-based perpetual futures exchange. The protocol allows users to open, settle, and liquidate leveraged positions on-chain without exposing any trade-related metadata, including order size, direction, execution intent, liquidation thresholds, or wallet identity.

This repository serves as the canonical specification of the SnarkSide protocol stack, cryptographic systems, MPC engine, and zk-circuit libraries. SnarkSide has been in stealth development since Q4 2022, with over two years of iteration, simulation, and field-level circuit optimization. This README captures architectural decisions, proof structures, circuit code snippets, backend runtime design, and liquidity models — with emphasis on non-trivial constraint systems, state transition correctness, and MEV-resistance mechanics.

---

## Table of Contents

* [Architecture Overview](#architecture-overview)
* [Design Principles](#design-principles)
* [ZK Circuits](#zk-circuits)

  * [Trade Intent Commitments](#trade-intent-commitments)
  * [CipherVault Liquidation Circuit](#ciphervault-liquidation-circuit)
  * [Merkle Path Validations](#merkle-path-validations)
  * [Nullifier Constraints](#nullifier-constraints)
* [DarkMatch Engine](#darkmatch-engine)

  * [Intent Aggregation](#intent-aggregation)
  * [Matching Algorithm](#matching-algorithm)
* [Oblivion Pool](#oblivion-pool)
* [zkOracle System](#zkoracle-system)
* [zkWallet & Gas Abstraction](#zkwallet--gas-abstraction)
* [State Transition Guarantees](#state-transition-guarantees)
* [Concurrency & Replay Protection](#concurrency--replay-protection)
* [Repository Overview](#repository-overview)
* [Development Milestones](#development-milestones)
* [Performance Benchmarks](#performance-benchmarks)
* [Security Model](#security-model)
* [License](#license)
* [Contact](#contact)

---

## Architecture Overview

SnarkSide separates visible trading logic into a five-layer cryptographically enforced stack:

1. **Encrypted Trade Intents** — Users submit zk-wrapped commitments to trade parameters.
2. **DarkMatch Engine** — Off-chain MPC coordinator aggregates and matches opposing flows.
3. **CipherVault** — Zero-knowledge margin vault utilizing UTXO-based accounting and stealth deposit nullifiers.
4. **Oblivion Pool** — Encrypted vAMM funding mechanism calculating OI skew without position exposure.
5. **Settlement Layer** — ZK verification of matched commitments and nullifier enforcement to prevent replay.

Each layer operates with strict isolation boundaries and selective disclosure. Nothing enters the chain without zero-knowledge proof of constraint satisfaction.

---

## Design Principles

* All user interactions are encrypted at origin.
* No mempool-exposed metadata; bundle relays abstract execution.
* Liquidation conditions provable without identifying counterparties.
* Matching system does not require trust — verification is non-interactive.
* No slippage leakage, order clustering, or volume inference.
* MEV is rendered infeasible via intent-level encryption and commitment masking.

---

## ZK Circuits

### Trade Intent Commitments

Each intent is a zkSNARK-valid proof of intent metadata:

```ts
component TradeIntent = template() {
    signal input hash;
    signal input direction; // 0 = short, 1 = long
    signal input leverage;
    signal input slippageBound;
    signal input expiry;
    signal input salt;

    signal output commitment;

    component poseidon = Poseidon(6);
    poseidon.inputs[0] <== hash;
    poseidon.inputs[1] <== direction;
    poseidon.inputs[2] <== leverage;
    poseidon.inputs[3] <== slippageBound;
    poseidon.inputs[4] <== expiry;
    poseidon.inputs[5] <== salt;

    commitment <== poseidon.output;
}
```

### CipherVault Liquidation Circuit

```ts
template Liquidation() {
    signal input entry_price;
    signal input current_price;
    signal input leverage;
    signal input margin;
    signal input liquidation_threshold;
    signal output is_liquidatable;

    component delta = Subtractor();
    delta.in[0] <== entry_price;
    delta.in[1] <== current_price;

    signal loss;
    loss <== delta.out * leverage;

    is_liquidatable <== loss > liquidation_threshold;
}
```

This is embedded into a larger state proof circuit and batched with oracle data.

### Merkle Path Validations

All position updates require valid Merkle membership proof:

```ts
component MerkleProof = MerkleTreeInclusionProof(depth);
MerkleProof.root <== merkle_root;
MerkleProof.leaf <== leaf_commitment;
MerkleProof.pathElements <== path_elements;
MerkleProof.pathIndices <== indices;
```

### Nullifier Constraints

Replay resistance is enforced via poseidon nullifiers:

```ts
signal input user_key;
signal input note_id;
signal output nullifier;

nullifier <== Poseidon(user_key, note_id);
```

---

## DarkMatch Engine

Implemented in Rust + Node.js hybrid runtime. Acts as:

* Trade aggregator
* Commitment generator
* Slippage checker
* Proof packager

### Intent Aggregation

Encrypted bundles are classified by expiry, then flattened:

```ts
matchQueue.push({
    commitment: intent.commitment,
    expiry: intent.expiry,
    side: intent.direction
});
```

### Matching Algorithm

```ts
function matchOpposites(queue) {
    for each pair (a, b) {
        if a.side != b.side && withinSlippage(a, b) {
            settlePair(a, b);
        }
    }
}
```

---

## Oblivion Pool

Encrypted vAMM driven by:

* zk funding computation
* LP stealth tokens
* OI commitments for skew
* Private LP rebalance proofs

Funding update circuit:

```ts
signal long_OI;
signal short_OI;
signal funding_rate;
funding_rate <== (long_OI - short_OI) / (long_OI + short_OI);
```

---

## zkOracle System

Uses commit-reveal proof-of-truth:

1. Oracle signs off-chain round ID
2. Post commit hash on-chain
3. Reveal + ZK proof of price timestamp correlation

---

## zkWallet & Gas Abstraction

Wallet SDK allows:

* ZK identity key
* Ephemeral sender
* Gas abstraction via paymaster

No trade or wallet correlation occurs on-chain.

---

## State Transition Guarantees

All state transitions are:

* Collision resistant
* Nullifier-bound
* Merkle-tree indexable
* Cryptographically final

---

## Concurrency & Replay Protection

Concurrency locks are applied per nullifier.
Replay is infeasible due to poseidon-encoded context hashes.

---

## Repository Overview

* `snarkside-circuits`
* `snarkside-core`
* `snarkside-matcher`
* `snarkside-contracts`
* `wallet-sdk`
* `zkoracle-wrapper`
* `oblivion-sim`

---

## Development Milestones

* Internal testnet: passed
* zk relayer: complete
* bundler rotation: ongoing
* audit: scheduled Q3
* launch: TBD

---

## Performance Benchmarks

* zk intent proof: \~220ms (Groth16)
* UTXO validation: \~470ms
* Batch settlement: 80 txs/block
* Oracle latency: <300ms (Pyth, commit-reveal mode)

---

## Security Model

* Circuit integrity guaranteed by constraint satisfaction
* Data leakage eliminated at mempool & transaction level
* Oracle abuse mitigated via round anchoring
* Full audit suite available in `/audits`

---

## License

MIT for code. GPL for SNARK runtime components. zkSNARK use licensed under contribution agreement for commercial reuse.

---

## Contact

[labs@snarkside.app](mailto:labs@snarkside.app)

For audits, integration, or relayer access requests.

---
