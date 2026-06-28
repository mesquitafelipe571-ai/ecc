---
name: orcid-database
description: ORCID iD researcher profile lookup, works and publication retrieval, employment and affiliation data, funding records, and Public API workflows for reproducible researcher identity resolution.
metadata:
  origin: community
---

# ORCID Database

Use this skill when a task needs verified researcher identity data, publication
records, affiliations, or funding information from ORCID rather than general
web search.

## When to Use

- Resolving researcher identity from a name to a persistent ORCID iD.
- Fetching a researcher's works, publications, or contribution records.
- Retrieving employment, education, or affiliation history.
- Looking up funding grants associated with a researcher.
- Building reproducible author-disambiguation or co-authorship analysis.
- Integrating ORCID data into academic search, citation, or CV pipelines.

## ORCID iD Format

An ORCID iD is a 16-digit code grouped in fours, optionally prefixed with the
ORCID URI:

```
0000-0002-1825-0097
https://orcid.org/0000-0002-1825-0097
```

The last character may be `X` (check-digit). Always validate the format before
making API requests.

## API Overview

ORCID provides two API tiers:

| Tier | Base URL | Auth | Use case |
|------|----------|------|----------|
| Public API | `https://pub.orcid.org/v3.0` | Client credentials (read-only) | Read public records |
| Member API | `https://api.orcid.org/v3.0` | OAuth 2.0 per-user | Read trusted + write data |
| Search API | `https://pub.orcid.org/v3.0/search` | Client credentials | Find ORCIDs by query |

Use the **Public API** for all read-only research workflows. Use the **Member
API** only when writing data or reading `limited-access` fields, and only with
explicit user consent via OAuth.

Store credentials in environment variables, never in committed files.

```bash
export ORCID_CLIENT_ID="APP-XXXXXXXXXXXXXXXXX"
export ORCID_CLIENT_SECRET="..."
```

## Authentication — Client Credentials Token

Obtain a bearer token using the OAuth 2.0 client credentials flow:

```python
import os
import requests

def get_orcid_token() -> str:
    resp = requests.post(
        "https://orcid.org/oauth/token",
        headers={"Accept": "application/json"},
        data={
            "client_id": os.environ["ORCID_CLIENT_ID"],
            "client_secret": os.environ["ORCID_CLIENT_SECRET"],
            "grant_type": "client_credentials",
            "scope": "/read-public",
        },
        timeout=30,
    )
    resp.raise_for_status()
    return resp.json()["access_token"]
```

Cache the token; it is valid for 20 years for public-read scope. Do not request
a new token on every API call.

## Fetching a Full Record

```python
import os
import requests

BASE = "https://pub.orcid.org/v3.0"

def get_record(orcid_id: str, token: str) -> dict:
    url = f"{BASE}/{orcid_id}/record"
    resp = requests.get(
        url,
        headers={
            "Accept": "application/json",
            "Authorization": f"Bearer {token}",
        },
        timeout=30,
    )
    resp.raise_for_status()
    return resp.json()
```

## Key Endpoints

| Endpoint | Returns |
|----------|---------|
| `/{orcid}/record` | Full public record |
| `/{orcid}/person` | Name, biography, keywords, external IDs |
| `/{orcid}/works` | Publication and contribution summaries |
| `/{orcid}/works/{put-code}` | Full work detail |
| `/{orcid}/employments` | Employment affiliations |
| `/{orcid}/educations` | Education affiliations |
| `/{orcid}/fundings` | Funding records |
| `/{orcid}/peer-reviews` | Peer-review activities |
| `/{orcid}/external-identifiers` | Linked IDs (Scopus, ResearcherID, etc.) |

## Searching for Researchers

Use the ORCID search API to find ORCIDs by name, affiliation, or identifier.
The search index uses Solr syntax.

```python
def search_orcid(query: str, token: str, rows: int = 10) -> list[dict]:
    resp = requests.get(
        f"{BASE}/search",
        headers={
            "Accept": "application/json",
            "Authorization": f"Bearer {token}",
        },
        params={"q": query, "rows": rows, "start": 0},
        timeout=30,
    )
    resp.raise_for_status()
    results = resp.json().get("result", [])
    return [r["orcid-identifier"] for r in results]

# Examples
# By name (use affiliation to disambiguate):
search_orcid('given-names:Maria+family-name:Silva+affiliation-org-name:USP', token)

# By external ID (e.g., Scopus Author ID):
search_orcid('external-id-reference:56023157900', token)

# By email (indexed only if researcher made it public):
search_orcid('email:researcher@institution.edu', token)
```

Useful Solr fields for ORCID search:

