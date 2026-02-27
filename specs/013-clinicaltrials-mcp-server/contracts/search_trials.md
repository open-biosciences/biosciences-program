# Tool Contract: search_trials

**Type**: Fuzzy Discovery
**Protocol Phase**: Phase 1 (Fuzzy-to-Fact)
**Returns**: PaginationEnvelope[TrialSearchCandidate]

## Purpose

Search ClinicalTrials.gov database using natural language queries and filters. Returns ranked trial candidates with NCT IDs for strict lookup.

## Parameters

```python
{
    "query": str,                   # Natural language search query (optional if filters provided)
    "condition": str | None,        # Disease/condition filter
    "intervention": str | None,     # Treatment/drug filter
    "status": str | None,           # Recruitment status filter
    "location": str | None,         # Geographic location filter
    "phase": str | None,            # Trial phase filter
    "page_size": int,               # Results per page (default: 50, max: 200)
    "cursor": str | None            # Pagination cursor from previous response
}
```

### Parameter Details

#### query (optional)
- **Type**: String
- **Description**: Natural language search across titles, descriptions, conditions, interventions
- **Examples**:
  - `"breast cancer immunotherapy"`
  - `"heart disease prevention"`
  - `"diabetes insulin"`
- **Validation**: No special characters beyond alphanumeric and spaces
- **Required**: No (can search using filters alone)

#### condition (optional)
- **Type**: String
- **Description**: Disease or condition name filter
- **Examples**: `"breast cancer"`, `"diabetes"`, `"melanoma"`
- **API Mapping**: `query.cond` parameter
- **Required**: No

#### intervention (optional)
- **Type**: String
- **Description**: Treatment, drug, or intervention filter
- **Examples**: `"bevacizumab"`, `"insulin"`, `"radiation therapy"`
- **API Mapping**: `query.intr` parameter
- **Required**: No

#### status (optional)
- **Type**: String (enum)
- **Description**: Recruitment status filter
- **Valid Values**:
  - `"RECRUITING"`
  - `"COMPLETED"`
  - `"ACTIVE_NOT_RECRUITING"`
  - `"NOT_YET_RECRUITING"`
  - `"SUSPENDED"`
  - `"TERMINATED"`
  - `"WITHDRAWN"`
- **API Mapping**: `filter.overallStatus` parameter
- **Required**: No

#### location (optional)
- **Type**: String
- **Description**: Geographic location filter (city, state, or country)
- **Examples**: `"Boston, MA"`, `"California"`, `"United States"`
- **API Mapping**: `query.locn` parameter
- **Required**: No

#### phase (optional)
- **Type**: String (enum)
- **Description**: Clinical trial phase filter
- **Valid Values**:
  - `"EARLY_PHASE1"`
  - `"PHASE1"`
  - `"PHASE2"`
  - `"PHASE3"`
  - `"PHASE4"`
  - `"NA"` (not applicable)
- **API Mapping**: Extracted from `protocolSection.designModule.phases`
- **Required**: No

#### page_size (optional)
- **Type**: Integer
- **Description**: Number of results per page
- **Default**: 50
- **Range**: 1-200
- **Required**: No

#### cursor (optional)
- **Type**: String
- **Description**: Opaque pagination token from previous response
- **Source**: `pagination.cursor` from previous search_trials response
- **Required**: No (omit for first page)

## Response Schema

### Success Response

```json
{
  "items": [
    {
      "id": "NCT:00461032",
      "title": "Bevacizumab and Erlotinib in Treating Patients With Metastatic Breast Cancer",
      "brief_summary": "This phase II trial is studying how well giving bevacizumab...",
      "phase": "PHASE2",
      "status": "COMPLETED",
      "conditions": ["Breast Cancer", "Metastatic Breast Cancer"],
      "interventions": ["Bevacizumab", "Erlotinib Hydrochloride"]
    }
  ],
  "pagination": {
    "cursor": "eyJuZXh0UGFnZVRva2VuIjoiQUJDMTIzIn0=",
    "total_count": 1540,
    "page_size": 50
  }
}
```

### Error Responses

