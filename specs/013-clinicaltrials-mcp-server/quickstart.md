# Quickstart: ClinicalTrials.gov MCP Server

**Feature**: 013-clinicaltrials-mcp-server
**Version**: v0.1.0
**Status**: Specification Phase

## Overview

The ClinicalTrials.gov MCP server provides LLM agents with access to 450K+ clinical trials via three tools:

1. **search_trials** - Fuzzy discovery using natural language queries
2. **get_trial** - Strict lookup by NCT ID for complete trial details
3. **get_trial_locations** - Geographic facility and contact information

All responses use Canonical Envelopes (PaginationEnvelope, ErrorEnvelope) with Agentic Biolink schema.

## Installation (Future)

```bash
# Install from repository
cd lifesciences-research
uv sync

# Run the server
uv run fastmcp run src/lifesciences_mcp/servers/clinicaltrials.py
```

## Basic Workflows

### Workflow 1: Simple Trial Search

**Goal**: Find recruiting breast cancer trials

**Code**:
```python
from lifesciences_mcp.clients.clinicaltrials import ClinicalTrialsClient

async with ClinicalTrialsClient() as client:
    # Fuzzy search for trials
    result = await client.search_trials(
        query="breast cancer",
        status="RECRUITING",
        page_size=10
    )

    # Print top results
    for trial in result["items"]:
        print(f"{trial['id']}: {trial['title']}")
        print(f"  Phase: {trial.get('phase', 'N/A')}")
        print(f"  Status: {trial['status']}\n")
```

**Expected Output**:
```
NCT:04123456: Pembrolizumab for Metastatic Breast Cancer
  Phase: PHASE3
  Status: RECRUITING

NCT:05234567: Trastuzumab Deruxtecan in HER2+ Breast Cancer
  Phase: PHASE2
  Status: RECRUITING
...
```

---

### Workflow 2: Fuzzy-to-Fact Protocol

**Goal**: Search for trials, then get complete details

**Code**:
```python
async with ClinicalTrialsClient() as client:
    # Phase 1: Fuzzy discovery
    search_result = await client.search_trials(
        condition="melanoma",
        intervention="immunotherapy",
        phase="PHASE3",
        page_size=5
    )

    if search_result["items"]:
        top_trial = search_result["items"][0]
        print(f"Found: {top_trial['id']} - {top_trial['title']}\n")

        # Phase 2: Strict lookup
        trial = await client.get_trial(top_trial["id"])

        # Print detailed information
        print(f"Title: {trial['title']}")
        print(f"Phase: {trial.get('phase', 'N/A')}")
        print(f"Status: {trial['status']}")
        print(f"Enrollment: {trial.get('enrollment', 'N/A')}")

        if trial.get("eligibility_criteria"):
            print(f"\nEligibility:")
            print(f"  Age: {trial['eligibility_criteria'].get('minimum_age', 'N/A')} - {trial['eligibility_criteria'].get('maximum_age', 'N/A')}")
            print(f"  Sex: {trial['eligibility_criteria'].get('sex', 'N/A')}")

        print(f"\nPrimary Outcomes:")
        for outcome in trial.get("primary_outcomes", []):
            print(f"  - {outcome['measure']}")

        print(f"\nSponsors:")
        for sponsor in trial.get("sponsors", []):
            print(f"  - {sponsor['name']} ({sponsor['role']})")
```

**Expected Output**:
```
Found: NCT:03513068 - Nivolumab vs Dacarbazine in Advanced Melanoma

Title: Nivolumab vs Dacarbazine in Advanced Melanoma
Phase: PHASE3
Status: COMPLETED
Enrollment: 418

Eligibility:
  Age: 18 Years - N/A
  Sex: ALL

Primary Outcomes:
  - Overall Survival
  - Progression-Free Survival

Sponsors:
  - Bristol-Myers Squibb (LEAD_SPONSOR)
```

---

### Workflow 3: Geographic Trial Search

**Goal**: Find trials near a specific location with contact information

