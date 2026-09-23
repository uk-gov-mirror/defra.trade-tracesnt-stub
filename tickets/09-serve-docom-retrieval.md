---
project: CDMS
type: Story
summary: Serve single DOCOM retrieval from the TRACES NT simulator
parent: CDMS-1620
---

## Overview

Implement `getDocomCertificate` in the simulator, backed by the same state store and control API as the other certificate types, so `GET /certificates/docoms/{id}` can be tested end to end.

DOCOM carries follow-up information that the other certificate types do not, and a DOCOM can hold several follow-ups. That was the substance of https://eaflood.atlassian.net/browse/CDMS-1436, so the simulator must be able to serve a DOCOM with more than one follow-up.

**Note:** This ticket is optional and low priority. Only CHED is required for the epic to deliver value, so this should not be started before the CHED path works end to end.

**Note:** The contracts for this port are already generated, so this ticket is implementation work only.

**Note:** DOCOM search is out of scope. The gateway does not call `findDocomCertificate`, and multiple-DOCOM search is being considered separately in https://eaflood.atlassian.net/browse/CDMS-1588. PDF retrieval is likewise out of scope, since `getDocomPdfCertificate` is generated but never called.

## User Story

**As a** tester writing Trade Gateway journey tests

**I want** the simulator to serve DOCOM certificates I control, including their follow-ups

**So that** I can test DOCOM retrieval deterministically without EU acceptance access

## Acceptance Criteria

**AC1 – Retrieve a single DOCOM**\
**Given** a DOCOM created through the control API\
**When** the gateway calls `getDocomCertificate` with its ID\
**Then** the stored certificate is returned in the TRACES response schema\
**And** `GET /certificates/docoms/{id}` returns the mapped JSON model

**AC2 – Multiple follow-ups**\
**Given** a DOCOM holds more than one follow-up\
**When** it is retrieved\
**Then** every follow-up is present in the response\
**And** the follow-ups appear in the mapped JSON model as a top-level section

**AC3 – Unknown DOCOM**\
**Given** no DOCOM with the requested ID exists\
**When** the gateway calls `getDocomCertificate`\
**Then** the simulator responds as TRACES does for an unknown certificate\
**And** the endpoint returns the agreed not-found response using `EuDocomCertificateNotFoundException`

**AC4 – TRACES errors are handled**\
**Given** the simulator is configured to fail the request\
**When** the gateway calls `getDocomCertificate`\
**Then** the gateway returns the agreed descriptive error response\
**And** a permission-denied fault uses `EuDocomCertificatePermissionDeniedException`

## Testing Comments

- Run `tests/Api.Tests/Endpoints/DocomEndpointsTests.cs` against the simulator and confirm the Verify snapshots still match
- Seed from `tests/Api.Tests/Samples/DOCOM/GetDocomCertificateResponse.xml`, extended to cover the multi-follow-up case
- Some DOCOM content is returned by TRACES in a formatted or structured form and must be retained rather than normalised by the simulator

## Technical Notes

- The gateway's call is in `src/TracesNT/Services/DocomCertificateService.cs`
- The DOCOM JSON model is defined by `schemas/profiles/imports/eu/defra-unvtd-profile-docom-v1.schema.json` in `DEFRA/trade-imports-schemas`

## Questions & Actions

1.
