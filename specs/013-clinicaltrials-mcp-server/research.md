# ClinicalTrials.gov API v2 Research

**Feature**: 013-clinicaltrials-mcp-server
**Date**: 2026-01-03
**Researcher**: Claude Code
**Purpose**: Comprehensive technical research for ClinicalTrials.gov MCP Server implementation

---

## Executive Summary

The ClinicalTrials.gov API v2 is a **public, REST-based API** using **OpenAPI 3.0 specification** that provides comprehensive access to clinical trial data. The API requires **no authentication**, supports **JSON, CSV, and FHIR formats**, and uses **cursor-based pagination**. Rate limiting is approximately **50 requests/minute per IP** (conservative implementation: 1 req/sec). The API provides rich search capabilities with multiple query parameters and returns structured trial data in nested JSON modules.

---

## 1. API Base URL and Endpoints

### Base URL
```
https://clinicaltrials.gov/api/v2
```

### Core Endpoints

| Endpoint | Method | Purpose | Example |
|----------|--------|---------|---------|
| `/studies` | GET | Search trials with filters | `/api/v2/studies?query.cond=diabetes&pageSize=20` |
| `/studies/{nctId}` | GET | Get single trial by NCT ID | `/api/v2/studies/NCT00461032` |
| `/studies/metadata` | GET | Get data model field definitions | `/api/v2/studies/metadata` |
| `/studies/search-areas` | GET | List available search areas | `/api/v2/studies/search-areas` |
| `/studies/enums` | GET | Get enumeration types/values | `/api/v2/studies/enums` |
| `/stats/size` | GET | JSON size statistics | `/api/v2/stats/size` |
| `/stats/field/values` | GET | Value statistics for fields | `/api/v2/stats/field/values` |
| `/stats/field/sizes` | GET | Array/list field sizes | `/api/v2/stats/field/sizes` |
| `/version` | GET | API and data version info | `/api/v2/version` |

**Key Endpoints for MCP Server:**
- **Primary**: `/studies` (search), `/studies/{nctId}` (get trial)
- **Supporting**: `/studies/metadata`, `/studies/enums`

---

## 2. Authentication

**Authentication**: **NONE REQUIRED**

The ClinicalTrials.gov API v2 is a **public API** with no authentication or API keys needed. Access is controlled solely via rate limiting.

**Implementation Note**: No authentication headers required in httpx client.

---

## 3. Rate Limiting

### Official Rate Limits

**Published Rate**: Approximately **50 requests per minute per IP address**

**Observed Recommendations**:
- Conservative: **1 request/second** (prevents thundering herd)
- Aggressive: **10 requests/second** (from third-party docs)

### Implementation Strategy

**Recommended for MCP Server**: **1 req/sec with exponential backoff**

```python
# Conservative rate limiting
async def _rate_limited_request(self, ...):
    await asyncio.sleep(1.0)  # 1 req/sec
    # Exponential backoff on 429 responses
```

**Rationale**:
- Prevents API throttling during agent workflows
- Allows 60 trials/minute (sufficient for most use cases)
- Matches patterns from UniProt/HGNC servers (ADR-001 compliance)
- Provides headroom for multiple concurrent agent sessions

---

## 4. Response Format

### Supported Formats

| Format | Parameter | Content-Type | Use Case |
|--------|-----------|--------------|----------|
| **JSON** (default) | `format=json` | `application/json` | Primary format for MCP server |
| **CSV** | `format=csv` | `text/csv` | Bulk export |
| **JSON.ZIP** | `format=json.zip` | `application/zip` | Large datasets |
| **FHIR JSON** | `format=fhir.json` | `application/fhir+json` | Healthcare systems |
| **RIS** | `format=ris` | `application/x-research-info-systems` | Reference management |

**Implementation**: Use `format=json` (default) for all MCP server tools.

### JSON Response Structure

All `/studies` responses follow this structure:

```json
{
  "studies": [
    {
      "protocolSection": { ... },
      "resultsSection": { ... },
      "derivedSection": { ... },
      "annotationSection": { ... },
      "documentSection": { ... }
    }
  ],
  "nextPageToken": "string|null",
  "totalCount": 12345
}
```

Individual `/studies/{nctId}` responses return a single study object:

```json
{
  "protocolSection": { ... },
  "resultsSection": { ... },
  "derivedSection": { ... },
  "annotationSection": { ... },
  "documentSection": { ... }
}
```

---

## 5. Pagination

### Mechanism: **Cursor-based with pageToken**

```python
# Initial request
GET /api/v2/studies?query.cond=diabetes&pageSize=20&countTotal=true

# Response
{
  "studies": [...],
  "nextPageToken": "eyJraW5kIjo...",  # Opaque cursor
  "totalCount": 1543
}

# Next page request
GET /api/v2/studies?query.cond=diabetes&pageSize=20&pageToken=eyJraW5kIjo...
```

### Pagination Parameters

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `pageSize` | integer | Results per page | 10 |
| `pageToken` | string | Cursor from previous response | null |
| `countTotal` | boolean | Include total count in response | false |

### Key Characteristics

- **Cursor-based**: Use `nextPageToken` from response as `pageToken` parameter
- **Opaque tokens**: Tokens are encoded strings (not page numbers)
- **Total count**: Optional via `countTotal=true` (may impact performance)
- **Last page**: `nextPageToken` is absent when no more results
- **Max page size**: 200 studies per page (observed from third-party implementations)

### Implementation for MCP Server

