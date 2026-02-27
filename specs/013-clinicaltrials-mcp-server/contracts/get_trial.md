# Tool Contract: get_trial

**Type**: Strict Lookup
**Protocol Phase**: Phase 2 (Fuzzy-to-Fact)
**Returns**: Trial

## Purpose

Retrieve complete trial information using a resolved NCT ID. Returns full trial details including protocol, eligibility criteria, outcomes, sponsors, dates, and cross-references.

## Parameters

```python
{
    "nct_id": str  # NCT CURIE format (e.g., "NCT:00461032")
}
```

### Parameter Details

#### nct_id (required)
- **Type**: String (CURIE format)
- **Description**: Resolved NCT identifier from search_trials
- **Format**: `NCT:########` (8 digits)
- **Regex**: `^NCT:\d{8}$`
- **Examples**:
  - ✅ Valid: `"NCT:00461032"`, `"NCT:04123456"`
  - ❌ Invalid: `"NCT00461032"` (missing colon), `"breast cancer"` (query string), `"nct:00461032"` (lowercase)
- **Required**: Yes

## Response Schema

### Success Response

```json
{
  "id": "NCT:00461032",
  "title": "Bevacizumab and Erlotinib in Treating Patients With Metastatic Breast Cancer",
  "brief_summary": "This phase II trial is studying how well giving bevacizumab together...",
  "detailed_description": "RATIONALE: Monoclonal antibodies, such as bevacizumab...",
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
    "maximum_age": "N/A",
    "sex": "FEMALE",
    "accepts_healthy_volunteers": false
  },
  "primary_outcomes": [
    {
      "measure": "Objective Response Rate",
      "time_frame": "6 months",
      "description": "Response evaluated by RECIST 1.1"
    }
  ],
  "secondary_outcomes": [
    {
      "measure": "Progression-Free Survival",
      "time_frame": "2 years"
    }
  ],
  "sponsors": [
    {
      "name": "National Cancer Institute",
      "role": "LEAD_SPONSOR"
    }
  ],
  "phase": "PHASE2",
  "status": "COMPLETED",
  "enrollment": 50,
  "start_date": "2007-04",
  "completion_date": "2010-12",
  "last_update_date": "2015-03-15",
  "cross_references": {
    "pubmed": "23456789",
    "clinicaltrials_gov": "https://clinicaltrials.gov/study/NCT00461032",
    "mesh_conditions": "D001943"
  }
}
```

### Error Responses

#### UNRESOLVED_ENTITY
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

#### INVALID_INPUT
```json
{
  "success": false,
  "error": {
    "code": "INVALID_INPUT",
    "message": "Invalid NCT ID format. Expected NCT:######## with 8 digits.",
    "recovery_hint": "Ensure NCT ID follows the format NCT:######## (e.g., NCT:00461032). If you have a raw NCT ID without the colon, add 'NCT:' prefix.",
    "invalid_input": "NCT0046103"
  }
}
```

#### ENTITY_NOT_FOUND
```json
{
  "success": false,
  "error": {
    "code": "ENTITY_NOT_FOUND",
    "message": "Trial NCT:99999999 not found in ClinicalTrials.gov database.",
    "recovery_hint": "Verify the NCT ID is correct. Use search_trials to find valid trials if uncertain.",
    "invalid_input": "NCT:99999999"
  }
}
```

#### RATE_LIMITED
```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded (1 req/sec). Too many requests in short time.",
    "recovery_hint": "Wait 1 second and retry. Implement exponential backoff for multiple failures.",
    "invalid_input": null
  }
}
```

#### UPSTREAM_ERROR
```json
{
  "success": false,
  "error": {
    "code": "UPSTREAM_ERROR",
    "message": "ClinicalTrials.gov API returned 503 Service Unavailable",
    "recovery_hint": "Retry after 30 seconds. Check https://clinicaltrials.gov/data-api/about-api/status for API status.",
    "invalid_input": null
  }
}
```

## Behavior Specifications

### Field Omission (Omit-If-Null Pattern)
Missing optional fields are omitted entirely (not set to `null`):

**Example**: Trial with no detailed description
```json
{
  "id": "NCT:12345678",
  "title": "...",
  "brief_summary": "...",
  // detailed_description omitted (not null)
  "protocol": {...}
}
```

### Nested Objects
All nested objects (protocol, eligibility_criteria, outcomes, sponsors) follow the same omit-if-null pattern.

**Example**: Trial with no eligibility age limits
```json
{
  "eligibility_criteria": {
    "criteria_text": "...",
    "sex": "ALL",
    "accepts_healthy_volunteers": false
    // minimum_age and maximum_age omitted
  }
}
```

### Cross-References
Limited cross-reference data compared to gene/protein APIs:

**Available**:
- `pubmed` - PubMed ID (if publications exist)
- `clinicaltrials_gov` - Registry URL (always present)
- `mesh_conditions` - MeSH disease term ID (if available)
- `mesh_interventions` - MeSH drug term ID (if available)

**Not Available**:
- DrugBank, ChEMBL, UniProt, HGNC (agents must query separately)

### Date Formats
Dates are stored as-is from API (partial dates allowed):
- `"2007-04"` (year-month)
- `"2010-12-15"` (full date)
- `"2007"` (year only)

## Rate Limiting

- **Limit**: 1 request per second
- **Enforcement**: Client-side with exponential backoff
- **Backoff Strategy**:
  - 1st failure: Wait 1s
  - 2nd failure: Wait 2s
  - 3rd failure: Wait 4s
  - Max wait: 16s

## Examples

### Example 1: Successful Lookup
**Request**:
```json
{
  "nct_id": "NCT:00461032"
}
```

**Response**: See success response schema above.

### Example 2: Invalid Format (Missing Colon)
**Request**:
```json
{
  "nct_id": "NCT00461032"
}
```

**Response**:
```json
{
  "success": false,
  "error": {
    "code": "INVALID_INPUT",
    "message": "Invalid NCT ID format. Expected NCT:######## with 8 digits.",
    "recovery_hint": "Ensure NCT ID follows the format NCT:######## (e.g., NCT:00461032)...",
    "invalid_input": "NCT00461032"
  }
}
```

### Example 3: Query String Instead of NCT ID
**Request**:
```json
{
  "nct_id": "breast cancer"
}
```

**Response**:
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

### Example 4: Non-Existent Trial
**Request**:
```json
{
  "nct_id": "NCT:99999999"
}
```

**Response**:
```json
{
  "success": false,
  "error": {
    "code": "ENTITY_NOT_FOUND",
    "message": "Trial NCT:99999999 not found in ClinicalTrials.gov database.",
    "recovery_hint": "Verify the NCT ID is correct. Use search_trials to find valid trials if uncertain.",
    "invalid_input": "NCT:99999999"
  }
}
```

## Implementation Notes

### API Endpoint
```
GET https://clinicaltrials.gov/api/v2/studies/{nctId}
```

### NCT ID Normalization
```python
# Input: "NCT:00461032"
# Extract raw NCT ID: "NCT00461032"
raw_nct_id = nct_id.replace("NCT:", "NCT")

# API request
url = f"https://clinicaltrials.gov/api/v2/studies/{raw_nct_id}"
```

### Response Parsing
```python
# API returns nested structure:
{
  "protocolSection": {
    "identificationModule": {"nctId": "NCT00461032", "officialTitle": "..."},
    "statusModule": {"overallStatus": "COMPLETED", ...},
    "sponsorCollaboratorsModule": {"leadSponsor": {...}},
    ...
  },
  "derivedSection": {...}
}

# Flatten to Agentic Biolink schema
trial = Trial(
    id=f"NCT:{response['protocolSection']['identificationModule']['nctId'][3:]}",
    title=response['protocolSection']['identificationModule']['officialTitle'],
    ...
)
```

### Field Extraction
```python
# Safe field extraction with omit-if-null
def extract_optional(data: dict, *keys):
    """Extract nested field, return None if missing."""
    value = data
    for key in keys:
        value = value.get(key)
        if value is None:
            return None
    return value

# Example
detailed_description = extract_optional(
    response,
    "protocolSection",
    "descriptionModule",
    "detailedDescription"
)
# Only include in Trial if not None
```

## Testing Requirements

### Unit Tests
- NCT ID format validation (valid/invalid formats)
- UNRESOLVED_ENTITY error for query strings
- Omit-if-null pattern compliance

### Integration Tests
- **Test 1**: Successful lookup for NCT:00461032 (stable test trial)
- **Test 2**: ENTITY_NOT_FOUND for NCT:99999999
- **Test 3**: INVALID_INPUT for malformed NCT ID
- **Test 4**: UNRESOLVED_ENTITY for query string
- **Test 5**: Trial with missing optional fields (verify omit-if-null)

### Contract Tests
- Trial schema compliance
- Cross-reference structure compliance
- Error envelope compliance

## Fuzzy-to-Fact Workflow

**Phase 1 (Fuzzy)**:
```json
// search_trials("breast cancer immunotherapy")
{
  "items": [
    {"id": "NCT:04123456", "title": "...", ...}
  ]
}
```

**Phase 2 (Strict)**:
```json
// get_trial("NCT:04123456")
{
  "id": "NCT:04123456",
  "title": "...",
  "protocol": {...},
  "eligibility_criteria": {...},
  ...
}
```

## Token Budget

**Expected**: 5K-10K tokens per full trial response

**Mitigation**: This is acceptable for strict lookups (not batch operations). Agents should use `get_trial` selectively after fuzzy search, not for bulk data retrieval.
