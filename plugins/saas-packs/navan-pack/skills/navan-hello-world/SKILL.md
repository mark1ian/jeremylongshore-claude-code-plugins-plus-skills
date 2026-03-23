---
name: navan-hello-world
description: |
  Make your first Navan API call to retrieve trip and user data.
  Use when verifying a new Navan integration works end-to-end after auth setup.
  Trigger with "navan hello world", "navan example", "test navan api", "first navan call".
allowed-tools: Read, Write, Edit, Bash(npm:*), Grep
version: 1.0.0
license: MIT
author: Jeremy Longshore <jeremy@intentsolutions.io>
tags: [saas, navan, travel]
compatible-with: claude-code
---

# Navan Hello World

## Overview

Execute a first API call against the Navan REST API to retrieve trip data. All examples use raw REST calls — Navan has **no public SDK**.

**Purpose:** Confirm end-to-end integration by retrieving real trip data and parsing `uuid` primary keys.

## Prerequisites

- Completed `navan-install-auth` with working OAuth 2.0 credentials
- `.env` file with `NAVAN_CLIENT_ID`, `NAVAN_CLIENT_SECRET`, and `NAVAN_BASE_URL`
- Node.js 18+ (for TypeScript) or Python 3.8+ (for Python)
- At least one trip or user in your Navan organization

## Instructions

### Step 1: Acquire a Bearer Token

Reuse the token exchange from `navan-install-auth`:

```typescript
import 'dotenv/config';

async function getNavanToken(): Promise<string> {
  const response = await fetch(`${process.env.NAVAN_BASE_URL}/authenticate`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      client_id: process.env.NAVAN_CLIENT_ID,
      client_secret: process.env.NAVAN_CLIENT_SECRET,
      grant_type: 'client_credentials',
    }),
  });
  if (!response.ok) throw new Error(`Auth failed: ${response.status}`);
  const data = await response.json();
  return data.access_token;
}
```

### Step 2: Retrieve User Trips (TypeScript)

Call `GET /get_user_trips` to fetch the authenticated user's trip history:

```typescript
interface NavanTrip {
  uuid: string;           // Primary key for all booking records
  traveler_name: string;
  origin: string;
  destination: string;
  departure_date: string;
  return_date: string;
  booking_status: string;
  booking_type: string;   // "flight", "hotel", "car"
}

async function getUserTrips(token: string): Promise<NavanTrip[]> {
  const response = await fetch(`${process.env.NAVAN_BASE_URL}/get_user_trips`, {
    headers: {
      Authorization: `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
  });

  if (!response.ok) {
    throw new Error(`GET /get_user_trips failed: ${response.status} ${response.statusText}`);
  }

  const data = await response.json();
  return data.trips ?? data.results ?? [];
}

// Execute
const token = await getNavanToken();
const trips = await getUserTrips(token);
console.log(`Retrieved ${trips.length} trips:`);
trips.forEach((t) =>
  console.log(`  [${t.uuid}] ${t.origin} -> ${t.destination} (${t.booking_status})`)
);
```

### Step 3: Retrieve User Trips (Python)

```python
import os
import requests
from dotenv import load_dotenv

load_dotenv()

def get_navan_token() -> str:
    resp = requests.post(f"{os.environ['NAVAN_BASE_URL']}/authenticate", json={
        "client_id": os.environ["NAVAN_CLIENT_ID"],
        "client_secret": os.environ["NAVAN_CLIENT_SECRET"],
        "grant_type": "client_credentials",
    })
    resp.raise_for_status()
    return resp.json()["access_token"]

def get_user_trips(token: str) -> list[dict]:
    resp = requests.get(
        f"{os.environ['NAVAN_BASE_URL']}/get_user_trips",
        headers={"Authorization": f"Bearer {token}"},
    )
    resp.raise_for_status()
    data = resp.json()
    return data.get("trips", data.get("results", []))

token = get_navan_token()
trips = get_user_trips(token)
print(f"Retrieved {len(trips)} trips:")
for t in trips:
    print(f"  [{t['uuid']}] {t.get('origin')} -> {t.get('destination')}")
```

### Step 4: Explore Other Endpoints

Once trips work, try these additional endpoints with the same bearer token:

```typescript
// Admin-level: all trips across the organization
const adminTrips = await fetch(`${process.env.NAVAN_BASE_URL}/get_admin_trips`, {
  headers: { Authorization: `Bearer ${token}` },
});

// User directory
const users = await fetch(`${process.env.NAVAN_BASE_URL}/get_users`, {
  headers: { Authorization: `Bearer ${token}` },
});

// Invoice data (if enabled)
const invoices = await fetch(`${process.env.NAVAN_BASE_URL}/get_invoices_poc`, {
  headers: { Authorization: `Bearer ${token}` },
});
```

### Step 5: Understand the Response Shape

Navan API responses use `uuid` as the primary key for booking records. Key fields to expect:

| Field | Type | Description |
|-------|------|-------------|
| `uuid` | string | Unique booking identifier (primary key) |
| `traveler_name` | string | Full name of the traveler |
| `booking_type` | string | "flight", "hotel", or "car" |
| `booking_status` | string | Current status of the booking |
| `origin` / `destination` | string | Airport codes or city names |

## Output

Successful completion produces:
- A working API call retrieving real trip data from the Navan organization
- Parsed response structure with `uuid` primary key identification
- Tested access to available endpoints (user trips, admin trips, users, invoices)

## Error Handling

| Error | Code | Cause | Solution |
|-------|------|-------|----------|
| Unauthorized | 401 | Expired or invalid OAuth token | Re-run token exchange; check credentials |
| Forbidden | 403 | Insufficient permissions or wrong tier | Verify admin role; contact Navan support |
| Not found | 404 | Invalid endpoint path | Check spelling; use exact paths from this guide |
| Rate limited | 429 | Too many requests | Wait and retry with exponential backoff |
| Server error | 500 | Navan service issue | Retry after 30s; check Navan status |
| Maintenance | 503 | Scheduled or unscheduled downtime | Wait and retry; check for maintenance windows |

## Examples

**Quick curl test from terminal:**

```bash
# Get token and fetch trips in one pipeline
TOKEN=$(curl -s -X POST https://app.navan.com/authenticate \
  -H "Content-Type: application/json" \
  -d "{\"client_id\":\"$NAVAN_CLIENT_ID\",\"client_secret\":\"$NAVAN_CLIENT_SECRET\",\"grant_type\":\"client_credentials\"}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

curl -s https://app.navan.com/get_user_trips \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

## Resources

- [Navan Help Center](https://app.navan.com/app/helpcenter) — primary documentation hub
- [Booking Data Integration](https://app.navan.com/app/helpcenter/articles/travel/admin/other-integrations/booking-data-integration) — booking data export guide
- [Navan TMC API Docs](https://app.navan.com/app/helpcenter/articles/travel/admin/other-integrations/navan-tmc-api-integration-documentation) — API integration reference
- [Navan Security](https://navan.com/security) — compliance certifications (SOC 2, ISO 27001)

## Next Steps

Now that your first API call works, proceed to `navan-sdk-patterns` to build a typed wrapper class, or see `navan-local-dev-loop` for a structured development environment.
