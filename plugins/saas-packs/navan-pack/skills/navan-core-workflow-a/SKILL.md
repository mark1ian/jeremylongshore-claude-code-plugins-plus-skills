---
name: navan-core-workflow-a
description: |
  Manage the complete Navan travel booking lifecycle via REST API.
  Use when building travel dashboards, automating trip reporting, or syncing booking data to internal systems.
  Trigger with "navan travel workflow", "navan booking management", "navan trip retrieval".
allowed-tools: Read, Write, Edit, Bash(npm:*), Bash(curl:*), Grep
version: 1.0.0
license: MIT
author: Jeremy Longshore <jeremy@intentsolutions.io>
tags: [saas, navan, travel]
compatible-with: claude-code
---

# Navan — Travel Booking & Management

## Overview

This skill provides the complete travel booking workflow through the Navan REST API. Navan has no public SDK — all access is via raw HTTP calls using OAuth 2.0 bearer tokens. This skill covers trip retrieval for both user and admin scopes, itinerary PDF downloads, invoice access, and trip filtering by date range and status. Every booking is keyed by a UUID primary key that must be tracked for deduplication and updates.

## Prerequisites

- Navan account with API credentials (client_id + client_secret)
- Credentials created in Admin > Travel admin > Settings > Integrations > Navan API Credentials
- OAuth 2.0 token obtained via POST /authenticate (see `navan-install-auth`)
- Node.js 18+ with `node-fetch` or Python 3.8+ with `requests`
- Environment variables: `NAVAN_CLIENT_ID`, `NAVAN_CLIENT_SECRET`, `NAVAN_BASE_URL`

## Instructions

### Step 1: Authenticate and Obtain Bearer Token

```typescript
const tokenRes = await fetch(`${process.env.NAVAN_BASE_URL}/authenticate`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    client_id: process.env.NAVAN_CLIENT_ID,
    client_secret: process.env.NAVAN_CLIENT_SECRET,
  }),
});
const { access_token } = await tokenRes.json();
const headers = { Authorization: `Bearer ${access_token}` };
```

### Step 2: Retrieve User Trips

```typescript
// GET /get_user_trips — returns trips for the authenticated user
const tripsRes = await fetch(`${process.env.NAVAN_BASE_URL}/get_user_trips`, {
  headers,
});
const trips = await tripsRes.json();

trips.forEach((trip: any) => {
  console.log(`UUID: ${trip.uuid}`);
  console.log(`  Route: ${trip.origin} -> ${trip.destination}`);
  console.log(`  Status: ${trip.status}`);
  console.log(`  Dates: ${trip.start_date} to ${trip.end_date}`);
});
```

### Step 3: Retrieve Admin Trips (All Employees)

```typescript
// GET /get_admin_trips — returns trips across the organization
// Requires admin-level API credentials
const adminTripsRes = await fetch(
  `${process.env.NAVAN_BASE_URL}/get_admin_trips?start_date=2026-01-01&end_date=2026-03-31`,
  { headers }
);
const adminTrips = await adminTripsRes.json();
console.log(`Total org trips: ${adminTrips.length}`);
```

### Step 4: Download Itinerary PDFs

```typescript
// GET /get_itineraries_pdf — download formatted itinerary documents
const pdfRes = await fetch(
  `${process.env.NAVAN_BASE_URL}/get_itineraries_pdf?trip_uuid=${trip.uuid}`,
  { headers }
);
const pdfBuffer = await pdfRes.arrayBuffer();
const fs = await import('fs');
fs.writeFileSync(`itinerary-${trip.uuid}.pdf`, Buffer.from(pdfBuffer));
console.log(`Saved itinerary for trip ${trip.uuid}`);
```

### Step 5: Retrieve Invoices

```typescript
// GET /get_invoices_poc — pull invoices for finance reconciliation
const invoicesRes = await fetch(
  `${process.env.NAVAN_BASE_URL}/get_invoices_poc?start_date=2026-01-01&end_date=2026-03-31`,
  { headers }
);
const invoices = await invoicesRes.json();
invoices.forEach((inv: any) => {
  console.log(`Invoice: ${inv.invoice_id} | Amount: $${inv.total} | Date: ${inv.date}`);
});
```

### Step 6: Token Refresh

```typescript
// POST /reauthenticate — refresh an expired token
const refreshRes = await fetch(`${process.env.NAVAN_BASE_URL}/reauthenticate`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    client_id: process.env.NAVAN_CLIENT_ID,
    client_secret: process.env.NAVAN_CLIENT_SECRET,
    access_token: access_token,  // current (possibly expired) token
  }),
});
const refreshed = await refreshRes.json();
console.log('Token refreshed successfully');
```

## Output

Successful execution returns:
- Trip objects with UUID primary keys, origin/destination, dates, and status
- PDF itinerary documents saved to local filesystem
- Invoice records with amounts, dates, and reconciliation IDs

## Error Handling

| Error | HTTP Code | Cause | Solution |
|-------|-----------|-------|----------|
| Unauthorized | 401 | Expired or invalid bearer token | Re-authenticate via POST /authenticate |
| Forbidden | 403 | Insufficient API scope | Verify admin credentials for /get_admin_trips |
| Not Found | 404 | Invalid trip UUID | Confirm UUID exists via /get_user_trips |
| Rate Limited | 429 | Too many requests | Implement exponential backoff (start at 1s) |
| Server Error | 500 | Navan platform issue | Retry with backoff; check Navan status page |

## Examples

**Python — Retrieve trips with date filtering:**

```python
import requests
import os

base_url = os.environ['NAVAN_BASE_URL']

# Authenticate
auth = requests.post(f'{base_url}/authenticate', json={
    'client_id': os.environ['NAVAN_CLIENT_ID'],
    'client_secret': os.environ['NAVAN_CLIENT_SECRET'],
})
token = auth.json()['access_token']
headers = {'Authorization': f'Bearer {token}'}

# Get admin trips for Q1
trips = requests.get(
    f'{base_url}/get_admin_trips',
    params={'start_date': '2026-01-01', 'end_date': '2026-03-31'},
    headers=headers,
).json()

for trip in trips:
    print(f"{trip['uuid']}: {trip['origin']} -> {trip['destination']}")
```

## Resources

- [Navan Help Center](https://app.navan.com/app/helpcenter) — Official documentation and support articles
- [Booking Data Integration](https://app.navan.com/app/helpcenter/articles/travel/admin/other-integrations/booking-data-integration) — Booking data export configuration
- [Navan Integrations](https://navan.com/integrations) — Available third-party integrations

## Next Steps

After retrieving trip data, proceed to `navan-core-workflow-b` for expense management or `navan-data-handling` for bulk data extraction patterns.
