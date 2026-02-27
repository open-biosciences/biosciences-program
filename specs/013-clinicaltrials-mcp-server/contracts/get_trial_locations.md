# Tool Contract: get_trial_locations

**Type**: Strict Lookup
**Protocol Phase**: Phase 2 (Fact Retrieval)
**Returns**: list[TrialLocation]

## Purpose

Retrieve geographic facility and contact information for a specific trial. Enables location-based filtering, patient recruitment workflows, and investigator discovery.

## Parameters

```python
{
    "nct_id": str  # NCT CURIE format (e.g., "NCT:00461032")
}
```

### Parameter Details

#### nct_id (required)
- **Type**: String (CURIE format)
- **Description**: Resolved NCT identifier
- **Format**: `NCT:########` (8 digits)
- **Regex**: `^NCT:\d{8}$`
- **Examples**:
  - ✅ Valid: `"NCT:00461032"`, `"NCT:04123456"`
  - ❌ Invalid: `"NCT00461032"` (missing colon), `"breast cancer"` (query string)
- **Required**: Yes

## Response Schema

### Success Response

```json
[
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
  },
  {
    "facility_name": "Memorial Sloan Kettering Cancer Center",
    "city": "New York",
    "state": "New York",
    "country": "United States",
    "zip": "10065",
    "contact_name": "Dr. John Doe",
    "contact_phone": "212-555-0456",
    "recruitment_status": "RECRUITING"
  }
]
```

### Empty Response (No Facilities)

```json
[]
```

**Note**: Some trials have no facility data (e.g., observational studies, registry studies). This is NOT an error.

### Error Responses

#### UNRESOLVED_ENTITY
```json
{
  "success": false,
  "error": {
    "code": "UNRESOLVED_ENTITY",
    "message": "get_trial_locations requires a resolved NCT ID in CURIE format (NCT:########), not a query string.",
    "recovery_hint": "Use search_trials to find trials, then call get_trial_locations with the NCT ID from search results.",
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
    "recovery_hint": "Ensure NCT ID follows the format NCT:######## (e.g., NCT:00461032).",
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
Missing optional contact fields are omitted entirely:

**Example**: Location with no contact information
```json
{
  "facility_name": "General Hospital",
  "city": "Chicago",
  "country": "United States",
  "recruitment_status": "NOT_YET_RECRUITING"
  // state, zip, contact_name, contact_phone, contact_email omitted
}
```

### State Field
- Present for US and Canadian facilities
- Omitted for other countries

### Contact Information
- `contact_name`, `contact_phone`, `contact_email` may all be missing
- If multiple contacts exist, first contact is used
- Missing contact fields are omitted (not `null`)

### Recruitment Status
**Valid Values**:
- `"RECRUITING"`
- `"NOT_YET_RECRUITING"`
- `"ACTIVE_NOT_RECRUITING"`
- `"COMPLETED"`
- `"SUSPENDED"`
- `"TERMINATED"`
- `"WITHDRAWN"`

**Note**: Facility status may differ from overall trial status.

### Empty Facility List
Trials may have no facilities for several reasons:
- Observational studies without intervention sites
- Registry studies (data collection only)
- Trials not yet enrolling (facilities TBD)
- Withdrawn trials before site activation

**Response**: Return empty array `[]`, NOT an error.

## Rate Limiting

- **Limit**: 1 request per second
- **Enforcement**: Client-side with exponential backoff
- **Backoff Strategy**:
  - 1st failure: Wait 1s
  - 2nd failure: Wait 2s
  - 3rd failure: Wait 4s
  - Max wait: 16s

## Examples

### Example 1: Multi-Site Trial
**Request**:
```json
{
  "nct_id": "NCT:00461032"
}
```

**Response**:
```json
[
  {
    "facility_name": "Dana-Farber Cancer Institute",
    "city": "Boston",
    "state": "Massachusetts",
    "country": "United States",
    "zip": "02215",
    "contact_name": "Clinical Trials Office",
    "contact_phone": "617-632-3476",
    "contact_email": "trials@dfci.harvard.edu",
    "recruitment_status": "COMPLETED"
  },
  {
    "facility_name": "Memorial Sloan Kettering Cancer Center",
    "city": "New York",
    "state": "New York",
    "country": "United States",
    "zip": "10065",
    "contact_name": "Protocol Office",
    "contact_phone": "212-639-7592",
    "recruitment_status": "COMPLETED"
  }
]
```

### Example 2: International Trial
**Request**:
```json
{
  "nct_id": "NCT:01234567"
}
```

**Response**:
```json
[
  {
    "facility_name": "Royal Marsden Hospital",
    "city": "London",
    "country": "United Kingdom",
    "recruitment_status": "RECRUITING"
  },
  {
    "facility_name": "Institut Gustave Roussy",
    "city": "Villejuif",
    "country": "France",
    "zip": "94805",
    "recruitment_status": "RECRUITING"
  }
]
```

### Example 3: Trial with No Facilities
**Request**:
```json
{
  "nct_id": "NCT:05555555"
}
```

**Response**:
```json
[]
```

### Example 4: Invalid NCT ID
**Request**:
```json
{
  "nct_id": "NCT99999999"
}
```

**Response**:
```json
{
  "success": false,
  "error": {
    "code": "INVALID_INPUT",
    "message": "Invalid NCT ID format. Expected NCT:######## with 8 digits.",
    "recovery_hint": "Ensure NCT ID follows the format NCT:######## (e.g., NCT:00461032).",
    "invalid_input": "NCT99999999"
  }
}
```

## Implementation Notes

### API Endpoint
```
GET https://clinicaltrials.gov/api/v2/studies/{nctId}
```

### Field Selection
Request only location-related fields to minimize response size:
```python
fields = [
    "ContactsLocationsModule"
]
```

### Response Parsing
```python
# API returns
{
  "protocolSection": {
    "contactsLocationsModule": {
      "locations": [
        {
          "facility": "Dana-Farber Cancer Institute",
          "city": "Boston",
          "state": "Massachusetts",
          "country": "United States",
          "zip": "02215",
          "contacts": [
            {
              "name": "Clinical Trials Office",
              "phone": "617-632-3476",
              "email": "trials@dfci.harvard.edu"
            }
          ],
          "status": "COMPLETED"
        }
      ]
    }
  }
}