**Code**:
```python
async with ClinicalTrialsClient() as client:
    # Search trials by location
    result = await client.search_trials(
        condition="diabetes",
        location="Boston, MA",
        status="RECRUITING",
        page_size=10
    )

    # Get locations for first trial
    if result["items"]:
        trial = result["items"][0]
        print(f"Trial: {trial['id']} - {trial['title']}\n")

        locations = await client.get_trial_locations(trial["id"])

        print(f"Found {len(locations)} facilities:\n")
        for loc in locations:
            print(f"{loc['facility_name']}")
            print(f"  {loc['city']}, {loc.get('state', '')}")
            if loc.get("contact_name"):
                print(f"  Contact: {loc['contact_name']}")
            if loc.get("contact_email"):
                print(f"  Email: {loc['contact_email']}")
            print(f"  Status: {loc['recruitment_status']}\n")
```

**Expected Output**:
```
Trial: NCT:04567890 - Metformin vs Placebo in Type 2 Diabetes

Found 3 facilities:

Joslin Diabetes Center
  Boston, Massachusetts
  Contact: Dr. Sarah Johnson
  Email: trials@joslin.harvard.edu
  Status: RECRUITING

Massachusetts General Hospital
  Boston, Massachusetts
  Contact: Clinical Research Coordinator
  Email: diabetes@mgh.harvard.edu
  Status: RECRUITING

Beth Israel Deaconess Medical Center
  Boston, Massachusetts
  Status: RECRUITING
```

---

### Workflow 4: Multi-Filter Advanced Search

**Goal**: Find trials matching multiple specific criteria

**Code**:
```python
async with ClinicalTrialsClient() as client:
    # Complex multi-filter search
    result = await client.search_trials(
        condition="non-small cell lung cancer",
        intervention="pembrolizumab",
        phase="PHASE3",
        status="RECRUITING",
        page_size=20
    )

    print(f"Total trials found: {result['pagination'].get('total_count', 'Unknown')}\n")

    for trial in result["items"]:
        print(f"{trial['id']}: {trial['title']}")
        print(f"  Conditions: {', '.join(trial['conditions'][:3])}")
        print(f"  Interventions: {', '.join(trial['interventions'][:3])}")
        print(f"  Status: {trial['status']}\n")
```

**Expected Output**:
```
Total trials found: 12

NCT:03409614: Pembrolizumab vs Chemotherapy in NSCLC
  Conditions: Non-Small Cell Lung Cancer, Lung Cancer, Carcinoma
  Interventions: Pembrolizumab, Carboplatin, Pemetrexed
  Status: RECRUITING

NCT:02775435: Pembrolizumab + Chemotherapy in Advanced NSCLC
  Conditions: Non-Small Cell Lung Cancer, Lung Neoplasms
  Interventions: Pembrolizumab, Platinum-based Chemotherapy
  Status: RECRUITING
...
```

---

### Workflow 5: Pagination

**Goal**: Retrieve all matching trials across multiple pages

**Code**:
```python
async with ClinicalTrialsClient() as client:
    all_trials = []
    cursor = None

    while True:
        # Fetch page
        result = await client.search_trials(
            condition="heart disease",
            status="RECRUITING",
            page_size=50,
            cursor=cursor
        )

        all_trials.extend(result["items"])
        print(f"Retrieved {len(result['items'])} trials (total: {len(all_trials)})")

        # Check for more pages
        cursor = result["pagination"].get("cursor")
        if not cursor:
            break

    print(f"\nFinal count: {len(all_trials)} trials")
```

**Expected Output**:
```
Retrieved 50 trials (total: 50)
Retrieved 50 trials (total: 100)
Retrieved 50 trials (total: 150)
Retrieved 23 trials (total: 173)

Final count: 173 trials
```

---

## Error Recovery

### Scenario 1: Query String to Strict Tool

**Mistake**: Passing a query string to `get_trial`

**Code**:
```python
async with ClinicalTrialsClient() as client:
    try:
        # ❌ Wrong: passing query string to strict tool
        trial = await client.get_trial("breast cancer")
    except Exception as e:
        print(f"Error: {e}")
```

**Error Response**:
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

**Recovery**:
```python
async with ClinicalTrialsClient() as client:
    # ✅ Correct: search first, then get details
    search_result = await client.search_trials(query="breast cancer")
    if search_result["items"]:
        trial = await client.get_trial(search_result["items"][0]["id"])
```

---

### Scenario 2: Invalid NCT ID Format

**Mistake**: Using raw NCT ID without CURIE format

