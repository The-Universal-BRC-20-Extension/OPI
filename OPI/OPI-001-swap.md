# OPI: 1

**Title**: `swap` Operation for Universal BRC-20 Extension
**Author**: LOL (lol@tx.lol)
**Discussions-To**: [https://t.me/theblacknode](https://t.me/theblacknode)  
**Status**: Draft
**Type**: Standards Track
**Created**: 2025-25-07

---

## Abstract

This document specifies `swap`, a decentralized asset exchange operation architected for the Universal BRC-20 Extension. It introduces a virtual Automated Market Maker (AMM) whose state is derived deterministically from `OP_RETURN` messages, obviating the need for custodial bridges or complex Layer 2 constructions. The protocol defines two atomic functions: `init` for providing time-locked liquidity and `exe` for swap execution, governed by a "Total Lock-up Model" to foster deep, stable markets. This operation is designed to serve as the foundational, composable liquidity layer for a sovereign financial system on Bitcoin.

---

## Copyright

This OPI is released under the MIT License.

---

## Motivation

Expressive finance on Bitcoin has historically been constrained by a false choice between custodial risk, Layer 2s, and programming language complexity. OPI-1 is motivated by a different vision: that sophisticated economic systems can emerge directly by using Bitcoin's elegant UTXO model to drive a clear, address-based accounting ledger, all interpreted by a deterministic off-chain state machine.

This specification is necessary to:

1.  **Establish a Canon for Trustless Exchange:** To provide a single, unambiguous, and deterministic rule set for an L1-native AMM, eliminating the possibility of state divergence among network participants.
2.  **Cure the Sickness of Mercenary Liquidity:** To introduce the "Total Lock-up Model" as a structural antidote to the transient, profit-seeking liquidity that plagues traditional AMMs, thereby creating markets with deep, predictable stability.
3.  **Mandate Aggregated Liquidity:** To solve the problem of liquidity fragmentation at the protocol level through a canonical ordering of token pairs, ensuring all market depth for a given pair is unified.

> Forge a Foundational Economic Primitive: To create a base layer of liquidity and a decentralized price oracle that can be trusted by a new generation of composable OPIs (eg. for lending, derivatives, and autonomous agents), establishing a truly sovereign financial stack.

---

## Specification

### Transaction Structure

A valid `swap` transaction must adhere to the following structure:

| Component     | Description                                                                                                                           |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| **Input[0]**  | The first input UTXO. Its controlling address is the unique, verifiable identifier of the operator.                                   |
| **Input[n+]** | Additional inputs as needed for fees                                                                                                  |
| **Output[0]** | An `OP_RETURN` output containing the JSON payload (`{ "p": "brc-20", "op": "swap", "...", ... }`) that dictates the state transition. |
| **Output[n]** | Optional change output and other standard transaction components.                                                                     |

> Input ordering is essential and must be preserved.  
> Execution semantics follow the Universal model, where operations bind to subsequent outputs.

---

### OP_RETURN Payloads (Output[0])

The operation is a JSON object inscribed within the `OP_RETURN` output.

- **Value**: `0 sats`
- **Script**: The payload is a UTF-8 encoded JSON string, which is then hex-encoded into the `OP_RETURN` script.

#### **1. Liquidity Provision: `init`**

A time-bound declaration of commitment to a liquidity pool.

- **Example Payload**:
  ```json
  {
    "p": "brc-20",
    "op": "swap",
    "init": "lol,wtf",
    "amt": "101000",
    "lock": "101"
  }
  ```

| Field  | Type   | Description                                                                                            |
| :----- | :----- | :----------------------------------------------------------------------------------------------------- |
| `init` | String | The token pair `<ticker_provided>,<ticker_sought>`, defining the direction of provision.               |
| `amt`  | String | The amount of `ticker_provided` being locked.                                                          |
| `lock` | String | The mandatory lock-up duration in Bitcoin blocks. The position remains locked for this period (`≥ 1`). |

#### **2. Swap Execution: `exe`**

A request to exchange assets against an active liquidity pool.

- **Example Payload**:
  ```json
  {
    "p": "brc-20",
    "op": "swap",
    "exe": "wtf,lol",
    "amt": "21",
    "slip": "0.5"
  }
  ```

| Field  | Type   | Description                                                                                        |
| :----- | :----- | :------------------------------------------------------------------------------------------------- |
| `exe`  | String | The swap direction `<ticker_given>,<ticker_received>`.                                             |
| `amt`  | String | The amount of `ticker_given` offered for the swap.                                                 |
| `slip` | String | Maximum slippage tolerance, as a (decimal) percentage string (`0 < slip < 100`). Example: `"0.5"`. |

---

## Indexer Validation Rules

Indexers that support this OPI **MUST** validate:

1.  **Chronological Order**: The transaction is processed strictly in the order of its confirmation (first by block height, then by transaction index within the block).
2.  **Operator Identity**: The operation is assigned to the address that controlled `Input[0]`.
3.  **Canonical Pool ID**: The pool identity is derived from the alphabetical ordering of its tickers (e.g., `lol-wtf`) to prevent liquidity fragmentation.
4.  **Pre-Execution State**: The operator has a sufficient available balance, and for `exe` operations, the target pool is `active` (contains liquidity for both assets).
5.  **Mandatory Partial Fill**: An `exe` operation that exceeds the `slip` tolerance is partially filled to the maximum amount possible that honors the slippage constraint; it is **never rejected** for this reason.
6.  **Atomic State Transition**: The entire state change for an operation (debiting/crediting the operator, updating all affected LPs, and adjusting pool reserves) is executed as a single, indivisible transaction.
7.  **Lock Expiration Priority**: All expired `lock` positions are processed and returned to their owners' main balances at the beginning of a new block's processing, before any other operations in that block are handled.

---

## Rationale

This protocol's design is an act of principled minimalism. Every choice is a deliberate rejection of complexity in favor of sovereignty and cryptographic truth.

- **The Bifurcated Security Model:** Security is not inherited; it is earned through sound architecture. Bitcoin provides the **Immutable Log of Intent**, a perfectly ordered and uncensorable record of commands. The execution logic, however, lives off-chain. Its security is not derived from L1 enforcement but from the **mathematical certainty that any honest participant running the open-source logic on the public L1 log will compute the exact same state.** Security is therefore an emergent property of radical transparency and verifiable computation.

- **Virtual State as a Principle:** State is not a chain-level concern; it is a computational result. On-chain systems are rigid, expensive, and have a large attack surface. By maintaining state virtually, we create a system that is infinitely more agile, efficient, and resilient.

- **The Total Lock-up Model as Economic Philosophy:** A declaration of war on the parasitic capital of DeFi 1.0. We do not seek liquidity; we seek commitment. The `lock` mandates skin in the game and align the incentives of liquidity providers with the long-term health of the protocol, creating a system built on investment, not transient opportunity.

- **UTXO-Based Identity:** There is no need for a separate, fallible identity system. Bitcoin already provides one. Tying identity to a UTXO is the most native, elegant, and Sybil-resistant solution possible.
