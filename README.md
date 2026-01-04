# Entity Resolution Spec

a deterministic, minimal schema for resolving identity across people, organizations, and addresses.

designed for analytical work in legal, compliance, audit, and investigative contexts.

## Overview

This repository defines a structured approach to entity resolution that prioritizes:

- **Determinism**: Same inputs always produce same outputs
- **Explainability**: Every resolution decision is traceable to source records
- **Reproducibility**: Results can be verified and audited
- **Conservative defaults**: Ambiguity is preserved rather than collapsed


## What This Schema Does

- Resolves identity across three entity types: people, organizations, and addresses
- Provides deterministic matching rules with clear precedence
- Assigns confidence tiers (exact, strong, weak) based on evidence quality
- Preserves ambiguity when evidence is insufficient
- Maintains complete lineage from resolved entities back to source records
- Supports human review workflows for contested or ambiguous resolutions

## Repository Structure

```
entity-resolution-spec/
├── docs/
│   ├── DESIGN.md         # Full design document
│   ├── RULES.md          # Resolution rules reference
│   └── TESTING.md        # Testing philosophy and guide
├── schema/
│   ├── entity/
│   │   ├── person.yaml       # Person entity schema
│   │   ├── organization.yaml # Organization entity schema
│   │   └── address.yaml      # Address entity schema
│   ├── resolution/
│   │   ├── rules.yaml        # Resolution rule definitions
│   │   ├── tiers.yaml        # Confidence tier semantics
│   │   └── conflicts.yaml    # Conflict handling
│   └── lineage/
│       ├── source_record.yaml    # Source record tracking
│       ├── transformation.yaml   # Data transformation lineage
│       └── decision.yaml         # Resolution decision audit
├── normalization/
│   ├── address.yaml      # Address normalization rules
│   ├── name.yaml         # Name normalization rules
│   └── identifier.yaml   # Identifier normalization rules
├── tests/
│   ├── golden/           # Behavioral contract tests
│   ├── negative/         # Must-not-match tests
│   ├── regression/       # Historical bug tests
│   └── invariants/       # System property tests
└── README.md
```

## Confidence Tiers

### Exact

Resolution based on high-specificity identifier match (SSN, EIN, passport number).
- Downstream systems may use for compliance filings and reporting
- False positives at this tier are treated as defects

### Strong

Resolution based on multiple corroborating attributes with no conflicts.
- Suitable for most analytical workflows
- Human review recommended for high-stakes decisions

### Weak

Partial attribute overlap; insufficient for confident resolution.
- Candidates for human review, not resolved entities
- Must not be treated as resolved without explicit human action

## Design Philosophy

### False Positives Are Defects

This system is designed for contexts where incorrectly merging two distinct entities has serious consequences. A single false positive in a legal filing or compliance report can contaminate evidence and damage trust.

The design errs toward under-merging. When in doubt, entities remain separate.

### Ambiguity Is Preserved

When evidence is insufficient or conflicting, the system preserves uncertainty rather than forcing resolution. Ambiguous cases are routed to human review with complete evidence documentation.

Accumulating weak signals does not produce strong confidence. Only new evidence can change tier assignment.

### Complete Lineage

Every resolved entity traces back to source records. Every transformation is documented. Every resolution decision is auditable.

This supports legal discovery, regulatory audit, and error correction.

### Deterministic and Reproducible

Given the same inputs, the system produces the same outputs. Always. There is no randomness, no time-dependent logic, no external state that affects resolution.

This enables testing, audit, and trust.

## Getting Started

1. Read the [Design Document](docs/DESIGN.md) for full context
2. Review [entity schemas](schema/entity/) to understand data structures
3. Study [resolution rules](schema/resolution/rules.yaml) to understand matching logic
4. Examine [test fixtures](tests/) to see expected behavior

## Implementing This Schema

This repository defines schemas and specifications, not executable code. Implementation requires:

1. A data store capable of representing the entity and lineage schemas
2. A processing pipeline that applies normalization rules
3. A resolution engine that evaluates matching rules
4. A review interface for handling ambiguous cases
5. A test harness that verifies behavior against fixtures

The schemas are defined in YAML for language-agnostic consumption. Implementations may use any language or platform that can faithfully represent the semantics.

## Testing
 The test suite serves as both verification and specification.

- **Golden tests**: Fixed inputs with expected outputs (behavioral contracts)
- **Negative tests**: Cases where resolution must not occur
- **Regression tests**: Historical bugs encoded as tests
- **Invariant tests**: System-wide properties (determinism, symmetry, monotonicity)

See [Testing Guide](docs/TESTING.md) for details.

## Contributing

Contributions should:

1. Maintain the conservative, deterministic nature of the system
2. Include tests for any new rules or behaviors
3. Document rationale for changes
4. Avoid introducing probabilistic or ML-based components
5. Preserve complete lineage for all data transformations

## License

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for full license text.

contact founders@audt.ai for inquiries. 
