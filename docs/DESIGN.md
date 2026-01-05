## this schema is meant for:

- Resolves identity across three entity types: people, organizations, and addresses
- Provides deterministic matching rules that produce identical outputs for identical inputs
- Preserves ambiguity when evidence is insufficient for resolution
- Maintains complete lineage from resolved entities back to source records
- Supports human review workflows for contested or ambiguous resolutions



### explicit Boundaries

- No machine learning components
- No probabilistic inference engines
- No "risk," "fraud," "suspicious," or similar language in schema or rules
- No behavioral analytics or pattern detection
- No automated decisioning hooks

---

## 2. Entity Types

### People

An entity representing a natural person. A person entity is defined by the combination of identifiers and attributes that distinguish one individual from another.

**Core Identifiers** (high-specificity, suitable for exact matching):
- Government-issued IDs (SSN, passport number, national ID)
- Date of birth (when combined with other identifiers)
- Biometric hashes (if present in source data)

**Attributes** (lower specificity, used for corroboration):
- Name components (given name, family name, middle names, suffixes)
- Contact information (phone, email)
- Associated addresses (via address entity references)

**Handling Partial Data**:
- Missing identifiers do not disqualify an entity; they reduce match strength
- Conflicting identifiers on the same record produce an ambiguous entity that cannot be merged without human review
- Partial names (initials, nicknames) are stored as-is; normalization is deferred to match rules

### Organizations

An entity representing a legal or informal organization: corporations, partnerships, trusts, sole proprietorships, unincorporated associations.

**Core Identifiers**:
- Registration numbers (EIN, company registration number, LEI)
- Jurisdiction of incorporation/registration

**Attributes**:
- Legal name, trade names, DBAs
- Formation date
- Registered agent
- Associated addresses (via address entity references)
- Officer/director relationships (via person entity references)

**Handling Partial Data**:
- Organizations without registration numbers are valid entities; they rely on name and address matching
- Multiple DBAs for the same legal entity are stored as attributes, not separate entities
- Jurisdictional ambiguity (e.g., same name in different states) results in separate entities until evidence supports merge

### Addresses

An entity representing a physical or mailing location. Addresses are first-class entities because they change over time, are shared across people and organizations, and are subject to normalization complexity.

**Components**:
- Street address (number, street name, street type, directional)
- Secondary unit (apartment, suite, floor, building)
- Locality (city, town, village)
- Administrative region (state, province, county)
- Postal code
- Country

**Handling Partial Data**:
- Missing components reduce match strength but do not invalidate the address
- PO boxes and rural routes are valid addresses with their own normalization rules
- Care-of lines and attention lines are metadata, not address components

---

## 3. Deterministic Resolution Rules

### Match Tiers

**Exact Match**:
- Two records share at least one high-specificity identifier with byte-for-byte equality after normalization
- Example: Same SSN, same passport number, same EIN
- Result: Records refer to the same entity with no ambiguity

**Strong Match**:
- Two records share multiple corroborating attributes that, in combination, uniquely identify an entity in the dataset
- Example: Same full legal name + same date of birth + same address
- Result: Records likely refer to the same entity; merge permitted without human review in most workflows

**Weak Match**:
- Two records share some attributes but lack sufficient evidence for confident resolution
- Example: Same common name + same city (but different addresses)
- Result: Records may refer to the same entity; merge requires human review or additional evidence

### Rule Precedence

1. Exact match rules are evaluated first; if any exact match rule fires, resolution is complete
2. Strong match rules are evaluated only if no exact match exists
3. Weak match rules are evaluated only if no strong match exists
4. Conflicting rules at the same tier produce an ambiguous result (no automatic resolution)

### Conflict Handling

- If two exact-match rules produce conflicting results (e.g., same SSN but different dates of birth), the resolution is flagged for review
- Conflicts never resolve silently; they produce explicit ambiguity markers
- Downstream systems must handle ambiguity markers; they cannot assume all entities are cleanly resolved

### Graceful Degradation

- Rules are designed to fail open: missing data prevents a match rather than forcing one
- A rule that requires three attributes will not fire if only two are present
- Edge cases (empty strings, placeholder values like "N/A" or "UNKNOWN") are treated as missing

### Intentional Ambiguity Retention

- When evidence is insufficient, the schema produces a "candidate cluster" rather than a resolved entity
- Candidate clusters preserve all source records and document why resolution was not possible
- Ambiguity is a valid output state, not an error condition

---

## 4. Confidence Tiers

### Exact

- **Definition**: Resolution based on at least one high-specificity identifier match
- **Operational Meaning**: These entities can be treated as resolved in downstream systems
- **Downstream May**: Use for reporting, compliance filings, aggregation
- **Downstream Must Not**: Assume absence of data quality issues in source records

### Strong

