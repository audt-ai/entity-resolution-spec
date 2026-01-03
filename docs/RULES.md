# Resolution Rules Reference

This document specifies the deterministic rules used for entity resolution.

## Rule Structure

Each rule is defined with:
- **Rule ID**: Unique identifier for traceability
- **Entity Type**: Person, Organization, or Address
- **Match Tier**: Exact, Strong, or Weak
- **Required Fields**: Fields that must be present for the rule to evaluate
- **Match Condition**: The deterministic condition for a match
- **Conflict Indicators**: Conditions that produce ambiguity instead of a match

---

## Person Rules

### Exact Match Rules

#### PERSON-EXACT-001: SSN Match
- **Required Fields**: `identifiers.ssn`
- **Match Condition**: Byte-for-byte equality of normalized SSN
- **Conflict Indicators**: Matching SSN with conflicting date of birth
- **Notes**: SSN must pass format validation (9 digits, no placeholder patterns)

#### PERSON-EXACT-002: Passport Match
- **Required Fields**: `identifiers.passport_number`, `identifiers.passport_country`
- **Match Condition**: Exact match on both passport number and issuing country
- **Conflict Indicators**: Matching passport with conflicting name (beyond normalization variance)
- **Notes**: Passport numbers are case-insensitive after normalization

#### PERSON-EXACT-003: National ID Match
- **Required Fields**: `identifiers.national_id`, `identifiers.national_id_country`
- **Match Condition**: Exact match on both national ID and issuing country
- **Conflict Indicators**: Matching ID with conflicting biometric hash
- **Notes**: Country-specific format validation applies

### Strong Match Rules

#### PERSON-STRONG-001: Name + DOB + Address
- **Required Fields**: `name.family_name`, `name.given_name`, `date_of_birth`, `addresses[]`
- **Match Condition**: All of:
  - Normalized family name matches
  - Normalized given name matches
  - Date of birth matches exactly
  - At least one address resolves to the same address entity
- **Conflict Indicators**: Any high-specificity identifier mismatch
- **Notes**: Name normalization includes handling of common variants

#### PERSON-STRONG-002: Name + DOB + Phone
- **Required Fields**: `name.family_name`, `name.given_name`, `date_of_birth`, `contact.phone`
- **Match Condition**: All of:
  - Normalized family name matches
  - Normalized given name matches
  - Date of birth matches exactly
  - Normalized phone number matches
- **Conflict Indicators**: Any high-specificity identifier mismatch
- **Notes**: Phone normalization removes formatting, applies country code

#### PERSON-STRONG-003: Name + DOB + Email
- **Required Fields**: `name.family_name`, `name.given_name`, `date_of_birth`, `contact.email`
- **Match Condition**: All of:
  - Normalized family name matches
  - Normalized given name matches
  - Date of birth matches exactly
  - Normalized email matches (case-insensitive)
- **Conflict Indicators**: Any high-specificity identifier mismatch

### Weak Match Rules

#### PERSON-WEAK-001: Name + Address (No DOB)
- **Required Fields**: `name.family_name`, `name.given_name`, `addresses[]`
- **Match Condition**: All of:
  - Normalized family name matches
  - Normalized given name matches
  - At least one address resolves to the same address entity
- **Conflict Indicators**: Mismatched date of birth (if present on either record)
- **Notes**: Common names with shared addresses are frequent false positives

#### PERSON-WEAK-002: Name + DOB (No Address)
- **Required Fields**: `name.family_name`, `name.given_name`, `date_of_birth`
- **Match Condition**: All of:
  - Normalized family name matches
  - Normalized given name matches
  - Date of birth matches exactly
- **Conflict Indicators**: Different addresses in same time period
- **Notes**: Without address, twins and common names are problematic

---

## Organization Rules

### Exact Match Rules

#### ORG-EXACT-001: EIN Match
- **Required Fields**: `identifiers.ein`
- **Match Condition**: Byte-for-byte equality of normalized EIN
- **Conflict Indicators**: Matching EIN with conflicting jurisdiction
- **Notes**: EIN must pass format validation (XX-XXXXXXX)

#### ORG-EXACT-002: LEI Match
- **Required Fields**: `identifiers.lei`
- **Match Condition**: Byte-for-byte equality of LEI
- **Conflict Indicators**: Matching LEI with conflicting legal name
- **Notes**: LEI is 20 characters, alphanumeric

#### ORG-EXACT-003: Registration Number + Jurisdiction
- **Required Fields**: `identifiers.registration_number`, `jurisdiction`
- **Match Condition**: Exact match on registration number within same jurisdiction
- **Conflict Indicators**: Matching registration with conflicting formation date
- **Notes**: Registration numbers are not unique across jurisdictions