```python
async def search_trials(..., page_size: int = 50, page_token: str | None = None):
    params = {
        "query.cond": condition,
        "pageSize": page_size,
        "countTotal": "true",
        "format": "json"
    }
    if page_token:
        params["pageToken"] = page_token

    # Map to PaginationEnvelope
    return {
        "items": [...],
        "pagination": {
            "cursor": response.get("nextPageToken"),
            "total_count": response.get("totalCount"),
            "page_size": page_size
        }
    }
```

---

## 6. Search Parameters

### Query Parameters (Fuzzy Search)

All query parameters use `query.*` prefix for field-specific search:

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `query.cond` | string | Condition/disease search | `query.cond=diabetes` |
| `query.term` | string | Other terms (general search) | `query.term=immunotherapy` |
| `query.intr` | string | Intervention/treatment | `query.intr=insulin` |
| `query.locn` | string | Location terms | `query.locn=Boston` |
| `query.titles` | string | Title/acronym search | `query.titles=ASCEND` |
| `query.outc` | string | Outcome measure | `query.outc=mortality` |
| `query.spons` | string | Sponsor/collaborator | `query.spons=Pfizer` |
| `query.lead` | string | Lead sponsor name | `query.lead=NIH` |
| `query.id` | string | Study IDs (NCT, etc.) | `query.id=NCT04123456` |
| `query.patient` | string | Patient search | `query.patient=age>18` |

### Filter Parameters (Strict Filtering)

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `filter.overallStatus` | enum | Recruitment status | `filter.overallStatus=RECRUITING` |
| `filter.geo` | string | Distance-based location | `filter.geo=distance(42.36,-71.06,50mi)` |
| `filter.ids` | string | NCT ID filtering | `filter.ids=NCT04123456,NCT04123457` |
| `filter.advanced` | string | Essie expression syntax | `filter.advanced=AREA[Phase]Phase3` |
| `filter.synonyms` | string | Synonym-based filtering | `filter.synonyms=true` |

### Advanced Search Syntax (Essie)

The API supports **Essie expression syntax** for complex queries:

```
# Phase filtering
AREA[Phase]Phase3

# Multiple phases
AREA[Phase](Phase2 OR Phase3)

# Study type + phase combination
AREA[StudyType]Interventional AND (AREA[Phase]Phase3 OR AREA[Phase]Phase4)

# Date ranges
AREA[LastUpdatePostDate]RANGE[2023-06-29,MAX]
```

**Implementation Note**: For MCP server, focus on simple `query.*` parameters for User Story 1. Advanced Essie syntax is out of scope for v0.1.0.

### Response Control Parameters

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `fields` | string | Comma-separated field list | `fields=NCTId,BriefTitle,OverallStatus` |
| `sort` | string | Sort order | `sort=@relevance`, `sort=EnrollmentCount:desc` |
| `markupFormat` | string | Markup rendering | `markupFormat=markdown` |
| `format` | string | Output format | `format=json` (default) |

**Token Optimization**: Use `fields` parameter to limit payload size for large batch operations.

---

## 7. NCT ID Format

### Format Specification

**Pattern**: `NCT` + 8 digits

**Regex**: `^NCT\d{8}$`

**Examples**:
- ✅ Valid: `NCT00461032`, `NCT04123456`
- ❌ Invalid: `NCT123` (too short), `NCT123456789` (too long), `nct00461032` (lowercase)

### CURIE Format for MCP Server

**Internal Representation**: `NCT:########`

**Examples**:
- Input: `NCT00461032` → Normalized: `NCT:00461032`
- Input: `NCT:00461032` → Validated: `NCT:00461032`

**Validation Logic**:

```python
import re

def validate_nct_id(nct_id: str) -> str:
    """Validate and normalize NCT ID to CURIE format."""
    # Strip whitespace
    nct_id = nct_id.strip()

    # Remove colon if present
    if nct_id.startswith("NCT:"):
        nct_id = nct_id[4:]

    # Validate format
    if not re.match(r'^NCT\d{8}$', nct_id):
        raise ValueError(f"Invalid NCT ID format: {nct_id}")

    # Return CURIE
    return f"NCT:{nct_id[3:]}"  # NCT:00461032
```

---

## 8. Cross-References

### Available Cross-Reference Data

ClinicalTrials.gov provides **limited** cross-reference data compared to other life sciences APIs. Primary sources:

#### PubMed Links

**Location**: `derivedSection.miscInfoModule.references` or `protocolSection.referencesModule`

**Types**:
- **Abstract links**: Structured trial-article links from manuscript abstracts (ICMJE guidelines)
- **PMID links**: PI-specified publication references stored as `<result_reference/PMID>`

**Example Structure**:
```json
{
  "referencesModule": {
    "references": [
      {
        "pmid": "20674830",
        "type": "RESULT",
        "citation": "Smith J, et al. Trial results. N Engl J Med. 2010;363(5):411-22."
      }
    ]
  }
}
```

#### Registry Links

**Location**: `protocolSection.identificationModule`

**Fields**:
- `orgStudyIdInfo.id`: Sponsor's internal ID
- `secondaryIdInfos`: Other registry IDs (EudraCT, ISRCTN, etc.)
- `nctIdAliases`: Historical NCT IDs (for merged/updated trials)

**Example**:
```json
{
  "identificationModule": {
    "nctId": "NCT00461032",
    "orgStudyIdInfo": {
      "id": "P04279"
    },
    "secondaryIdInfos": [
      {
        "id": "2006-004123-42",
        "type": "EudraCT Number"
      }
    ]
  }
}
```