**Code**:
```python
async with ClinicalTrialsClient() as client:
    try:
        # ❌ Wrong: missing colon in CURIE
        trial = await client.get_trial("NCT00461032")
    except Exception as e:
        print(f"Error: {e}")
```

**Error Response**:
```json
{
  "success": false,
  "error": {
    "code": "INVALID_INPUT",
    "message": "Invalid NCT ID format. Expected NCT:######## with 8 digits.",
    "recovery_hint": "Ensure NCT ID follows the format NCT:######## (e.g., NCT:00461032). If you have a raw NCT ID without the colon, add 'NCT:' prefix.",
    "invalid_input": "NCT00461032"
  }
}
```

**Recovery**:
```python
async with ClinicalTrialsClient() as client:
    # ✅ Correct: use CURIE format
    trial = await client.get_trial("NCT:00461032")
```

---

### Scenario 3: Rate Limit Exceeded

**Mistake**: Making too many requests too quickly

**Error Response**:
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

**Recovery**: Client automatically implements exponential backoff (1s → 2s → 4s → 8s → 16s)

---

### Scenario 4: Trial Not Found

**Mistake**: Using non-existent NCT ID

**Code**:
```python
async with ClinicalTrialsClient() as client:
    try:
        # ❌ Wrong: non-existent trial
        trial = await client.get_trial("NCT:99999999")
    except Exception as e:
        print(f"Error: {e}")
```

**Error Response**:
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

**Recovery**:
```python
async with ClinicalTrialsClient() as client:
    # ✅ Correct: verify NCT ID via search
    search_result = await client.search_trials(query="NCT99999999")
    if search_result["items"]:
        trial = await client.get_trial(search_result["items"][0]["id"])
    else:
        print("Trial not found in ClinicalTrials.gov")
```

---

## Cross-Database Integration

### Example: Link to PubMed Publications

**Code**:
```python
async with ClinicalTrialsClient() as client:
    trial = await client.get_trial("NCT:00461032")

    # Extract PubMed ID from cross-references
    pubmed_id = trial.get("cross_references", {}).get("pubmed")

    if pubmed_id:
        print(f"Trial publications:")
        print(f"  https://pubmed.ncbi.nlm.nih.gov/{pubmed_id}/")
    else:
        print("No publications linked to this trial")
```

---

### Example: Map to Drug Databases

**Code**:
```python
async with ClinicalTrialsClient() as client:
    # Search for drug trials
    result = await client.search_trials(
        intervention="bevacizumab",
        phase="PHASE3"
    )

    # Extract drug names for ChEMBL lookup
    for trial in result["items"]:
        print(f"{trial['id']}: {trial['title']}")
        print(f"  Interventions: {', '.join(trial['interventions'])}")

        # Agent would then call ChEMBL MCP server:
        # chembl_result = await chembl_client.search_compounds(query="bevacizumab")
```

**Note**: ClinicalTrials.gov does not provide direct ChEMBL/DrugBank IDs. Agents must query those APIs separately using intervention names.

---

## Advanced Queries

### Find Completed Trials with Results

**Code**:
```python
async with ClinicalTrialsClient() as client:
    result = await client.search_trials(
        condition="alzheimer disease",
        status="COMPLETED",
        page_size=20
    )

    for trial in result["items"]:
        details = await client.get_trial(trial["id"])

        # Check for results
        if details.get("primary_outcomes"):
            print(f"{trial['id']}: {trial['title']}")
            print(f"  Completion: {details.get('completion_date', 'N/A')}")
            print(f"  Primary Outcome: {details['primary_outcomes'][0]['measure']}\n")
```

---

### Filter by Eligibility Criteria

**Code**:
```python
async with ClinicalTrialsClient() as client:
    # Search pediatric trials
    result = await client.search_trials(
        condition="leukemia",
        status="RECRUITING"
    )

    # Filter by age criteria
    pediatric_trials = []
    for trial_candidate in result["items"]:
        trial = await client.get_trial(trial_candidate["id"])

        eligibility = trial.get("eligibility_criteria", {})
        min_age = eligibility.get("minimum_age", "")

        # Check if accepts children
        if "child" in min_age.lower() or ("year" in min_age and int(min_age.split()[0]) < 18):
            pediatric_trials.append(trial)

    print(f"Found {len(pediatric_trials)} pediatric leukemia trials")
```