- **Definition**: Resolution based on multiple corroborating attributes with no conflicts
- **Operational Meaning**: High confidence; suitable for most analytical workflows
- **Downstream May**: Use for analysis, alerting, case preparation
- **Downstream Must Not**: Use as sole basis for legal action without human review

### Weak

- **Definition**: Partial attribute overlap; insufficient for confident resolution
- **Operational Meaning**: Candidate for review; not resolved
- **Downstream May**: Present to analysts as potential matches; include in review queues
- **Downstream Must Not**: Treat as resolved; merge without explicit human action

### Tier Boundaries

- Confidence tiers do not automatically escalate
- Accumulating weak matches does not produce a strong match
- Only new evidence (new records with additional identifiers) can change tier assignment
- Tier downgrades are possible if new conflicting evidence emerges

---

## 5. Address Normalization and Canonicalization

### Normalization Strategy

Address normalization is applied before matching. The goal is standardization for comparison, not data correction.

**Steps**:
1. Parse raw address into components
2. Standardize abbreviations (St → Street, Apt → Apartment)
3. Normalize case (uppercase or lowercase, applied consistently)
4. Remove punctuation except where semantically significant
5. Standardize directionals (N → North, NW → Northwest)
6. Do not infer missing components; leave them null

### Secondary Units

- Units, suites, apartments, floors, and buildings are preserved as separate components
- Missing unit numbers on multi-unit addresses produce weaker matches
- "#" and "Unit" and "Apt" are normalized to a consistent form

### PO Boxes and Rural Routes

- PO boxes are valid addresses with their own format: "PO Box [number]"
- Rural routes follow format: "RR [number] Box [number]"
- These do not normalize to street addresses; they are distinct types

### Temporal Changes

- Address entities have effective date ranges
- A person's association with an address has start and end dates
- Historical address associations are preserved, not overwritten

### Known Failure Modes

- USPS validation APIs may correct addresses in ways that break linkage to source records
- International addresses have highly variable formats; normalization is best-effort
- New construction may not appear in postal databases
- Vanity addresses (building names instead of numbers) require special handling

### Tradeoffs

- Over-normalization risks merging distinct addresses
- Under-normalization risks missing valid matches
- The schema errs toward under-normalization

---

## 6. Temporal Considerations

### Identity Over Time

Entities evolve. People change names, organizations merge, addresses are reassigned. The schema must represent these changes without losing history.

**Principles**:
- Entity identity is stable; attributes change over time
- Historical states are preserved as snapshots, not overwritten
- Resolution decisions are timestamped

### Same vs. Similar

Two entities that were the same at one point in time may diverge (e.g., corporate spinoff). Two entities that appear similar may never have been the same.

**Representation**:
- "Same" is represented by a resolution link with an effective date range
- "Similar" is represented by a candidate cluster with no resolution link
- Historical sameness with subsequent divergence is represented by a resolution link with an end date

### Merges, Splits, and Corrections

**Merge**: Two previously separate entities are determined to be the same
- Both original entities are retained
- A merge record links them with an effective date
- The merge record documents the evidence for the decision

**Split**: A previously resolved entity is determined to be two distinct entities
- The resolution link is ended (given an end date)
- A split record documents the evidence
- No data is deleted

**Correction**: An attribute on an entity is determined to be incorrect
- The old value is retained with an end date
- The new value is added with a start date
- A correction record documents the source of the correction

### Immutability

- Source records are immutable
- Resolution decisions are append-only
- Entity attributes are versioned, not mutated
- Deletions are soft (marked as deleted, not removed)

---

## 7. Evidence Lineage Hooks

### Traceability Requirements

Every resolved entity must trace back to source records:
- Which source records contributed to this entity?
- Which source records provided which attributes?
- Which rules fired to produce this resolution?
- What was the confidence tier at time of resolution?

### Transformation Documentation

Every transformation applied to source data is recorded:
- Normalization steps (before and after)
- Parsing decisions (how raw strings became structured components)
- Rule evaluations (which rules were evaluated, which fired, which did not)

### Review and Challenge

Resolution decisions must support review:
- All evidence for a resolution is retrievable
- Resolutions can be flagged for human review
- Human overrides are recorded with justification
- Overrides do not delete original decisions; they add a new decision layer

### Implementation Hooks

The schema includes extension points for:
- Audit logging
- Decision review queues
- Override workflows
- Lineage queries

---

## 8. Testing Philosophy and Structure

### Golden-Case Tests

Fixed inputs with deterministic expected outputs. These tests serve as behavioral contracts.

**Characteristics**:
- Inputs are checked into the repository as fixtures
- Expected outputs are checked in alongside inputs
- Any change to outputs requires explicit review
- Tests are named descriptively (e.g., `test_ssn_exact_match_produces_exact_tier`)

