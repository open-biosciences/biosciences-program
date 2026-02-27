# Data Model: ClinicalTrials.gov MCP Server

**Feature**: 013-clinicaltrials-mcp-server
**Created**: 2026-01-03
**Input**: Feature specification and research phase

## Overview

This document defines the Pydantic data models for the ClinicalTrials.gov MCP server. All models follow the Agentic Biolink schema with flattened JSON structure, omit-if-null pattern, and Canonical Envelopes.

## Entity Models

### TrialSearchCandidate

**Purpose**: Lightweight trial representation returned by fuzzy search (`search_trials`).

**Token Budget**: ~100-200 tokens per candidate

**Fields**:
```python
class TrialSearchCandidate(BaseModel):
    """Lightweight trial search result for fuzzy discovery."""

    id: str                     # NCT CURIE (e.g., "NCT:00461032")
    title: str                  # Official title
    brief_summary: str          # Brief description
    phase: str | None           # Phase (PHASE1, PHASE2, PHASE3, PHASE4, NA, EARLY_PHASE1)
    status: str                 # Recruitment status (RECRUITING, COMPLETED, etc.)
    conditions: list[str]       # Disease/condition names
    interventions: list[str]    # Treatment/intervention names

    # Validation
    @field_validator("id")
    def validate_nct_id(cls, v: str) -> str:
        if not re.match(r"^NCT:\d{8}$", v):
            raise ValueError(f"Invalid NCT CURIE format: {v}")
        return v
```

**Example**:
```json
{
  "id": "NCT:00461032",
  "title": "Bevacizumab and Erlotinib in Treating Patients With Metastatic Breast Cancer",
  "brief_summary": "This phase II trial is studying how well giving bevacizumab together...",
  "phase": "PHASE2",
  "status": "COMPLETED",
  "conditions": ["Breast Cancer", "Metastatic Breast Cancer"],
  "interventions": ["Bevacizumab", "Erlotinib Hydrochloride"]
}
```

---

### Trial

**Purpose**: Complete trial entity returned by strict lookup (`get_trial`).

**Token Budget**: ~5K-10K tokens (deep nested structure from API)

**Fields**:
```python
class Trial(BaseModel):
    """Complete clinical trial entity with full protocol details."""

    # Identity
    id: str                           # NCT CURIE (e.g., "NCT:00461032")
    title: str                        # Official title
    brief_summary: str                # Brief description
    detailed_description: str | None  # Full protocol description (omit if missing)

    # Protocol
    protocol: TrialProtocol           # Nested protocol details (see below)

    # Eligibility
    eligibility_criteria: EligibilityCriteria | None  # Inclusion/exclusion (omit if missing)

    # Outcomes
    primary_outcomes: list[Outcome]   # Primary endpoints
    secondary_outcomes: list[Outcome] # Secondary endpoints

    # Administrative
    sponsors: list[Sponsor]           # Lead sponsor + collaborators
    phase: str | None                 # Trial phase
    status: str                       # Recruitment status
    enrollment: int | None            # Target/actual enrollment

    # Dates
    start_date: str | None            # Study start date (YYYY-MM-DD or partial)
    completion_date: str | None       # Primary completion date
    last_update_date: str | None      # Last registry update

    # Cross-references
    cross_references: dict[str, str]  # External IDs (see below)

    # Validation
    @field_validator("id")
    def validate_nct_id(cls, v: str) -> str:
        if not re.match(r"^NCT:\d{8}$", v):
            raise ValueError(f"Invalid NCT CURIE format: {v}")
        return v
```

**Nested Models**:

#### TrialProtocol
```python
class TrialProtocol(BaseModel):
    """Protocol design details."""

    study_type: str                   # INTERVENTIONAL, OBSERVATIONAL, EXPANDED_ACCESS
    allocation: str | None            # RANDOMIZED, NON_RANDOMIZED, NA
    intervention_model: str | None    # PARALLEL, CROSSOVER, SINGLE_GROUP, etc.
    masking: str | None               # NONE, SINGLE, DOUBLE, TRIPLE, QUADRUPLE
    primary_purpose: str | None       # TREATMENT, PREVENTION, DIAGNOSTIC, etc.
```

#### EligibilityCriteria
```python
class EligibilityCriteria(BaseModel):
    """Patient eligibility requirements."""

    criteria_text: str | None         # Formatted inclusion/exclusion criteria
    minimum_age: str | None           # Minimum age (e.g., "18 Years")
    maximum_age: str | None           # Maximum age (e.g., "65 Years")
    sex: str | None                   # ALL, FEMALE, MALE
    accepts_healthy_volunteers: bool  # Whether healthy volunteers accepted
```