### Cross-Reference Mapping for Agentic Biolink

**Available Keys** (limited compared to gene/protein databases):

| Key | Source | Availability |
|-----|--------|--------------|
| `pubmed` | `referencesModule.references[].pmid` | When publications exist |
| `clinicaltrials_gov` | Always `https://clinicaltrials.gov/study/{nctId}` | Always |
| `eudract` | `identificationModule.secondaryIdInfos` | EU trials only |
| `isrctn` | `identificationModule.secondaryIdInfos` | UK trials only |

**Implementation Note**: Unlike gene/protein databases, ClinicalTrials.gov does not provide extensive cross-references to other databases (no direct links to DrugBank, ChEMBL, UniProt, etc.). Agents must use intervention names to query those APIs separately.

---

## 9. Response Data Structure

### Overview of JSON Sections

All trial records contain up to 5 top-level sections:

```json
{
  "protocolSection": { ... },      // ALWAYS PRESENT
  "resultsSection": { ... },       // Optional (when results posted)
  "derivedSection": { ... },       // ALWAYS PRESENT
  "annotationSection": { ... },    // Optional (for annotations)
  "documentSection": { ... }       // Optional (large documents)
}
```

### 9.1 protocolSection (Required)

Contains study design and planning information.

#### identificationModule

```json
{
  "identificationModule": {
    "nctId": "NCT00461032",
    "orgStudyIdInfo": {
      "id": "P04279"
    },
    "secondaryIdInfos": [...],
    "organization": {
      "fullName": "Organon and Company",
      "class": "INDUSTRY"
    },
    "briefTitle": "Montelukast Back to School Asthma Study",
    "officialTitle": "A Multicenter, Randomized, Double-Blind, Placebo-Controlled..."
  }
}
```

**Key Fields**:
- `nctId`: NCT identifier (string)
- `briefTitle`: Short title (string)
- `officialTitle`: Full scientific title (string)
- `organization.fullName`: Study organizer (string)
- `secondaryIdInfos`: Other registry IDs (array)

#### statusModule

```json
{
  "statusModule": {
    "statusVerifiedDate": "2006-11",
    "overallStatus": "COMPLETED",
    "expandedAccessInfo": {
      "hasExpandedAccess": false
    },
    "startDateStruct": {
      "date": "2006-06",
      "type": "ACTUAL"
    },
    "primaryCompletionDateStruct": {
      "date": "2006-11",
      "type": "ACTUAL"
    },
    "completionDateStruct": {
      "date": "2006-11",
      "type": "ACTUAL"
    },
    "studyFirstSubmitDate": "2007-04-13",
    "lastUpdatePostDateStruct": {
      "date": "2015-08-27",
      "type": "ESTIMATE"
    }
  }
}
```

**Key Fields**:
- `overallStatus`: Recruitment status (enum)
- `startDateStruct.date`: Trial start (ISO 8601 partial date)
- `completionDateStruct.date`: Trial completion (ISO 8601 partial date)
- `lastUpdatePostDateStruct.date`: Last registry update (ISO 8601 date)

**Status Values** (from OpenAPI spec):
- `ACTIVE_NOT_RECRUITING`
- `COMPLETED`
- `ENROLLING_BY_INVITATION`
- `NOT_YET_RECRUITING`
- `RECRUITING`
- `SUSPENDED`
- `TERMINATED`
- `WITHDRAWN`
- `AVAILABLE` (expanded access)
- `NO_LONGER_AVAILABLE`
- `TEMPORARILY_NOT_AVAILABLE`
- `APPROVED_FOR_MARKETING`
- `WITHHELD`
- `UNKNOWN`

#### sponsorCollaboratorsModule

```json
{
  "sponsorCollaboratorsModule": {
    "leadSponsor": {
      "name": "Organon and Company",
      "class": "INDUSTRY"
    },
    "collaborators": [
      {
        "name": "National Heart, Lung, and Blood Institute (NHLBI)",
        "class": "NIH"
      }
    ]
  }
}
```

**Key Fields**:
- `leadSponsor.name`: Primary sponsor (string)
- `collaborators[].name`: Supporting organizations (array)

#### descriptionModule

```json
{
  "descriptionModule": {
    "briefSummary": "The purpose of this study is to evaluate...",
    "detailedDescription": "Detailed explanation of study rationale..."
  }
}
```

**Key Fields**:
- `briefSummary`: Short description (CommonMark Markdown)
- `detailedDescription`: Long description (CommonMark Markdown)

#### conditionsModule

```json
{
  "conditionsModule": {
    "conditions": [
      "Asthma"
    ]
  }
}
```

**Key Fields**:
- `conditions[]`: Disease/condition names (array of strings)

#### designModule

```json
{
  "designModule": {
    "studyType": "INTERVENTIONAL",
    "phases": [
      "PHASE3"
    ],
    "designInfo": {
      "allocation": "RANDOMIZED",
      "interventionModel": "PARALLEL",
      "primaryPurpose": "TREATMENT",
      "maskingInfo": {
        "masking": "DOUBLE",
        "whoMasked": [
          "PARTICIPANT",
          "INVESTIGATOR"
        ]
      }
    },
    "enrollmentInfo": {
      "count": 1162,
      "type": "ACTUAL"
    }
  }
}
```

**Key Fields**:
- `studyType`: INTERVENTIONAL | OBSERVATIONAL | EXPANDED_ACCESS (enum)
- `phases[]`: Trial phase (array of enums)
- `enrollmentInfo.count`: Participant count (integer)