# Map to TrialLocation
locations = []
for loc in api_response["protocolSection"]["contactsLocationsModule"]["locations"]:
    first_contact = loc.get("contacts", [{}])[0]

    location = TrialLocation(
        facility_name=loc["facility"],
        city=loc["city"],
        state=loc.get("state"),  # Omit if missing
        country=loc["country"],
        zip=loc.get("zip"),
        contact_name=first_contact.get("name"),
        contact_phone=first_contact.get("phone"),
        contact_email=first_contact.get("email"),
        recruitment_status=loc["status"]
    )
    locations.append(location)
```

### NCT ID Normalization
```python
# Input: "NCT:00461032"
# Extract raw NCT ID: "NCT00461032"
raw_nct_id = nct_id.replace("NCT:", "NCT")

# API request
url = f"https://clinicaltrials.gov/api/v2/studies/{raw_nct_id}?fields=ContactsLocationsModule"
```

## Testing Requirements

### Unit Tests
- NCT ID format validation
- Omit-if-null pattern for contact fields
- Empty location list handling

### Integration Tests
- **Test 1**: Multi-site trial with complete contact info (NCT:00461032)
- **Test 2**: International trial (verify state omission for non-US)
- **Test 3**: Trial with no facilities (empty array response)
- **Test 4**: Trial with partial contact info (verify omit-if-null)
- **Test 5**: Invalid NCT ID (INVALID_INPUT error)
- **Test 6**: Non-existent trial (ENTITY_NOT_FOUND error)

### Contract Tests
- TrialLocation schema compliance
- Error envelope compliance

## Use Cases

### Patient Recruitment
```python
# Find trials near patient location
locations = get_trial_locations("NCT:04123456")
boston_sites = [loc for loc in locations if loc.city == "Boston"]

# Contact coordinator
if boston_sites:
    contact = boston_sites[0].contact_email
```

### Geographic Analysis
```python
# Analyze trial site distribution
all_locations = []
for trial in search_results:
    locations = get_trial_locations(trial.id)
    all_locations.extend(locations)

# Count by country
from collections import Counter
country_counts = Counter(loc.country for loc in all_locations)
```

### Investigator Discovery
```python
# Find principal investigators
locations = get_trial_locations("NCT:00461032")
for loc in locations:
    if loc.contact_name:
        print(f"{loc.contact_name} at {loc.facility_name}")
```

## Token Budget

**Expected**: ~50-100 tokens per TrialLocation

**Typical Response**: 3-10 facilities per trial = ~500-1000 tokens

**Large Trials**: 50+ sites = ~5K tokens (acceptable for targeted lookups)