### Strong Match Rules

#### ORG-STRONG-001: Legal Name + Jurisdiction + Address
- **Required Fields**: `names.legal_name`, `jurisdiction`, `addresses[]`
- **Match Condition**: All of:
  - Normalized legal name matches
  - Jurisdiction matches
  - At least one address resolves to the same address entity
- **Conflict Indicators**: Any high-specificity identifier mismatch
- **Notes**: Name normalization removes legal suffixes for comparison

#### ORG-STRONG-002: Legal Name + Jurisdiction + Formation Date
- **Required Fields**: `names.legal_name`, `jurisdiction`, `formation_date`
- **Match Condition**: All of:
  - Normalized legal name matches
  - Jurisdiction matches
  - Formation date matches
- **Conflict Indicators**: Any high-specificity identifier mismatch

### Weak Match Rules

#### ORG-WEAK-001: Legal Name + Jurisdiction (No Address)
- **Required Fields**: `names.legal_name`, `jurisdiction`
- **Match Condition**: All of:
  - Normalized legal name matches
  - Jurisdiction matches
- **Conflict Indicators**: Different addresses, different formation dates
- **Notes**: Many organizations share names within jurisdictions

#### ORG-WEAK-002: Trade Name Match
- **Required Fields**: `names.trade_names[]`
- **Match Condition**: Any trade name matches after normalization
- **Conflict Indicators**: Different legal names, different jurisdictions
- **Notes**: Trade names are highly ambiguous; use with caution

---

## Address Rules

### Exact Match Rules

#### ADDR-EXACT-001: Full Normalized Address
- **Required Fields**: All address components
- **Match Condition**: Byte-for-byte equality of all normalized components
- **Conflict Indicators**: None (exact match is definitive for addresses)
- **Notes**: Includes street, unit, locality, region, postal code, country

### Strong Match Rules

#### ADDR-STRONG-001: Street + Locality + Postal Code
- **Required Fields**: `street`, `locality`, `postal_code`
- **Match Condition**: All of:
  - Normalized street matches
  - Normalized locality matches
  - Postal code matches
- **Conflict Indicators**: Different unit numbers on multi-unit addresses
- **Notes**: Missing country assumes same country context

#### ADDR-STRONG-002: Street + Unit + Postal Code
- **Required Fields**: `street`, `unit`, `postal_code`
- **Match Condition**: All of:
  - Normalized street matches
  - Normalized unit matches
  - Postal code matches
- **Conflict Indicators**: Different localities
- **Notes**: Unit match distinguishes apartments in same building

### Weak Match Rules

#### ADDR-WEAK-001: Street + Locality (No Postal Code)
- **Required Fields**: `street`, `locality`
- **Match Condition**: All of:
  - Normalized street matches
  - Normalized locality matches
- **Conflict Indicators**: Different postal codes (if present)
- **Notes**: Street names repeat across localities

#### ADDR-WEAK-002: Postal Code + Unit (No Street)
- **Required Fields**: `postal_code`, `unit`
- **Match Condition**: All of:
  - Postal code matches
  - Normalized unit matches
- **Conflict Indicators**: Different streets
- **Notes**: Useful for PO boxes within same post office

---

## Rule Evaluation Order

1. All exact match rules for the entity type are evaluated
2. If any exact match fires without conflict, resolution is complete at Exact tier
3. If exact matches conflict, mark as ambiguous and stop
4. All strong match rules are evaluated
5. If any strong match fires without conflict, resolution is complete at Strong tier
6. If strong matches conflict, mark as ambiguous and stop
7. All weak match rules are evaluated
8. If any weak match fires, create candidate cluster at Weak tier
9. Weak matches do not resolve; they suggest potential matches for review

---

## Conflict Resolution

When rules at the same tier produce conflicting results:

1. **Log all conflicting rules**: Record which rules fired and their evidence
2. **Mark entity pair as ambiguous**: Do not resolve
3. **Flag for human review**: Add to review queue with conflict details
4. **Preserve both interpretations**: Store both potential resolutions as candidates

Conflicts are never resolved automatically. Human review is required.

---

## Adding Custom Rules

Custom rules must specify:

1. Unique rule ID following naming convention: `{ENTITY}-{TIER}-{NUMBER}`
2. All required fields with exact schema paths
3. Match condition as a deterministic boolean expression
4. Conflict indicators that prevent automatic resolution
5. Documentation explaining the rule's purpose and known edge cases

Custom rules are inserted into the evaluation order based on their tier. Within a tier, rules are evaluated in ID order.

