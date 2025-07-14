<pre>
OPI: <to be assigned>
Title: <Your Operation Title Here>
Author: <Your Name or Team> <contact@example.com>
Discussions-To: <URL to issue, forum, or chat>
Status: Draft
Type: Standards Track
Created: <YYYY-MM-DD>
</pre>

---

## Abstract

Concise summary of what the operation does and why it exists. It should describe the goal of the operation, the mechanism it introduces, and the context in which it's intended to be used. Limit this to a few sentences.

---

## Copyright

This OPI is released under the MIT License.

---

## Motivation

Explain why this OPI is necessary. What problem does it solve? What use cases does it enable? Reference known limitations or community needs that justify the introduction of this new operation.

---

## Specification

### Transaction Structure

Describe the required transaction structure. Include a table with inputs and outputs, and how they relate to the operation:

| Component | Description                                 |
| --------- | ------------------------------------------- |
| Input[0]  | Description of sender or operation source   |
| Output[0] | OP_RETURN output with the operation JSON    |
| Output[1] | Recipient output, burn, or execution output |
| Output[n] | Optional change or additional outputs       |

### OP_RETURN Payload

Show the JSON schema expected in the `OP_RETURN`, including required fields:

```json
{
  "p": "brc-20",
  "op": "<your_op>",
  "...": "..."
}
```

---

## Indexer Validation Rules

Define what rules an indexer must follow to validate and process this operation. Make sure they are deterministic, testable, and protocol-safe.

---

## Rationale

Justify the design decisions. Why is this structure chosen? Why are specific inputs or outputs necessary? Explain the trade-offs made, and what alternative designs were rejected.

---

## Backwards Compatibility

State whether this OPI breaks any existing behavior or introduces new semantics. If it is backwards-compatible, explain how. If not, explain the upgrade path.

---

## Reference Implementation

If applicable, provide links to a working implementation or example code (e.g. indexer modules, parsing logic, wallet support):

- **Repo**: <repo-url>
- **Branch**: <branch-name>
- **Examples**: Include PSBTs, test cases, transaction hex, etc.

---

## Security Considerations

List any known risks or attack vectors that this operation mitigates or introduces. Explain how replay protection, spoofing, misuse, or ambiguity are handled.