| Field | Description |
|-------|-------------|
| `given-names` | First / given name |
| `family-name` | Last / family name |
| `credit-name` | Preferred display name |
| `affiliation-org-name` | Current or past organization name |
| `ringgold-org-id` | Ringgold ID of affiliated organization |
| `grid-org-id` | GRID ID of affiliated organization |
| `ror-org-id` | ROR ID of affiliated organization |
| `external-id-type` | Type of linked identifier (e.g., `scopus-author-id`) |
| `external-id-reference` | Value of the linked identifier |
| `email` | Researcher email (public only) |
| `keyword` | Research keywords |
| `digital-object-ids` | DOIs of linked works |

## Extracting Works

```python
def get_works_summary(orcid_id: str, token: str) -> list[dict]:
    resp = requests.get(
        f"{BASE}/{orcid_id}/works",
        headers={
            "Accept": "application/json",
            "Authorization": f"Bearer {token}",
        },
        timeout=30,
    )
    resp.raise_for_status()
    groups = resp.json().get("group", [])
    works = []
    for group in groups:
        for summary in group.get("work-summary", []):
            works.append({
                "put_code": summary.get("put-code"),
                "title": summary.get("title", {}).get("title", {}).get("value"),
                "type": summary.get("type"),
                "year": summary.get("publication-date", {}).get("year", {}).get("value"),
                "doi": next(
                    (
                        eid.get("external-id-value")
                        for eid in summary.get("external-ids", {}).get("external-id", [])
                        if eid.get("external-id-type") == "doi"
                    ),
                    None,
                ),
                "source": summary.get("source", {}).get("source-name", {}).get("value"),
            })
    return works
```

## Bulk Fetch (Multiple ORCIDs)

ORCID does not provide a native bulk endpoint. For lists of ORCIDs, fetch
sequentially with a small delay to respect rate limits (~24 requests/second
for the Public API):

```python
import time

def bulk_fetch(orcid_ids: list[str], token: str, delay: float = 0.05) -> list[dict]:
    records = []
    for orcid_id in orcid_ids:
        try:
            records.append(get_record(orcid_id, token))
        except requests.HTTPError as exc:
            records.append({"orcid": orcid_id, "error": str(exc)})
        time.sleep(delay)
    return records
```

## External ID Types

Researchers may link external identifiers to their ORCID record. Common types:

| Type | Description |
|------|-------------|
| `scopus-author-id` | Elsevier Scopus Author Identifier |
| `researcher-id` | Clarivate Web of Science ResearcherID |
| `loop-profile` | Frontiers Loop researcher profile |
| `researchgate` | ResearchGate profile ID |
| `google-scholar` | Google Scholar profile |
| `github` | GitHub username |
| `linkedin` | LinkedIn URL |
| `ssrn` | SSRN author ID |

Use external IDs to cross-link ORCID records with other databases (PubMed,
Scopus, WoS, etc.).

## Output Discipline

For every ORCID research pass, record:

- ORCID iD(s) queried
- API endpoint(s) called
- Date queried
- Token tier used (public / member)
- Whether data was public or limited-access
- Result counts per category (works, employments, fundings)

Example:

```markdown
| ORCID iD | Endpoint | Date queried | Token tier | Works | Employments |
| --- | --- | --- | --- | ---: | ---: |
| 0000-0002-1825-0097 | /record | 2026-06-28 | public | 47 | 3 |
| 0000-0001-5000-0007 | /works | 2026-06-28 | public | 12 | — |
```

## Review Checklist

- Is the ORCID iD format valid (16 digits, optional `X` check digit)?
- Are credentials loaded from environment variables, not hardcoded?
- Is the token cached and reused rather than re-requested per call?
- Are rate limits respected (sequential calls with delay for bulk)?
- Does the code call `raise_for_status()` before parsing responses?
- Is data labeled as public-record vs. limited-access?
- Are non-public fields accessed only with explicit user OAuth consent?
- Is the query log reproducible (exact endpoint, date, ORCID)?

## References

- [ORCID Public API documentation](https://info.orcid.org/documentation/api-tutorials/api-tutorial-get-and-authenticated-orcid-id/)
- [ORCID API v3.0 reference](https://github.com/ORCID/ORCID-Source/tree/main/orcid-model/src/main/resources/record_3.0)
- [ORCID Search API guide](https://info.orcid.org/documentation/api-tutorials/api-tutorial-searching-the-orcid-registry/)
- [ORCID developer tools](https://info.orcid.org/documentation/features/developer-tools/)
- [ORCID sandbox (testing)](https://sandbox.orcid.org/)
- Support: <support@orcid.org>