#### Outcome
```python
class Outcome(BaseModel):
    """Trial outcome measure."""

    measure: str                      # Outcome description
    time_frame: str | None            # Assessment timeframe
    description: str | None           # Additional details
```

#### Sponsor
```python
class Sponsor(BaseModel):
    """Trial sponsor or collaborator."""

    name: str                         # Organization name
    role: str                         # LEAD_SPONSOR, COLLABORATOR
```

**Cross-References**:
```python
# Limited cross-references from ClinicalTrials.gov API
{
    "pubmed": "12345678",                              # PubMed ID (if publications exist)
    "clinicaltrials_gov": "https://clinicaltrials.gov/study/NCT00461032",
    "mesh_conditions": "D001943",                      # MeSH disease ID
    "mesh_interventions": "D000068258"                 # MeSH drug ID
}
```

**Example** (abbreviated):
```json
{
  "id": "NCT:00461032",
  "title": "Bevacizumab and Erlotinib in Treating Patients...",
  "brief_summary": "This phase II trial is studying...",
  "protocol": {
    "study_type": "INTERVENTIONAL",
    "allocation": "RANDOMIZED",
    "intervention_model": "PARALLEL",
    "masking": "NONE",
    "primary_purpose": "TREATMENT"
  },
  "eligibility_criteria": {
    "criteria_text": "Inclusion Criteria:\n- Metastatic breast cancer...",
    "minimum_age": "18 Years",
    "sex": "FEMALE",
    "accepts_healthy_volunteers": false
  },
  "primary_outcomes": [
    {
      "measure": "Objective Response Rate",
      "time_frame": "6 months"
    }
  ],
  "sponsors": [
    {"name": "National Cancer Institute", "role": "LEAD_SPONSOR"}
  ],
  "phase": "PHASE2",
  "status": "COMPLETED",
  "enrollment": 50,
  "start_date": "2007-04",
  "completion_date": "2010-12",
  "cross_references": {
    "pubmed": "23456789",
    "clinicaltrials_gov": "https://clinicaltrials.gov/study/NCT00461032",
    "mesh_conditions": "D001943"
  }
}
```

---

### TrialLocation

**Purpose**: Geographic facility information returned by `get_trial_locations`.

**Token Budget**: ~50-100 tokens per location

**Fields**:
```python
class TrialLocation(BaseModel):
    """Trial facility with contact information."""

    facility_name: str                # Facility/hospital name
    city: str                         # City name
    state: str | None                 # State/province (US/Canada only)
    country: str                      # Country name
    zip: str | None                   # Postal code
    contact_name: str | None          # Contact person name
    contact_phone: str | None         # Contact phone number
    contact_email: str | None         # Contact email address
    recruitment_status: str           # RECRUITING, NOT_YET_RECRUITING, etc.
```

**Example**:
```json
{
  "facility_name": "Dana-Farber Cancer Institute",
  "city": "Boston",
  "state": "Massachusetts",
  "country": "United States",
  "zip": "02215",
  "contact_name": "Dr. Jane Smith",
  "contact_phone": "617-555-0123",
  "contact_email": "trials@dfci.harvard.edu",
  "recruitment_status": "RECRUITING"
}
```

---

## Envelope Models

### PaginationEnvelope

**Purpose**: Wrap list responses with pagination metadata.

**Usage**: `search_trials` returns `PaginationEnvelope[TrialSearchCandidate]`

**Fields**:
```python
class PaginationEnvelope(BaseModel, Generic[T]):
    """Standard pagination wrapper for list responses."""

    items: list[T]                    # Result items
    pagination: PaginationMetadata

class PaginationMetadata(BaseModel):
    cursor: str | None                # Opaque page token for next page
    total_count: int | None           # Total result count (if available)
    page_size: int                    # Items per page
```

**Example**:
```json
{
  "items": [
    {"id": "NCT:00461032", "title": "...", ...},
    {"id": "NCT:01234567", "title": "...", ...}
  ],
  "pagination": {
    "cursor": "eyJuZXh0UGFnZVRva2VuIjoiQUJDMTIzIn0=",
    "total_count": 1540,
    "page_size": 50
  }
}
```

---

### ErrorEnvelope

**Purpose**: Standardized error responses with recovery hints.

**Usage**: All errors return ErrorEnvelope

**Fields**:
```python
class ErrorEnvelope(BaseModel):
    """Standard error wrapper with recovery guidance."""

    success: Literal[False]           # Always false
    error: ErrorDetails

class ErrorDetails(BaseModel):
    code: str                         # Error code (see below)
    message: str                      # Human-readable error message
    recovery_hint: str                # Actionable guidance for agents
    invalid_input: str | None         # The input that caused the error
```