**Phase Values**:
- `NA`
- `EARLY_PHASE1`
- `PHASE1`
- `PHASE2`
- `PHASE3`
- `PHASE4`

#### armsInterventionsModule

```json
{
  "armsInterventionsModule": {
    "armGroups": [
      {
        "label": "Montelukast 5 mg",
        "type": "EXPERIMENTAL",
        "description": "Montelukast 5 mg chewable tablet...",
        "interventionNames": [
          "Drug: Montelukast 5 mg"
        ]
      },
      {
        "label": "Placebo",
        "type": "PLACEBO_COMPARATOR",
        "interventionNames": [
          "Drug: Placebo"
        ]
      }
    ],
    "interventions": [
      {
        "type": "DRUG",
        "name": "Montelukast 5 mg",
        "description": "Montelukast 5 mg chewable tablet...",
        "armGroupLabels": [
          "Montelukast 5 mg"
        ]
      }
    ]
  }
}
```

**Key Fields**:
- `interventions[].type`: DRUG | DEVICE | BIOLOGICAL | PROCEDURE | etc. (enum)
- `interventions[].name`: Intervention name (string)

#### outcomesModule

```json
{
  "outcomesModule": {
    "primaryOutcomes": [
      {
        "measure": "Mean Percentage of Days With Worsening Asthma",
        "description": "Worsening asthma was defined as...",
        "timeFrame": "8 weeks"
      }
    ],
    "secondaryOutcomes": [
      {
        "measure": "Percentage of Participants With Asthma Attacks",
        "timeFrame": "8 weeks"
      }
    ]
  }
}
```

**Key Fields**:
- `primaryOutcomes[].measure`: Primary endpoint (string)
- `secondaryOutcomes[].measure`: Secondary endpoints (array)

#### eligibilityModule

```json
{
  "eligibilityModule": {
    "eligibilityCriteria": "Inclusion Criteria:\n\n- Chronic asthma...",
    "healthyVolunteers": false,
    "sex": "ALL",
    "minimumAge": "6 Years",
    "maximumAge": "14 Years",
    "stdAges": [
      "CHILD"
    ]
  }
}
```

**Key Fields**:
- `eligibilityCriteria`: Full criteria text (CommonMark Markdown)
- `sex`: ALL | FEMALE | MALE (enum)
- `minimumAge` / `maximumAge`: Age range (string)

#### contactsLocationsModule

```json
{
  "contactsLocationsModule": {
    "locations": [
      {
        "facility": "Investigational Site 001",
        "city": "Birmingham",
        "state": "Alabama",
        "zip": "35209",
        "country": "United States",
        "geoPoint": {
          "lat": 33.45,
          "lon": -86.80
        }
      }
    ]
  }
}
```

**Key Fields**:
- `locations[].facility`: Site name (string)
- `locations[].city`, `state`, `country`: Geographic info (strings)
- `locations[].geoPoint`: Coordinates (lat/lon)

**Note**: Contact details (name, phone, email, recruitment status) are **not included** in this example. Check API metadata endpoint for complete schema.

---

### 9.2 resultsSection (Optional)

Contains trial outcomes and safety data. **Only present when results are posted.**

#### participantFlowModule

```json
{
  "participantFlowModule": {
    "preAssignmentDetails": "...",
    "recruitmentDetails": "165 sites in 7 countries enrolled...",
    "groups": [
      {
        "id": "FG000",
        "title": "Placebo",
        "description": "Placebo chewable tablet..."
      },
      {
        "id": "FG001",
        "title": "Montelukast 5 mg",
        "description": "Montelukast 5 mg chewable tablet..."
      }
    ],
    "periods": [
      {
        "title": "Overall Study",
        "milestones": [
          {
            "type": "STARTED",
            "achievements": [
              {"groupId": "FG000", "numSubjects": "582"},
              {"groupId": "FG001", "numSubjects": "580"}
            ]
          },
          {
            "type": "COMPLETED",
            "achievements": [
              {"groupId": "FG000", "numSubjects": "543"},
              {"groupId": "FG001", "numSubjects": "554"}
            ]
          }
        ],
        "dropWithdraws": [
          {
            "type": "Adverse Event",
            "reasons": [
              {"groupId": "FG000", "numSubjects": "6"},
              {"groupId": "FG001", "numSubjects": "3"}
            ]
          }
        ]
      }
    ]
  }
}
```

**Key Fields**:
- `groups[]`: Treatment arms (array)
- `periods[].milestones[]`: Enrollment flow (STARTED, COMPLETED, NOT_COMPLETED)
- `dropWithdraws[]`: Dropout reasons (array)

#### baselineCharacteristicsModule

```json
{
  "baselineCharacteristicsModule": {
    "groups": [...],
    "denoms": [
      {
        "units": "Participants",
        "counts": [
          {"groupId": "BG000", "value": "582"},
          {"groupId": "BG001", "value": "580"}
        ]
      }
    ],
    "measures": [
      {
        "title": "Age, Categorical",
        "paramType": "COUNT_OF_PARTICIPANTS",
        "unitOfMeasure": "Participants",
        "classes": [
          {
            "title": "<=18 years",
            "categories": [
              {
                "measurements": [
                  {"groupId": "BG000", "value": "582"},
                  {"groupId": "BG001", "value": "580"}
                ]
              }
            ]
          }
        ]
      }
    ]
  }
}
```

**Key Fields**:
- `measures[]`: Baseline demographics (age, sex, race, etc.)

#### outcomeMeasuresModule

