# Testing Guide

Testing is a first-class component of this system. The test suite serves as both verification and specification.

## Testing Philosophy

### Tests as Contracts

Each test represents a behavioral contract. If a test fails, either:
1. The implementation has regressed, or
2. The contract needs explicit renegotiation

Changing test expectations requires review and justification.

### False Positives are Defects

Over-merging (incorrectly resolving two distinct entities as the same) is treated as a defect. This is fundamentally different from typical software testing or ML evaluation where some error rate is acceptable.

A single false positive in a legal or compliance context can:
- Contaminate evidence
- Trigger incorrect regulatory filings
- Damage trust in the entire system

### Coverage Philosophy

Coverage is measured by:
- Rule coverage: Every rule has positive and negative tests
- Edge case coverage: Known failure modes are encoded as tests
- Tier coverage: Each confidence tier has tests demonstrating boundaries

Code coverage metrics are secondary. A system can have 100% code coverage and still fail to catch resolution errors.

---

## Coverage Strategy by Tier

Test coverage is intentionally allocated based on the consequences of incorrect behavior at each confidence tier.

### Exact-Match Rules: Full Deterministic Coverage

Every exact-match rule has complete test coverage, including:
- At least one golden-case fixture demonstrating correct resolution
- At least one negative fixture demonstrating non-merge when conditions are not met

Exact-match rules use high-specificity identifiers (SSN, EIN, LEI, passport numbers, registration numbers). A false positive at this tier contaminates downstream systems that treat exact matches as authoritative. There is no acceptable error rate.

### Strong-Match Rules: Representative Coverage

Strong-match rules have representative coverage:
- One golden-case fixture per rule demonstrating the expected resolution
- One negative or conflict-preserving fixture per rule

Strong matches use multiple corroborating attributes. The test suite verifies that rules behave as designed. Exhaustive permutation testing is not performed because:
- Strong matches already require human review for high-stakes decisions
- The combinatorial space of attribute variations is large
- Representative tests establish the behavioral contract

### Weak-Match Rules: Selective Coverage

Weak-match rules have selective test coverage. This is intentional.

Weak matches by definition represent insufficient evidence for confident resolution. They produce candidate clusters for human review, not resolved entities. Exhaustive testing of weak-match rules:
- Provides diminishing returns given that output requires human review
- Risks over-specifying behavior that should remain flexible
- Is better addressed through invariant tests (e.g., weak matches never auto-escalate)

The invariant tests verify system-wide guarantees about weak-tier behavior. Individual rule permutations are not exhaustively specified.

### Rationale

This coverage allocation reflects the design principle that false positives are defects. Test investment is concentrated where incorrect automated behavior causes the most harm: at exact and strong tiers where systems may act on resolution decisions without human review.

---

## Test Categories

### Golden-Case Tests

Location: `tests/golden/`

Fixed inputs with deterministic expected outputs. These are the primary behavioral contracts.

**Structure**:
```
tests/golden/
├── person/
│   ├── exact/
│   │   ├── ssn_match.yaml
│   │   ├── passport_match.yaml
│   │   └── national_id_match.yaml
│   ├── strong/
│   │   ├── name_dob_address.yaml
│   │   └── name_dob_phone.yaml
│   └── weak/
│       ├── name_address_no_dob.yaml
│       └── name_dob_no_address.yaml
├── organization/
│   └── ...
└── address/
    └── ...
```

**Fixture Format**:
```yaml
test_id: PERSON-EXACT-001-001
description: Two records with matching SSN resolve as exact match
rule_tested: PERSON-EXACT-001

input:
  record_a:
    identifiers:
      ssn: "123-45-6789"
    name:
      given_name: "John"
      family_name: "Smith"
    date_of_birth: "1980-01-15"
  record_b:
    identifiers:
      ssn: "123-45-6789"
    name:
      given_name: "Jonathan"
      family_name: "Smith"
    date_of_birth: "1980-01-15"

expected:
  resolution: resolved
  tier: exact
  rule_fired: PERSON-EXACT-001
  entity_count: 1

notes: |
  Name variation (John vs Jonathan) does not prevent exact match
  when SSN matches. SSN is authoritative.
```

### Negative Tests

Location: `tests/negative/`

Explicit tests for cases where resolution must NOT occur.

**Purpose**:
- Verify that similar-but-distinct entities remain separate
- Ensure conflict detection works correctly
- Validate that weak evidence does not produce strong matches

**Example**:
```yaml
test_id: PERSON-NEG-001
description: Same name in different cities must not resolve
rule_tested: PERSON-WEAK-001

input:
  record_a:
    name:
      given_name: "John"
      family_name: "Smith"
    addresses:
      - locality: "New York"
        region: "NY"
  record_b:
    name:
      given_name: "John"
      family_name: "Smith"
    addresses:
      - locality: "Los Angeles"
        region: "CA"

expected:
  resolution: unresolved
  tier: null
  reason: "Different localities; common name without corroborating evidence"

notes: |
  John Smith is a common name. Without shared address, DOB, or
  identifier, these must remain separate entities.
```

### Regression Tests

Location: `tests/regression/`

