# Test Suite

This directory contains the test fixtures and specifications for the entity resolution schema.

## Directory Structure

```
tests/
├── golden/           # Behavioral contract tests
│   ├── person/
│   │   ├── exact/    # Exact tier match tests
│   │   ├── strong/   # Strong tier match tests
│   │   └── weak/     # Weak tier match tests
│   ├── organization/
│   │   ├── exact/
│   │   ├── strong/
│   │   └── weak/
│   └── address/
│       ├── exact/
│       ├── strong/
│       └── weak/
├── negative/         # Must-not-match tests
│   ├── person/
│   ├── organization/
│   ├── address/
│   └── ambiguity/    # Conflict detection tests
├── regression/       # Historical bug tests
└── invariants/       # System property tests
```

## Test Categories

### Golden Tests

Fixed inputs with deterministic expected outputs. These are the primary behavioral contracts for the system.

- Located in `tests/golden/`
- Organized by entity type and confidence tier
- Each test specifies input records and expected resolution

### Negative Tests

Explicit tests for cases where resolution must NOT occur.

- Located in `tests/negative/`
- Verify that similar-but-distinct entities remain separate
- Ensure weak evidence does not produce strong matches

### Ambiguity Tests

Tests for conflict detection and ambiguous resolution handling.

- Located in `tests/negative/ambiguity/`
- Verify conflicting evidence produces ambiguous state
- Ensure human review is required for conflicts

### Regression Tests

Historical bugs encoded as tests.

- Located in `tests/regression/`
- Each test documents the original bug and fix
- Prevents re-introduction of fixed issues

### Invariant Tests

System-wide property tests.

- Located in `tests/invariants/`
- Verify determinism, symmetry, monotonicity
- Run against generated test data

## Test Fixture Format

All test fixtures use YAML format with the following structure:

```yaml
test_id: ENTITY-TIER-RULE-SEQUENCE
description: Human-readable description
rule_tested: RULE-ID
category: golden | negative | ambiguity | regression
entity_type: person | organization | address

input:
  record_a: { ... }
  record_b: { ... }

expected:
  resolution: resolved | unresolved | candidate | ambiguous
  tier: exact | strong | weak | null
  rule_fired: RULE-ID | null
  entity_count: integer
  conflicts_detected: []
  must_not_merge: boolean  # For negative tests

notes: |
  Explanation of what this test verifies and why.

assertions:
  - type: assertion_type
    expected: value
```

## Running Tests

Tests are executed by a test harness that:

1. Loads fixture files from this directory
2. Applies normalization to input records
3. Executes resolution rules
4. Compares actual results to expected results
5. Reports any discrepancies

## Adding New Tests

When adding new tests:

1. Choose the appropriate category (golden, negative, regression, invariant)
2. Follow the naming convention: `ENTITY-TIER-RULE-SEQUENCE` or `ENTITY-NEG-SEQUENCE`
3. Include comprehensive documentation in the `notes` field
4. Add relevant assertions
5. For regression tests, document the original bug

## Test IDs

- Golden: `{ENTITY}-{TIER}-{RULE}-{SEQUENCE}` (e.g., PERSON-EXACT-001-001)
- Negative: `{ENTITY}-NEG-{SEQUENCE}` (e.g., PERSON-NEG-001)
- Regression: `REG-{YEAR}-{SEQUENCE}` (e.g., REG-2024-001)
- Ambiguity: `AMB-{SEQUENCE}` (e.g., AMB-001)
- Invariant: `INV-{SEQUENCE}` (e.g., INV-001)

## Maintenance

- All test changes require code review
- Changing expected outputs requires documented justification
- Regression tests must never be deleted
- Failed tests block deployment