```json
{
  "outcomeMeasuresModule": {
    "outcomeMeasures": [
      {
        "type": "PRIMARY",
        "title": "Mean Percentage of Days With Worsening Asthma",
        "timeFrame": "8 weeks",
        "groups": [...],
        "denoms": [...],
        "classes": [
          {
            "categories": [
              {
                "measurements": [
                  {"groupId": "OG000", "value": "20.5", "spread": "21.5"},
                  {"groupId": "OG001", "value": "17.9", "spread": "19.8"}
                ]
              }
            ]
          }
        ],
        "analyses": [
          {
            "groupIds": ["OG000", "OG001"],
            "pValue": "0.027",
            "statisticalMethod": "t-test, 2 sided"
          }
        ]
      }
    ]
  }
}
```

**Key Fields**:
- `outcomeMeasures[]`: Results for primary/secondary outcomes
- `analyses[]`: Statistical comparisons (p-values, confidence intervals)

#### adverseEventsModule

```json
{
  "adverseEventsModule": {
    "frequencyThreshold": "5",
    "timeFrame": "8 weeks",
    "groups": [...],
    "seriousEvents": [...],
    "otherEvents": [
      {
        "term": "Headache",
        "organSystem": "Nervous system disorders",
        "stats": [
          {"groupId": "EG000", "numEvents": "45", "numAffected": "42", "numAtRisk": "566"},
          {"groupId": "EG001", "numEvents": "38", "numAffected": "35", "numAtRisk": "566"}
        ]
      }
    ]
  }
}
```

**Key Fields**:
- `seriousEvents[]`: Serious adverse events (SAEs)
- `otherEvents[]`: Common adverse events (AEs)

---

### 9.3 derivedSection (Required)

Contains searchable metadata derived from trial data.

#### miscInfoModule

```json
{
  "miscInfoModule": {
    "versionHolder": "2024-12-27",
    "removedCountries": [],
    "submissionTracking": {...}
  }
}
```

#### conditionBrowseModule

```json
{
  "conditionBrowseModule": {
    "meshes": [
      {
        "id": "D001249",
        "term": "Asthma"
      }
    ],
    "ancestors": [
      {
        "id": "D012140",
        "term": "Respiratory Tract Diseases"
      },
      {
        "id": "D008171",
        "term": "Lung Diseases"
      }
    ],
    "browseLeaves": [...],
    "browseBranches": [...]
  }
}
```

**Key Fields**:
- `meshes[]`: MeSH terms for conditions (id + term)
- `ancestors[]`: MeSH hierarchy (broader terms)

**Cross-Reference Potential**: MeSH IDs can link to PubMed, OMIM, other databases.

#### interventionBrowseModule

```json
{
  "interventionBrowseModule": {
    "meshes": [
      {
        "id": "C093875",
        "term": "montelukast"
      }
    ]
  }
}
```

**Key Fields**:
- `meshes[]`: MeSH terms for interventions (drugs, devices, etc.)

**Cross-Reference Potential**: Drug MeSH IDs can link to DrugBank, ChEMBL, PubChem.

---

### 9.4 annotationSection (Optional)

Contains annotations for unposted data or protocol violations.

```json
{
  "annotationSection": {
    "annotationModule": {
      "unpostedAnnotation": {
        "unpostedResponsibleParty": "Sponsor did not provide...",
        "unpostedEvents": [...]
      }
    }
  }
}
```

---

### 9.5 documentSection (Optional)

Contains large document attachments (protocols, consent forms, statistical analysis plans).

```json
{
  "documentSection": {
    "largeDocumentModule": {
      "largeDocs": [
        {
          "typeAbbrev": "Prot",
          "hasProtocol": true,
          "hasSap": false,
          "hasIcf": false,
          "label": "Study Protocol",
          "date": "2021-03-15",
          "uploadDate": "2021-04-01T12:34:56.789",
          "filename": "Prot_001.pdf",
          "size": 1024000
        }
      ]
    }
  }
}
```

**Key Fields**:
- `largeDocs[].typeAbbrev`: Document type (Prot=Protocol, SAP=Statistical Analysis Plan, ICF=Informed Consent)
- `largeDocs[].filename`: File name (string)

**Note**: Documents are **not directly downloadable via API**. Users must access via web interface.

---

## 10. Implementation Recommendations

### 10.1 Fuzzy Search Tool (`search_trials`)

**Tool Signature**:
```python
@mcp.tool()
async def search_trials(
    query: str | None = None,
    condition: str | None = None,
    intervention: str | None = None,
    location: str | None = None,
    status: str | None = None,
    phase: str | None = None,
    page_size: int = 50,
    page_token: str | None = None
) -> dict:
```

**API Call**:
```python
params = {
    "format": "json",
    "pageSize": page_size,
    "countTotal": "true"
}

if query:
    params["query.term"] = query
if condition:
    params["query.cond"] = condition
if intervention:
    params["query.intr"] = intervention
if location:
    params["query.locn"] = location
if status:
    params["filter.overallStatus"] = status.upper().replace(" ", "_")
if phase:
    params["filter.advanced"] = f"AREA[Phase]{phase}"
if page_token:
    params["pageToken"] = page_token

response = await self.client.get(f"{self.base_url}/studies", params=params)
```