#### INVALID_INPUT
```json
{
  "success": false,
  "error": {
    "code": "INVALID_INPUT",
    "message": "Invalid phase value. Must be one of: EARLY_PHASE1, PHASE1, PHASE2, PHASE3, PHASE4, NA",
    "recovery_hint": "Check the phase parameter and use a valid value from the allowed list.",
    "invalid_input": "Phase 5"
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

### Ranking
- Results are ranked by relevance (API default: `@relevance` sort)
- Most relevant trials appear first in `items` array
- Relevance score NOT included in response (API limitation)

### Pagination
- First page: Omit `cursor` parameter
- Subsequent pages: Use `cursor` from previous response
- Last page: `cursor` will be `null`
- `total_count` may be `null` if API does not provide count

### Empty Results
```json
{
  "items": [],
  "pagination": {
    "cursor": null,
    "total_count": 0,
    "page_size": 50
  }
}
```

### Filter Combinations
- All filters are ANDed together (intersection)
- Example: `condition="diabetes" AND intervention="insulin" AND status="RECRUITING"`
- Empty `query` with filters is valid

### Field Omission
- If a TrialSearchCandidate has no `phase`, field is omitted (not `null`)
- `interventions` and `conditions` are always arrays (may be empty `[]`)

## Rate Limiting

- **Limit**: 1 request per second
- **Enforcement**: Client-side with exponential backoff
- **Backoff Strategy**:
  - 1st failure: Wait 1s
  - 2nd failure: Wait 2s
  - 3rd failure: Wait 4s
  - Max wait: 16s

## Examples

### Example 1: Simple Query
**Request**:
```json
{
  "query": "breast cancer immunotherapy",
  "page_size": 10
}
```

**Response**:
```json
{
  "items": [
    {
      "id": "NCT:04123456",
      "title": "Pembrolizumab for Metastatic Breast Cancer",
      "brief_summary": "This study evaluates pembrolizumab...",
      "phase": "PHASE3",
      "status": "RECRUITING",
      "conditions": ["Breast Cancer", "Triple Negative Breast Cancer"],
      "interventions": ["Pembrolizumab"]
    }
  ],
  "pagination": {
    "cursor": "abc123xyz",
    "total_count": 245,
    "page_size": 10
  }
}
```

### Example 2: Multi-Filter Search
**Request**:
```json
{
  "condition": "diabetes",
  "intervention": "insulin",
  "status": "RECRUITING",
  "phase": "PHASE3",
  "location": "Boston, MA"
}
```

**Response**: Returns recruiting Phase 3 diabetes trials testing insulin in Boston area.

### Example 3: Pagination
**Request (Page 1)**:
```json
{
  "query": "heart disease",
  "page_size": 50
}
```

**Response**:
```json
{
  "items": [...],  // 50 trials
  "pagination": {
    "cursor": "nextpage123",
    "total_count": 2500,
    "page_size": 50
  }
}
```

**Request (Page 2)**:
```json
{
  "query": "heart disease",
  "page_size": 50,
  "cursor": "nextpage123"
}
```

## Implementation Notes

### API Endpoint
```
GET https://clinicaltrials.gov/api/v2/studies
```

### Query Parameters
```python
params = {
    "query.term": query if query else None,
    "query.cond": condition,
    "query.intr": intervention,
    "filter.overallStatus": status,
    "query.locn": location,
    "pageSize": page_size,
    "pageToken": cursor,
    "countTotal": "true",
    "fields": "NCTId|OfficialTitle|BriefSummary|Phase|OverallStatus|Condition|InterventionName"
}
```

### Response Parsing
```python
# API returns
{
  "studies": [
    {
      "protocolSection": {
        "identificationModule": {"nctId": "NCT00461032"},
        ...
      }
    }
  ],
  "nextPageToken": "...",
  "totalCount": 1540
}

# Map to TrialSearchCandidate
id = f"NCT:{study['protocolSection']['identificationModule']['nctId'][3:]}"
title = study['protocolSection']['identificationModule']['officialTitle']
...
```

## Testing Requirements

### Unit Tests
- Parameter validation (valid/invalid enums)
- Empty query + filters validation
- Page size bounds checking

### Integration Tests
- **Test 1**: Search by query only
- **Test 2**: Search by condition + intervention
- **Test 3**: Multi-filter search (condition + phase + status + location)
- **Test 4**: Pagination (retrieve 100 results across 2 pages)
- **Test 5**: Empty results (obscure query)

### Contract Tests
- PaginationEnvelope structure compliance
- TrialSearchCandidate schema compliance
- Error envelope compliance