**Error Codes**:
- `UNRESOLVED_ENTITY` - Query string passed to strict tool
- `ENTITY_NOT_FOUND` - NCT ID does not exist
- `INVALID_INPUT` - Malformed input (e.g., invalid NCT ID format)
- `RATE_LIMITED` - Exceeded 1 req/sec rate limit
- `UPSTREAM_ERROR` - ClinicalTrials.gov API unavailable

**Example**:
```json
{
  "success": false,
  "error": {
    "code": "UNRESOLVED_ENTITY",
    "message": "get_trial requires a resolved NCT ID in CURIE format (NCT:########), not a query string.",
    "recovery_hint": "Use search_trials to find trials matching 'breast cancer', then call get_trial with the NCT ID from search results.",
    "invalid_input": "breast cancer"
  }
}
```

---

## Field Mappings

### API → Agentic Biolink Schema

| API Field | Agentic Field | Notes |
|-----------|---------------|-------|
| `protocolSection.identificationModule.nctId` | `id` | Normalized to NCT:######## CURIE |
| `protocolSection.identificationModule.officialTitle` | `title` | Official title |
| `protocolSection.descriptionModule.briefSummary` | `brief_summary` | Plain text |
| `protocolSection.descriptionModule.detailedDescription` | `detailed_description` | Omit if missing |
| `protocolSection.designModule.studyType` | `protocol.study_type` | INTERVENTIONAL, OBSERVATIONAL, etc. |
| `protocolSection.designModule.phases` | `phase` | First phase from array |
| `protocolSection.statusModule.overallStatus` | `status` | RECRUITING, COMPLETED, etc. |
| `protocolSection.statusModule.startDateStruct.date` | `start_date` | YYYY-MM-DD or YYYY-MM |
| `protocolSection.statusModule.primaryCompletionDateStruct.date` | `completion_date` | YYYY-MM-DD or YYYY-MM |
| `protocolSection.statusModule.lastUpdatePostDateStruct.date` | `last_update_date` | YYYY-MM-DD |
| `protocolSection.designModule.enrollmentInfo.count` | `enrollment` | Integer |
| `protocolSection.sponsorCollaboratorsModule.leadSponsor` | `sponsors[0]` | role=LEAD_SPONSOR |
| `protocolSection.sponsorCollaboratorsModule.collaborators` | `sponsors[1:]` | role=COLLABORATOR |
| `protocolSection.outcomesModule.primaryOutcomes` | `primary_outcomes` | Array |
| `protocolSection.outcomesModule.secondaryOutcomes` | `secondary_outcomes` | Array |
| `protocolSection.eligibilityModule.eligibilityCriteria` | `eligibility_criteria.criteria_text` | Formatted text |
| `protocolSection.eligibilityModule.minimumAge` | `eligibility_criteria.minimum_age` | String |
| `protocolSection.eligibilityModule.sex` | `eligibility_criteria.sex` | ALL, FEMALE, MALE |
| `protocolSection.conditionsModule.conditions` | `conditions` | Array (search only) |
| `protocolSection.armsInterventionsModule.interventions[].name` | `interventions` | Array (search only) |
| `derivedSection.miscInfoModule.versionHolder` | - | Ignored |
| `derivedSection.conditionBrowseModule.meshes` | `cross_references.mesh_conditions` | MeSH ID |
| `derivedSection.interventionBrowseModule.meshes` | `cross_references.mesh_interventions` | MeSH ID |

### Location API → TrialLocation

| API Field | Model Field | Notes |
|-----------|-------------|-------|
| `protocolSection.contactsLocationsModule.locations[].facility` | `facility_name` | Required |
| `protocolSection.contactsLocationsModule.locations[].city` | `city` | Required |
| `protocolSection.contactsLocationsModule.locations[].state` | `state` | Omit if missing |
| `protocolSection.contactsLocationsModule.locations[].country` | `country` | Required |
| `protocolSection.contactsLocationsModule.locations[].zip` | `zip` | Omit if missing |
| `protocolSection.contactsLocationsModule.locations[].contacts[0].name` | `contact_name` | First contact |
| `protocolSection.contactsLocationsModule.locations[].contacts[0].phoneNumber` | `contact_phone` | First contact |
| `protocolSection.contactsLocationsModule.locations[].contacts[0].email` | `contact_email` | First contact |
| `protocolSection.contactsLocationsModule.locations[].status` | `recruitment_status` | RECRUITING, etc. |