Historical bugs encoded as tests. Each test includes documentation of the original bug.

**Structure**:
```yaml
test_id: REG-2024-001
description: Transposed SSN digits must not match
original_bug: |
  On 2024-03-15, records with SSNs 123-45-6789 and 123-45-6798
  were incorrectly merged. The normalization step was stripping
  hyphens before digit-order validation.
date_discovered: "2024-03-15"
fixed_in: "v1.2.3"

input:
  record_a:
    identifiers:
      ssn: "123-45-6789"
    name:
      given_name: "Jane"
      family_name: "Doe"
  record_b:
    identifiers:
      ssn: "123-45-6798"
    name:
      given_name: "Jane"
      family_name: "Doe"

expected:
  resolution: unresolved
  reason: "SSN mismatch (transposed digits)"

notes: |
  Even though names match, transposed SSN digits indicate different
  individuals. This is a common data entry error that must not
  produce false positives.
```

### Ambiguity Tests

Location: `tests/negative/ambiguity/`

Tests for cases where conflicting evidence must preserve uncertainty.

**Example**:
```yaml
test_id: AMB-001
description: Matching SSN with conflicting DOB produces ambiguity
rule_tested: PERSON-EXACT-001

input:
  record_a:
    identifiers:
      ssn: "123-45-6789"
    name:
      given_name: "John"
      family_name: "Smith"
    date_of_birth: "1980-01-15"
  record_b:
    identifiers:
      ssn: "123-45-6789"
    name:
      given_name: "John"
      family_name: "Smith"
    date_of_birth: "1985-03-22"

expected:
  resolution: ambiguous
  tier: null
  conflict_type: "identifier_attribute_mismatch"
  requires_review: true

notes: |
  Same SSN but different DOB. Could be:
  - Data entry error on one record
  - SSN reuse (rare but documented)
  - Identity fraud
  Human review required.
```

### Invariant Tests

Location: `tests/invariants/`

Property-based tests that verify system-wide guarantees.

**Invariants to Test**:

1. **Determinism**: Same inputs always produce same outputs
```yaml
invariant_id: INV-001
description: Resolution is deterministic
property: |
  For any two records A and B:
    resolve(A, B) == resolve(A, B)
  Repeated execution produces identical results.
```

2. **Symmetry**: Order of inputs does not affect result
```yaml
invariant_id: INV-002
description: Resolution is symmetric
property: |
  For any two records A and B:
    resolve(A, B) == resolve(B, A)
  Input order does not affect resolution.
```

3. **Monotonic Confidence**: Removing information cannot increase confidence
```yaml
invariant_id: INV-003
description: Confidence is monotonically related to evidence
property: |
  For any two records A and B:
    If resolve(A, B) produces tier T, then
    resolve(A', B) where A' is A with one field removed
    produces tier T' where T' <= T
  (exact > strong > weak > unresolved)
```

4. **Tier Boundaries**: Weak matches never automatically escalate
```yaml
invariant_id: INV-004
description: Weak matches do not escalate automatically
property: |
  For any set of records where all pairwise resolutions
  produce weak matches, the aggregate resolution is weak.
  Accumulating weak evidence does not produce strong matches.
```

---

## Test Execution

### Running Tests

Tests are executed against the resolution implementation. The test harness:

1. Loads fixture files
2. Applies normalization to inputs
3. Executes resolution rules
4. Compares actual output to expected output
5. Reports discrepancies

### Failure Handling

When a test fails:

1. **Golden test failure**: Implementation has regressed; fix required
2. **Negative test failure**: Over-merging detected; this is a critical bug
3. **Invariant test failure**: System-wide guarantee violated; investigation required

### Continuous Integration

Tests run on every commit. The build fails if:
- Any golden test fails
- Any negative test fails
- Any regression test fails
- Any invariant test fails

There is no "acceptable failure rate." All tests must pass.

---

## Writing New Tests

### When to Add Tests

Add tests when:
- A new rule is added (positive and negative tests required)
- A bug is discovered (regression test required)
- A new edge case is identified (appropriate category)
- Rule behavior is clarified (golden test to document)

### Test Naming

Test IDs follow the pattern:
- Golden: `{ENTITY}-{TIER}-{RULE}-{SEQUENCE}`
- Negative: `{ENTITY}-NEG-{SEQUENCE}`
- Regression: `REG-{YEAR}-{SEQUENCE}`
- Ambiguity: `AMB-{SEQUENCE}`
- Invariant: `INV-{SEQUENCE}`

### Documentation Requirements

Every test must include:
- Clear description of what is being tested
- Reference to the rule being tested (if applicable)
- Notes explaining why this test exists
- For regression tests: original bug description and date

---

## Test Review Process

### New Test Review

New tests are reviewed for:
- Correctness of expected output
- Completeness of input data
- Clarity of documentation
- Appropriate categorization

### Expectation Changes

Changing test expectations requires:
- Justification for why the old expectation was wrong
- Review by at least one other team member
- Documentation of the change in commit message

### Audit Trail

All test changes are tracked in version control. The history of test expectations is part of the audit trail for the system's behavior over time.