---

## Performance Tips

### 1. Use Filters Over Query Strings

**Slow**:
```python
# API searches all text fields
result = await client.search_trials(query="recruiting breast cancer phase 2")
```

**Fast**:
```python
# API uses indexed filters
result = await client.search_trials(
    condition="breast cancer",
    phase="PHASE2",
    status="RECRUITING"
)
```

---

### 2. Request Minimal Fields for Search

The client automatically requests minimal fields for `search_trials` to reduce token usage:

- Search: ~100-200 tokens/trial (TrialSearchCandidate)
- Full lookup: ~5K-10K tokens/trial (Trial)

**Strategy**: Use `search_trials` to find relevant trials, then `get_trial` selectively for details.

---

### 3. Batch Location Lookups

Instead of calling `get_trial_locations` for each trial individually:

```python
# ❌ Slow: Sequential location lookups
for trial in search_results["items"]:
    locations = await client.get_trial_locations(trial["id"])
    # Process locations...
```

```python
# ✅ Fast: Gather concurrent requests (respects rate limiting)
import asyncio

location_tasks = [
    client.get_trial_locations(trial["id"])
    for trial in search_results["items"][:10]  # First 10 trials
]

all_locations = await asyncio.gather(*location_tasks)
```

**Note**: Client enforces 1 req/sec rate limit automatically with queuing.

---

## Testing Strategy

### Unit Tests

```python
# tests/unit/test_trial_models.py
def test_nct_id_validation():
    """NCT ID must match NCT:######## format."""
    # Valid
    trial = TrialSearchCandidate(
        id="NCT:00461032",
        title="...",
        brief_summary="...",
        status="RECRUITING",
        conditions=[],
        interventions=[]
    )

    # Invalid
    with pytest.raises(ValidationError):
        TrialSearchCandidate(
            id="NCT00461032",  # Missing colon
            ...
        )
```

---

### Integration Tests

```python
# tests/integration/test_clinicaltrials_api.py
@pytest.mark.asyncio
@pytest.mark.integration
async def test_fuzzy_to_fact_workflow():
    """Test complete Fuzzy-to-Fact workflow."""
    async with ClinicalTrialsClient() as client:
        # Fuzzy search
        search_result = await client.search_trials(
            query="breast cancer",
            page_size=5
        )
        assert len(search_result["items"]) > 0

        # Strict lookup
        trial = await client.get_trial(search_result["items"][0]["id"])
        assert trial["id"] == search_result["items"][0]["id"]
        assert "protocol" in trial
```

---

## Common Pitfalls

### ❌ Don't: Pass Query Strings to Strict Tools
```python
# Wrong
trial = await client.get_trial("breast cancer")
```

### ✅ Do: Use Fuzzy-to-Fact Protocol
```python
# Correct
search_result = await client.search_trials(query="breast cancer")
trial = await client.get_trial(search_result["items"][0]["id"])
```

---

### ❌ Don't: Ignore Empty Responses
```python
# Wrong - will raise IndexError
trial = await client.get_trial(search_result["items"][0]["id"])
```

### ✅ Do: Check for Empty Results
```python
# Correct
if search_result["items"]:
    trial = await client.get_trial(search_result["items"][0]["id"])
else:
    print("No trials found")
```

---

### ❌ Don't: Expect All Cross-References
```python
# Wrong - trial may not have PubMed link
pubmed_id = trial["cross_references"]["pubmed"]  # KeyError if missing
```

### ✅ Do: Use .get() for Optional Fields
```python
# Correct
pubmed_id = trial.get("cross_references", {}).get("pubmed")
if pubmed_id:
    # Use PubMed ID
```

---

## Next Steps

1. **Implementation**: See `specs/013-clinicaltrials-mcp-server/plan.md` for architecture details
2. **Data Model**: See `data-model.md` for Pydantic schemas
3. **API Contracts**: See `contracts/` for detailed tool specifications
4. **Testing**: See `tests/` for test suite (after implementation)

## Resources

- **ClinicalTrials.gov API Docs**: https://clinicaltrials.gov/data-api/api
- **Research Doc**: `research.md` (comprehensive API exploration)
- **ADR-001**: Agentic-First Architecture (Fuzzy-to-Fact protocol)
- **Constitution**: Life Sciences MCP principles
