# OPI: 0

**Title**: `no_return` Operation for Universal BRC-20 Extension  
**Author**: Blacknode <blacknodebtc@protonmail.com>  
**Discussions-To**: [https://t.me/theblacknode](https://t.me/theblacknode)  
**Status**: Draft  
**Type**: Standards Track
**Created**: 2025-07-07

---

## Abstract

This document specifies `no_return`, the first Operation Proposal Improvement (OPI-000) under the Universal BRC-20 Extension. It defines a canonical, non-reusable operation that enables users to migrate legacy BRC-20 balances—originating from witness inscriptions—into the Universal protocol.

The `no_return` structure combines a legacy `transfer` inscription with an `OP_RETURN` semantic marker. The inscription is intentionally sent to Satoshi’s address, and the tokens are assigned to the address that controlled the input containing the inscription. This mechanism provides a one-way, final transition into the Universal system, ensuring safety, determinism, and clean state inheritance.

> This specification assumes familiarity with the Universal BRC-20 execution model. See [The Universal BRC-20 Extension](https://github.com/The-Universal-BRC-20-Extension/The-Universal-BRC-20-Extension) for background.

---

## Copyright

This OPI is released under the MIT License.

---

## Motivation

BRC-20 inscriptions, originally deployed using Ordinals, encode their logic in witness data and associate token state with specific satoshis. This approach enabled a breakthrough in fungible token design on Bitcoin, but its reliance on sat-position tracking introduces challenges for transaction design, interoperability, and composability.

The Universal BRC-20 Extension introduces an explicit, operation-driven model based on `OP_RETURN` outputs and address-bound state. It eliminates ambiguity around token ownership and transfer semantics, and enables scalable functionality such as batched operations, composable swaps, and delegation.

The `no_return` operation enables users to permanently migrate legacy BRC-20 tokens into this modern, declarative model. It achieves this through:

- **One-way migration**: The inscription is burned by sending it to Satoshi’s address.
- **Explicit intent**: A semantic `OP_RETURN` marker declares the operation as a Universal migration.
- **State reassignment**: The sender (controller of the inscription input) is credited in the Universal protocol.

This approach preserves the legacy asset’s token identity while transitioning its logic to a safer, composable future.

---

## Specification

### Transaction Structure

A valid `no_return` transaction must adhere to the following structure:

| Component     | Description                                                                                   |
| ------------- | --------------------------------------------------------------------------------------------- |
| **Input[0]**  | SegWit input (P2TR, P2WPKH, or P2WSH) containing a valid legacy BRC-20 `transfer` inscription |
| **Input[n+]** | Additional inputs as needed for fees                                                          |
| **Output[0]** | `OP_RETURN` output containing `{ "p": "brc-20", "op": "no_return" }`                          |
| **Output[1]** | Output sending the inscription to Satoshi’s address (`1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa`)    |
| **Output[2]** | Optional change output                                                                        |

> Output ordering is essential and must be preserved.  
> Execution semantics follow the Universal model, where operations bind to subsequent outputs.

---

### OP_RETURN Marker (Output[0])

- **Value**: `0 sats`
- **Script**:
  ```
  OP_RETURN 7b2270223a226272632d3230222c226f70223a226e6f5f72657475726e227d
  ```
  This is the hex encoding of:
  ```json
  { "p": "brc-20", "op": "no_return" }
  ```

---

### Legacy Inscription (Input[0] Witness)

- Must follow the Ordinals envelope format, such as:
  ```json
  {
    "p": "brc-20",
    "op": "transfer",
    "tick": "ORDI",
    "amt": "100"
  }
  ```
- The inscription must be valid under the BRC-20 legacy rules.
- It must be transferred to Output[1] (Satoshi address) in accordance with Ordinals inscription rules, ensuring its destruction.

---

## Indexer Validation Rules

Indexers that support this OPI MUST validate:

1. The transaction includes **exactly one** `OP_RETURN` output matching the expected `no_return` payload.
2. Input[0] spends a UTXO containing a valid BRC-20 `transfer` inscription.
3. The inscription is sent to the Satoshi address (Output[1]), ensuring it is burned.
4. The `amt` and `tick` in the inscription are parsed and validated for correctness.
5. The Universal BRC-20 balance is assigned to the **address that controlled Input[0]**.

---

## Rationale

The `no_return` operation establishes a clean transition path from legacy BRC-20 to Universal, without introducing ambiguity or replay risks.

- **Non-reusability**: The inscription is burned and cannot be reinterpreted or replayed.
- **Explicit semantics**: The `OP_RETURN` acts as a declarative, prunable instruction visible at surface level.
- **Modular indexer logic**: Indexers process `no_return` via a deterministic rule set, requiring no sat-tracking heuristics.

By enforcing inscription destruction and binding token reassignment to the sender, `no_return` maintains security guarantees while enabling future composability under the Universal standard.

---

## Backwards Compatibility

This OPI is **fully non-breaking**:

- **Legacy indexers** that do not recognize `OP_RETURN` will see the inscription as burned and will not misinterpret the transaction.
- **Universal-compliant indexers** will recognize the `no_return` operation and update the sender’s balance under the new model.

This allows wallet and protocol implementers to opt in at their own pace.

---

## Reference Implementation

The Simplicity Indexer will implement `no_return` validation logic in the following repository:

- **Repo**: [https://github.com/blacknode/simplicity](https://github.com/blacknode/simplicity)
- **Branch**: `main`
- **Examples**: `examples/opi-0/` directory will include PSBT samples and transaction templates to simplify the integration.

---

## Security Considerations

- **Replay Prevention**: The inscription is sent to Satoshi and can no longer be reused in any context.
- **Recipient Guarantee**: Token state is deterministically reassigned to the address controlling Input[0].
- **Finality**: Once processed, the transfer is one-way and irreversible.
