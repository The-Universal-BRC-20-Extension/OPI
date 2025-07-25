# Universal BRC-20 OPIs

This folder contains Operation Proposal Improvements (OPIs) for the [Universal BRC-20 Extension](https://github.com/The-Universal-BRC-20-Extension/The-Universal-BRC-20-Extension).  
OPIs define new operations, migration patterns, and extension logic on top of the Universal BRC-20 protocol, and are intended to evolve the ecosystem in a structured, forward-compatible way.

---

## What is an OPI?

An **OPI (Operation Proposal Improvement)** is a formal specification that:

- Introduces a new `"op"` value in BRC-20 operation JSON (e.g. `no_return`, `swap`,...)
- Defines the required transaction structure and validation rules
- Describes how indexers and tools should interpret the operation
- Maintains backwards compatibility with Bitcoin consensus

OPIs must conform to the principles of the Universal BRC-20 Extension: **explicitness, modularity, auditability, and composability**.

---

## Folder Structure

Each OPI resides in its own folder under `OPI` [The-Universal-BRC-20-Extension/OPI: Operation Protocol Improvements. A collaborative space to propose, track, and refine protocol operations.](https://github.com/The-Universal-BRC-20-Extension/OPI):

```
OPI/
├── opi-000-no_return/
│   ├── OPI-000-no_return.md        # Main specification document
│   ├── examples/                    # PSBTs, tx hex, validation test vectors
│   └── reference-implementation/    # Optional: indexer logic, parsers
├── opi-001-.../
│   └── ...
└── README.md
```

---

## How to Propose an OPI

1. Fork the repo.
2. Copy `template/OPI-template.md` as a starting point.
3. Write your new spec using the `/opis/opi-###-<name>/` structure.
4. Include at least one example transaction or PSBT.
5. Submit a Pull Request and discuss with the community ([Telegram](https://t.me/theblacknode)).

---

## Status & Governance

- `Status: Draft` → open for discussion
- `Status: Final` → accepted and implemented
- `Status: Withdrawn` → deprecated or replaced

Changes to an OPI’s status are governed by off-chain discussion until a formal governance process is established.

---

## References

- [The Universal BRC-20 Extension (Core Protocol)](https://github.com/The-Universal-BRC-20-Extension/The-Universal-BRC-20-Extension)
- [Simplicity Indexer Reference](https://github.com/The-Universal-BRC-20-Extension/simplicity)

---

## Contributing

- Follow the format and tone of the provided OPI template.
- Be precise, deterministic, and testable.
- Do not break protocol semantics unless necessary, and always specify migration paths.

Let's build the future together—block by block.