**Response Mapping**:
```python
# Extract search candidates
candidates = [
    TrialSearchCandidate(
        nct_id=f"NCT:{study['protocolSection']['identificationModule']['nctId'][3:]}",
        title=study['protocolSection']['identificationModule']['briefTitle'],
        brief_summary=study['protocolSection']['descriptionModule'].get('briefSummary'),
        phase=study['protocolSection']['designModule']['phases'][0] if 'phases' in study['protocolSection']['designModule'] else None,
        status=study['protocolSection']['statusModule']['overallStatus'],
        conditions=study['protocolSection'].get('conditionsModule', {}).get('conditions', []),
        interventions=[
            i['name'] for i in study['protocolSection'].get('armsInterventionsModule', {}).get('interventions', [])
        ]
    )
    for study in response['studies']
]

return PaginationEnvelope(
    items=candidates,
    pagination={
        "cursor": response.get("nextPageToken"),
        "total_count": response.get("totalCount"),
        "page_size": page_size
    }
)
```

### 10.2 Strict Lookup Tool (`get_trial`)

**Tool Signature**:
```python
@mcp.tool()
async def get_trial(nct_id: str) -> dict:
```

**Validation**:
```python
# Validate CURIE format
nct_id = validate_nct_id(nct_id)  # Returns NCT:########

# Extract raw NCT ID for API call
raw_nct = f"NCT{nct_id.split(':')[1]}"  # NCT:00461032 -> NCT00461032
```

**API Call**:
```python
response = await self.client.get(f"{self.base_url}/studies/{raw_nct}?format=json")
```

**Response Mapping**:
```python
protocol = response['protocolSection']
derived = response.get('derivedSection', {})

trial = Trial(
    nct_id=nct_id,  # NCT:00461032
    title=protocol['identificationModule']['briefTitle'],
    brief_summary=protocol['descriptionModule'].get('briefSummary'),
    detailed_description=protocol['descriptionModule'].get('detailedDescription'),
    protocol=protocol['identificationModule'].get('officialTitle'),

    # Eligibility
    eligibility_criteria=protocol['eligibilityModule'].get('eligibilityCriteria'),
    minimum_age=protocol['eligibilityModule'].get('minimumAge'),
    maximum_age=protocol['eligibilityModule'].get('maximumAge'),
    sex=protocol['eligibilityModule'].get('sex'),

    # Outcomes
    primary_outcomes=[
        o['measure'] for o in protocol['outcomesModule'].get('primaryOutcomes', [])
    ],
    secondary_outcomes=[
        o['measure'] for o in protocol['outcomesModule'].get('secondaryOutcomes', [])
    ],

    # Administrative
    sponsors={
        "lead": protocol['sponsorCollaboratorsModule']['leadSponsor']['name'],
        "collaborators": [
            c['name'] for c in protocol['sponsorCollaboratorsModule'].get('collaborators', [])
        ]
    },
    phase=protocol['designModule'].get('phases', [None])[0],
    status=protocol['statusModule']['overallStatus'],
    enrollment=protocol['designModule']['enrollmentInfo'].get('count'),
    study_type=protocol['designModule']['studyType'],

    # Dates
    start_date=protocol['statusModule']['startDateStruct'].get('date'),
    completion_date=protocol['statusModule']['completionDateStruct'].get('date'),
    last_update_date=protocol['statusModule']['lastUpdatePostDateStruct'].get('date'),

    # Cross-references
    cross_references=extract_cross_references(response)
)

return trial
```

### 10.3 Cross-Reference Extraction

```python
def extract_cross_references(study_data: dict) -> dict:
    """Extract cross-references from trial data."""
    refs = {}

    # Always include registry link
    nct_id = study_data['protocolSection']['identificationModule']['nctId']
    refs['clinicaltrials_gov'] = f"https://clinicaltrials.gov/study/{nct_id}"

    # PubMed references
    if 'referencesModule' in study_data.get('protocolSection', {}):
        pmids = [
            ref['pmid']
            for ref in study_data['protocolSection']['referencesModule'].get('references', [])
            if 'pmid' in ref
        ]
        if pmids:
            refs['pubmed'] = pmids  # List of PMIDs

    # Secondary IDs (EudraCT, ISRCTN, etc.)
    secondary_ids = study_data['protocolSection']['identificationModule'].get('secondaryIdInfos', [])
    for sec_id in secondary_ids:
        id_type = sec_id.get('type', '').lower()
        if 'eudract' in id_type:
            refs['eudract'] = sec_id['id']
        elif 'isrctn' in id_type:
            refs['isrctn'] = sec_id['id']

    # MeSH terms (potential links to other databases)
    if 'derivedSection' in study_data:
        # Condition MeSH
        if 'conditionBrowseModule' in study_data['derivedSection']:
            condition_meshes = [
                m['id'] for m in study_data['derivedSection']['conditionBrowseModule'].get('meshes', [])
            ]
            if condition_meshes:
                refs['mesh_conditions'] = condition_meshes

        # Intervention MeSH
        if 'interventionBrowseModule' in study_data['derivedSection']:
            intervention_meshes = [
                m['id'] for m in study_data['derivedSection']['interventionBrowseModule'].get('meshes', [])
            ]
            if intervention_meshes:
                refs['mesh_interventions'] = intervention_meshes

    return refs
```

### 10.4 Location Tool (`get_trial_locations`)

**Tool Signature**:
```python
@mcp.tool()
async def get_trial_locations(nct_id: str) -> list[dict]:
```

**API Call**:
```python
# Same as get_trial, but extract only locations
response = await self.client.get(f"{self.base_url}/studies/{raw_nct}?format=json&fields=NCTId,ContactsLocationsModule")
```