---

## Validation Rules

### NCT ID Format

**Pattern**: `^NCT:\d{8}$`

**Examples**:
- ✅ Valid: `NCT:00461032`, `NCT:04123456`
- ❌ Invalid: `NCT00461032` (missing colon), `NCT:0046103` (7 digits), `nct:00461032` (lowercase)

**Normalization**: Client must normalize raw NCT IDs (e.g., `NCT00461032`) to CURIE format (`NCT:00461032`) before validation.

### Omit-If-Null Pattern

**Rule**: If a field has no data, omit the key entirely from the JSON response. Never return `null` or empty string.

**Examples**:
```json
// ✅ Correct - detailed_description omitted
{
  "id": "NCT:00461032",
  "title": "...",
  "brief_summary": "..."
}

// ❌ Incorrect - null values waste tokens
{
  "id": "NCT:00461032",
  "title": "...",
  "brief_summary": "...",
  "detailed_description": null
}
```

### Date Formats

**API Returns**: Partial dates in `YYYY-MM-DD`, `YYYY-MM`, or `YYYY` format

**Storage**: Store as-is (string), do not parse to datetime

**Validation**: Accept any non-empty string for date fields

---

## Token Optimization

### Search Results (Fuzzy Phase)

**Strategy**: Use API `fields` parameter to request minimal data.

**Fields Requested**:
```python
fields = [
    "NCTId",
    "OfficialTitle",
    "BriefSummary",
    "Phase",
    "OverallStatus",
    "Condition",
    "InterventionName"
]
```

**Result**: ~100-200 tokens per TrialSearchCandidate (vs 5K-10K for full trial)

### Strict Lookup (Fact Phase)

**Strategy**: Request complete trial data but flatten nested JSON in Agentic Biolink schema.

**Challenge**: API returns 5 top-level sections with deep nesting (e.g., `protocolSection.sponsorCollaboratorsModule.leadSponsor.name`)

**Solution**: Flatten to 2-level structure in Agentic schema (e.g., `sponsors[0].name`)

**Token Budget**: ~5K-10K tokens per Trial (acceptable for strict lookups, not batch operations)

---

## Cross-Reference Strategy

### Limited External Mappings

Unlike gene/protein APIs (22 cross-reference keys), ClinicalTrials.gov provides limited external IDs:

**Available**:
- `pubmed` - PubMed IDs (when publications exist)
- `clinicaltrials_gov` - Registry URL (always)
- `mesh_conditions` - MeSH disease terms (when available)
- `mesh_interventions` - MeSH drug terms (when available)

**Not Available**:
- Direct links to DrugBank, ChEMBL, UniProt, HGNC, etc.
- Agents must query those APIs separately using intervention/condition names

### MeSH ID Extraction

**Source**: `derivedSection.conditionBrowseModule.meshes` and `derivedSection.interventionBrowseModule.meshes`

**Format**: Array of `{id: "D001943", term: "Breast Neoplasms"}`

**Mapping**: Extract first MeSH ID for each category and store in `cross_references.mesh_conditions` / `cross_references.mesh_interventions`

---

## Implementation Notes

### Pydantic Configuration

All models use:
```python
class Config:
    populate_by_name = True      # Allow field aliases
    validate_assignment = True   # Validate on attribute assignment
    extra = "forbid"             # Reject unknown fields
```

### Field Validators

Key validators:
- NCT ID format: `^NCT:\d{8}$`
- Phase enum: `PHASE1`, `PHASE2`, `PHASE3`, `PHASE4`, `NA`, `EARLY_PHASE1`
- Status enum: `RECRUITING`, `COMPLETED`, `ACTIVE_NOT_RECRUITING`, etc.

### Optional vs Required

**Required fields**: `id`, `title`, `brief_summary`, `status`

**Optional fields**: All others (omit if API returns null/missing)

---

## Testing Strategy

### Unit Tests

**File**: `tests/unit/test_trial_models.py`, `tests/unit/test_trial_location_models.py`

**Coverage**:
- Pydantic validation (valid/invalid NCT IDs, enums)
- Omit-if-null pattern compliance
- Field aliases and deserialization

### Integration Tests

**File**: `tests/integration/test_clinicaltrials_api.py`

**Coverage**:
- Round-trip API → Pydantic → JSON
- Real API responses from NCT00461032 (stable test trial)
- Cross-reference extraction

### Contract Tests

**File**: `tests/contract/test_canonical_envelopes.py`

**Coverage**:
- PaginationEnvelope structure compliance
- ErrorEnvelope structure compliance
- Cross-reference regex validation
