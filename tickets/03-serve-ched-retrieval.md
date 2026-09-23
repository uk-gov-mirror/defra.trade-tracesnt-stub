---
project: CDMS
type: Story
summary: Serve single CHED retrieval from the TRACES NT simulator
parent: CDMS-1620
key: CDMS-1623
---

## Overview

Implement `getChedCertificate` in the simulator, reading from the state the control API writes to. This is the first SOAP operation with real behaviour, and completes the first vertical slice of the epic: a CHED created through the control API is now retrievable through the gateway's own HTTP endpoint, `GET /certificates/cheds/{id}`.

**Note:** CHED search is not included in this ticket and is delivered separately, because the `findChedCertificate` paging and date-range contract is substantial enough to stand alone.

## User Story

**As a** tester writing Trade Gateway journey tests

**I want** the simulator to serve CHED certificates I control

**So that** I can test CHED retrieval deterministically without EU acceptance access

## Acceptance Criteria

**AC1 – Retrieve a single CHED**\
**Given** a CHED created through the control API\
**When** the gateway calls `getChedCertificate` with its ID\
**Then** the stored certificate is returned in the TRACES response schema\
**And** `GET /certificates/cheds/{id}` returns the mapped JSON model

**AC2 – Unknown CHED**\
**Given** no CHED with the requested ID exists in the simulator\
**When** the gateway calls `getChedCertificate`\
**Then** the simulator responds as TRACES does for an unknown certificate\
**And** `GET /certificates/cheds/{id}` returns the agreed not-found response

**AC4 – TRACES errors are handled**\
**Given** the gateway presents credentials the simulator rejects\
**When** it calls `getChedCertificate`\
**Then** the simulator returns a permission-denied fault\
**And** the gateway returns the agreed descriptive error response with a 403 status, without exposing unnecessary TRACES implementation detail

## Testing Comments

- Run `tests/Api.Tests/Endpoints/ChedEndpointsTests.cs` against the simulator in place of WireMock and confirm the Verify snapshots still match — this is the strongest available evidence that the simulator is wire-compatible

## Technical Notes

- The SOAP action literal is `getChedCertificate`; the gateway's call is in `src/TracesNT/Services/ChedCertificateService.cs`
- Every operation is invoked with the security header, `WebServiceClientId` and a language argument, so the simulator must tolerate the language argument even if responses do not vary by it

## Questions & Actions

1. None