**Response Mapping**:
```python
locations_module = response['protocolSection'].get('contactsLocationsModule', {})
locations = locations_module.get('locations', [])

return [
    TrialLocation(
        facility_name=loc['facility'],
        city=loc.get('city'),
        state=loc.get('state'),
        country=loc.get('country'),
        zip=loc.get('zip'),
        # Note: Contact details may not be present in API response
        # Check metadata endpoint for complete schema
        contact_name=loc.get('contact', {}).get('name'),  # Verify structure
        contact_phone=loc.get('contact', {}).get('phone'),
        contact_email=loc.get('contact', {}).get('email'),
        recruitment_status=loc.get('status')  # Verify field name
    )
    for loc in locations
]
```

**Important**: The example trial (NCT00461032) does **not** include contact details in `contactsLocationsModule`. Verify the complete schema using `/studies/metadata` endpoint.

---

## 11. Error Handling

### HTTP Status Codes

| Code | Meaning | MCP Error Code | Recovery Hint |
|------|---------|----------------|---------------|
| 200 | Success | N/A | N/A |
| 400 | Bad Request | `INVALID_INPUT` | Check parameter format |
| 404 | Not Found | `ENTITY_NOT_FOUND` | Verify NCT ID format |
| 429 | Too Many Requests | `RATE_LIMITED` | Retry with exponential backoff |
| 500 | Server Error | `UPSTREAM_ERROR` | Retry later or check API status |
| 503 | Service Unavailable | `UPSTREAM_ERROR` | API maintenance, retry later |

### Error Envelope Mapping

```python
async def _handle_api_error(self, response: httpx.Response, nct_id: str | None = None) -> dict:
    """Map HTTP errors to ErrorEnvelope."""

    if response.status_code == 404:
        return {
            "success": False,
            "error": {
                "code": "ENTITY_NOT_FOUND",
                "message": f"Trial {nct_id} not found in ClinicalTrials.gov registry",
                "recovery_hint": "Verify NCT ID format (NCT:########). Try search_trials to find valid NCT IDs.",
                "invalid_input": nct_id
            }
        }

    elif response.status_code == 429:
        return {
            "success": False,
            "error": {
                "code": "RATE_LIMITED",
                "message": "ClinicalTrials.gov API rate limit exceeded (>50 req/min)",
                "recovery_hint": "Retry after 60 seconds with exponential backoff. Consider reducing request frequency.",
                "invalid_input": None
            }
        }

    elif response.status_code >= 500:
        return {
            "success": False,
            "error": {
                "code": "UPSTREAM_ERROR",
                "message": f"ClinicalTrials.gov API error: {response.status_code} {response.reason_phrase}",
                "recovery_hint": "Retry request later. Check API status at https://clinicaltrials.gov/data-api/api",
                "invalid_input": None
            }
        }

    else:
        return {
            "success": False,
            "error": {
                "code": "UPSTREAM_ERROR",
                "message": f"Unexpected API error: {response.status_code}",
                "recovery_hint": "Contact system administrator if problem persists.",
                "invalid_input": None
            }
        }
```

---

## 12. Performance Optimization

### 12.1 Field Selection

Use `fields` parameter to reduce payload size:

```python
# Minimal search response (NCT ID + title only)
params["fields"] = "NCTId,BriefTitle,OverallStatus"

# Full trial details (all protocol fields)
# No fields parameter = return all data
```

**Token Budget**:
- Full trial: ~5,000-10,000 tokens (with results section)
- Search candidate: ~200-500 tokens
- Minimal fields: ~50-100 tokens

### 12.2 Caching Strategy

**Recommendation**: Implement in-memory cache for frequently accessed trials.

```python
from functools import lru_cache
import hashlib

@lru_cache(maxsize=100)
async def _cached_get_trial(nct_id: str) -> dict:
    """Cache trial data for 5 minutes."""
    return await self._get_trial_uncached(nct_id)
```

**Rationale**: Clinical trial data updates infrequently (weekly/monthly). Caching reduces API load and improves response time.

### 12.3 Batch Operations

**Not Supported**: ClinicalTrials.gov API v2 does **not** support batch trial lookups in a single request.

**Workaround**: Use `filter.ids` for multiple NCT IDs:

```python
# Search for specific trials
params["filter.ids"] = "NCT00461032,NCT04123456,NCT05678901"
```

**Limitation**: Still returns paginated results (not true batch).

---

## 13. Testing Strategy

### 13.1 Integration Test Trials

Use these well-known trials for integration tests:

| NCT ID | Title | Features | Use For |
|--------|-------|----------|---------|
| NCT00461032 | Montelukast Asthma Study | Phase 3, Completed, Results posted | Complete workflow |
| NCT04280705 | COVID-19 Vaccine Trial | Phase 3, Large enrollment | Search relevance |
| NCT05436431 | Breast Cancer Trial | Active, Multi-location | Location data |
| NCT99999999 | (Invalid) | N/A | 404 error handling |

### 13.2 Test Scenarios

**User Story 1 (Fuzzy Search)**:
1. Simple search: `search_trials(query="diabetes")`
2. Multi-filter: `search_trials(condition="asthma", phase="Phase 3", status="COMPLETED")`
3. Location search: `search_trials(condition="cancer", location="Boston")`
4. Pagination: Iterate through all pages for large result set

**User Story 2 (Strict Lookup)**:
1. Valid NCT ID: `get_trial("NCT:00461032")`
2. Invalid format: `get_trial("breast cancer")` → UNRESOLVED_ENTITY
3. Non-existent: `get_trial("NCT:99999999")` → ENTITY_NOT_FOUND
4. Missing fields: Verify null handling (omit, not null)

