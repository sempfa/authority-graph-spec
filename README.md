# Architecting Autonomy: Authority Graph Specification

Formal specification for encoding authority as a designable, composable structure in autonomous systems.

## What This Is

The Authority Graph is an architectural primitive for governing autonomous systems. It encodes decision rights, not permissions, not capabilities, not policies, as the foundational structure through which governance is designed, composed, and enforced.

This repository contains the implementable specification. It is the reference companion to the [Authority Graph Formalization](https://architectingautonomy.substack.com/) companion paper in the Architecting Autonomy series.

## Why It Exists

Every major multi-agent framework today (LangGraph, CrewAI, AutoGen, OpenAI Agents SDK, MetaGPT) satisfies zero of four governance requirements:

1. **Authority as a primitive**: every component answers: what am I authorised to decide, under what conditions, who granted this, when does it expire?
2. **Composition contracts**: when two agents interact, whose rules apply?
3. **Legibility**: every decision is attributable to a specific authority, traceable through every boundary it crossed
4. **Enforcement separation**: the governance layer is architecturally separate from the coordination layer

The composition contract layer does not exist anywhere yet as a first-class primitive. This specification addresses that gap.

## Contents

```
authority-graph/
  spec.md              — The full specification (schemas, protocols, validation rules)
  examples/
    minimal.yaml       — Hello world: 3-agent system with delegation and composition
    refund-fraud.yaml  — The series' primary scenario: payment + fraud domains
```

## Quick Start

1. Read `authority-graph/spec.md` — Appendix C (Implementation Notes) has a recommended starting order
2. Study `authority-graph/examples/minimal.yaml` to see every element in use
3. Map your system following the eight-step methodology in the companion paper

## The Specification Covers

| Section | What it defines |
|---|---|
| Authority Unit | Schema for a decision right with six properties (explicit, scoped, enforceable, delegable, observable, terminable) |
| Expressions | Tree-structured evaluable predicates with nesting, field references, and temporal operators |
| Edges | Four relationship types: delegation, composition, precedence, scope overlap (diagnostic) |
| Authority Domains | Subgraphs with shared constitutional basis, domain contracts, export policies |
| Composition Contract | Governance instrument between domains: primitives, invariants, conflict resolution, sovereign retention |
| Constitutional Hierarchy | Three-tier scaling: constitution → domain contracts → pairwise contracts, with evaluation precedence |
| Monotonic Reduction | Protocol for handling uncertainty: scope contracts when conditions are unconfirmed |
| Graph Analysis | Functions for overlap detection, gap detection, delegation depth, compositional reachability |
| Arbitration Interface | Request/response between the graph and the enforcement layer (control-surface band) |
| Validity Rules | Eight rules a graph must satisfy before deployment |

## Series Context

This specification is part of the [Architecting Autonomy](https://architectingautonomy.substack.com/) series:

- **Article 8: The Unit of Authority**: introduces authority as a first-class primitive
- **Article 9: Authority Composition**: establishes composition primitives and contracts
- **Article 10: Legibility as Structural Requirement**: defines what makes authority readable
- **Article 11: Governance at Machine Speed**: proves that constraint must precede cognition
- **Authority Graph Formalization** (companion paper): translates the arc's claims into architectural blueprint

The specification makes the companion paper implementable. The companion explains why and how. The specification says exactly what.

## Version

**Current:** 0.1.0 (Draft)

This specification follows semantic versioning. See [CHANGELOG.md](authority-graph/CHANGELOG.md) for version history.

## Contributing

This specification is a draft. If you are implementing an authority graph, building governance for autonomous systems, or have feedback on the schemas:

- **Open an issue** for gaps, ambiguities, or missing types
- **Propose extensions** for domain-specific patterns
- **Share worked examples** from your system (with appropriate anonymisation)

The specification evolves through the same principle it defines: staged candidates, verified before promotion, previous versions retained. Changes follow semantic versioning.

## License

This work is licensed under [CC BY 4.0](LICENSE). You are free to share and adapt the specification with attribution to the Architecting Autonomy series.

## Author

[Aaron Sempf](https://architectingautonomy.substack.com/) — Architecting Autonomy series
