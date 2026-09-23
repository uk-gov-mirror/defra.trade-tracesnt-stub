---
project: CDMS
type: Story
summary: Serve single INTRA retrieval from the TRACES NT simulator
parent: CDMS-1620
---

## Overview

Implement `getEuIntraCertificate` in the simulator, backed by the same state store and control API as CHED, so `GET /certificates/intras/{id}` can be tested end to end.

**Note:** This ticket is optional and low priority. Only CHED is required for the epic to deliver value, so this should not be started before the CHED path works end to end.

**Note:** The contracts for this port are already generated, so this ticket is implementation work only.

**Note:** INTRA search is not included in this ticket and is delivered separately, matching the split already made for CHED.

## User Story

**As a** tester writing Trade Gateway journey tests

**I want** the simulator to serve INTRA certificates I control

**So that** I can test INTRA retrieval deterministically without EU acceptance access

## Acceptance Criteria

**AC1 – Retrieve a single INTRA**\
**Given** an INTRA certificate created through the control API\
**When** the gateway calls `getEuIntraCertificate` with its ID\
**Then** the stored certificate is returned in the TRACES response schema\
**And** `GET /certificates/intras/{id}` returns the mapped JSON model

**AC2 – Unknown INTRA**\
**Given** no INTRA certificate with the requested ID exists\
**When** the gateway calls `getEuIntraCertificate`\
**Then** the simulator responds as TRACES does for an unknown certificate\
**And** `GET /certificates/intras/{id}` returns the agreed not-found response

**AC3 – TRACES errors are handled**\
**Given** the simulator is configured to fail the request\
**When** the gateway calls `getEuIntraCertificate`\
**Then** the gateway returns the agreed descriptive error response\
**And** a permission-denied fault surfaces as 403 and a communication failure as 502

## Testing Comments

- Run the single-certificate cases in `tests/Api.Tests/Endpoints/IntraEndpointsTests.cs` against the simulator and confirm the Verify snapshots still match
- Seed from `tests/Api.Tests/Samples/INTRA/GetEuIntraCertificateResponse.xml`

## Technical Notes

- The gateway's call is in `src/TracesNT/Services/EuIntraCertificateService.cs`
- Coded values in the response carry list identifiers that the gateway's mappers turn into a full `https://traces-codelists.ec.europa.eu/{listId}` URI, so seed data must include them

## Questions & Actions

1.