**User Story 3 (Locations)**:
1. Trial with locations: `get_trial_locations("NCT:05436431")`
2. Trial without locations: `get_trial_locations("NCT:00461032")` → empty list
3. Invalid NCT ID: `get_trial_locations("invalid")` → UNRESOLVED_ENTITY

**User Story 4 (Error Recovery)**:
1. UNRESOLVED_ENTITY: Pass query to get_trial → hint to use search_trials
2. RATE_LIMITED: Trigger 429 response → hint to retry with backoff
3. UPSTREAM_ERROR: Mock 503 response → hint to retry later

---

## 14. Key Differences from Other Life Sciences APIs

### Compared to UniProt/HGNC/ChEMBL

| Aspect | ClinicalTrials.gov | Gene/Protein APIs |
|--------|-------------------|-------------------|
| **Cross-references** | Limited (PubMed, registry IDs) | Extensive (22+ databases) |
| **Search complexity** | High (Essie syntax, multiple filters) | Medium (simple query strings) |
| **Data structure** | Deep nesting (5 sections, many modules) | Flat or shallow nesting |
| **Pagination** | Cursor-based (opaque tokens) | Cursor-based (transparent) |
| **Rate limits** | 50 req/min | 10 req/sec (UniProt) |
| **Authentication** | None | None |
| **Data volatility** | Medium (weekly updates) | Low (infrequent updates) |
| **Token budget** | High (5K-10K tokens/trial) | Medium (300-1K tokens/entity) |

### Unique Challenges

1. **Deep nesting**: Requires careful navigation of JSON structure (e.g., `protocolSection.identificationModule.nctId`)
2. **Optional sections**: `resultsSection` only present when results posted
3. **Limited cross-refs**: Cannot directly link to DrugBank/ChEMBL/UniProt (must query separately)
4. **Complex search**: Essie syntax is powerful but complex (defer to v2.0)
5. **Large payloads**: Full trials with results can exceed 10K tokens

---

## 15. References

### Official Documentation

- **API Home**: https://clinicaltrials.gov/data-api/api
- **Migration Guide**: https://clinicaltrials.gov/data-api/about-api/api-migration
- **OpenAPI Spec**: https://github.com/everygene/clinicaltrials-python/blob/main/openapi-spec.yaml
- **NLM Technical Bulletin**: https://www.nlm.nih.gov/pubs/techbull/ma24/ma24_clinicaltrials_api.html

### Third-Party Implementations

- **Python Examples**: https://github.com/TLDWTutorials/ClinicaltrialsAPIPython
- **MCP Server (cyanheads)**: https://github.com/cyanheads/clinicaltrialsgov-mcp-server
- **BioMCP Reference**: https://biomcp.org/backend-services-reference/04-clinicaltrials-gov/

### Related Literature

- **Linking ClinicalTrials.gov and PubMed**: https://pmc.ncbi.nlm.nih.gov/articles/PMC3706420/
- **Precision of CT.gov-PubMed Links**: https://pmc.ncbi.nlm.nih.gov/articles/PMC3540528/

---

## 16. Recommendations for Implementation

### Phase 1: Core Tools (v0.1.0)

✅ **In Scope**:
- `search_trials`: Fuzzy search with query, condition, intervention, location, status, phase filters
- `get_trial`: Strict lookup with complete trial data
- `get_trial_locations`: Geographic facility data
- Error envelopes with recovery hints
- Rate limiting (1 req/sec)
- Pagination support

❌ **Out of Scope**:
- Advanced Essie syntax (defer to v2.0)
- Batch operations (API limitation)
- Historical data versioning
- Document downloads
- Custom relevance ranking

### Phase 2: Future Enhancements (v2.0)

- Advanced search with Essie syntax
- Trial comparison tools
- Trend analysis across trials
- Enhanced cross-reference resolution (query DrugBank/ChEMBL based on intervention names)
- Caching layer for frequently accessed trials

### Critical Implementation Notes

1. **CURIE Normalization**: Always normalize NCT IDs to `NCT:########` format internally
2. **Null Handling**: Omit missing fields per ADR-001 (never return `null`)
3. **Rate Limiting**: Enforce conservative 1 req/sec to prevent throttling
4. **Error Recovery**: Provide actionable hints for all error codes
5. **Field Selection**: Use `fields` parameter for token optimization in batch workflows
6. **Pagination**: Store `nextPageToken` for cursor-based navigation
7. **Cross-References**: Extract PubMed links and MeSH IDs for integration with other APIs
8. **Testing**: Use NCT00461032 as primary integration test trial (complete data)

---

## Appendix A: Sample API Responses

### A.1 Search Response (`/studies`)

```json
{
  "studies": [
    {
      "protocolSection": {
        "identificationModule": {
          "nctId": "NCT00461032",
          "briefTitle": "Montelukast Back to School Asthma Study"
        },
        "statusModule": {
          "overallStatus": "COMPLETED"
        },
        "designModule": {
          "phases": ["PHASE3"]
        },
        "conditionsModule": {
          "conditions": ["Asthma"]
        }
      }
    }
  ],
  "nextPageToken": "eyJraW5kIjoiQ2xpbmljYWxUcmlhbHMiLCJwYWdlIjoxfQ==",
  "totalCount": 1543
}
```

### A.2 Single Trial Response (`/studies/NCT00461032`)

See Section 9 for complete structure.

---

**End of Research Document**