### Negative and Ambiguity Tests

Explicit tests for cases where resolution must not occur.

**Categories**:
- Similar identifiers that must remain distinct (e.g., John Smith in NYC vs. John Smith in LA)
- Conflicting data that must preserve uncertainty (e.g., same name, different SSN)
- Edge cases that must fail open (e.g., placeholder values, empty strings)

### Regression Tests for False Positives

Over-merging is treated as a failure. Historical bugs are encoded as tests.

**Approach**:
- When a false positive is discovered, the inputs are added as a test case
- The test asserts that the entities remain distinct
- Tests include comments explaining the original bug

### Invariant and Property Tests

Tests that verify system-wide guarantees:

- **Resolution stability**: Same inputs always produce same outputs
- **Monotonic confidence**: Removing information cannot increase confidence
- **Tier boundaries**: Weak matches never automatically escalate to strong

### Tests as Documentation

The test suite is the authoritative specification for system behavior. Tests are organized to be readable by auditors and legal reviewers:

- Test names describe business rules, not implementation details
- Test files are organized by entity type and match tier
- Comments explain why each test exists

### Legal and Audit Alignment

Tests reflect legal and audit expectations:

- No precision/recall tradeoff optimization (this is not ML evaluation)
- False positives are defects, not acceptable error rates
- Test coverage is measured by rule coverage, not code coverage

---

## 9. Failure Modes and Tradeoffs

### Known Edge Cases

**Data Quality**:
- Transposed digits in identifiers
- Misspellings in names
- Outdated addresses
- Missing or null identifiers

**Systemic Issues**:
- Source systems with known data quality problems
- Jurisdictions with non-unique identifiers
- Common names in small geographic areas
- Placeholder values that passed upstream validation

### Human Review Triggers

Certain conditions require human review:

- Conflicting high-specificity identifiers
- Exact match on identifier with strong match on different entity
- Resolution that would merge more than a configurable threshold of records
- Any override of a previous human decision

### Why Ambiguity is Preferable

Forced resolution produces silent errors that compound over time:

- A false positive merged into a legal filing is difficult to correct
- Downstream systems may make decisions based on incorrect entity resolution
- Trust in the data degrades when users discover errors

Retained ambiguity produces visible uncertainty:

- Analysts can investigate and resolve
- Human review can be prioritized
- Downstream systems can handle uncertainty explicitly

---

## 10. Reference Implementation Outline

### Module Structure

```
entity-resolution-spec/
├── schema/
│   ├── entity/
│   │   ├── person.yaml
│   │   ├── organization.yaml
│   │   └── address.yaml
│   ├── resolution/
│   │   ├── rules.yaml
│   │   ├── tiers.yaml
│   │   └── conflicts.yaml
│   └── lineage/
│       ├── source_record.yaml
│       ├── transformation.yaml
│       └── decision.yaml
├── normalization/
│   ├── address.yaml
│   ├── name.yaml
│   └── identifier.yaml
├── tests/
│   ├── golden/
│   ├── negative/
│   ├── regression/
│   └── invariants/
├── docs/
│   ├── DESIGN.md
│   ├── RULES.md
│   └── TESTING.md
└── README.md
```

### Schema Definitions

Schemas are defined in YAML for language-agnostic consumption. Each schema includes:

- Field definitions with types and constraints
- Required vs. optional fields
- Validation rules
- Documentation strings

### Extension Points

The schema supports extension without modification:

- Custom identifier types can be added to entity schemas
- Custom normalization rules can be added to normalization modules
- Custom match rules can be added with explicit precedence
- Custom lineage hooks can be added for organization-specific audit requirements

---

## 11. Operational Philosophy

### Intended Use

This schema supports analytical workflows where identity resolution is a prerequisite for investigation, compliance, or reporting. It is not a product; it is infrastructure.

**Appropriate uses**:
- Building entity graphs for investigation
- Deduplicating records for compliance reporting
- Linking records across data sources for audit
- Preparing evidence for legal proceedings

**Inappropriate uses**:
- Real-time transaction decisioning
- Automated blocking or denial
- Risk scoring or fraud detection
- Any automated adverse action

### Restraint as Design Choice

The schema is deliberately conservative:

- It under-merges rather than over-merges
- It preserves ambiguity rather than forcing resolution
- It requires human review for edge cases
- It prioritizes correctness over completeness

### Long-Term Trust Over Short-Term Coverage

A system that is 80% complete and 99.9% correct is more valuable than a system that is 95% complete and 95% correct:

- Errors compound over time
- Trust, once lost, is difficult to recover
- Analysts will route around systems they do not trust
- Legal and compliance teams will require manual verification of untrusted systems

The schema is designed to be trusted. Trust is earned through predictable behavior, complete lineage, and conservative defaults.

