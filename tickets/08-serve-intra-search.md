---
project: CDMS
type: Story
summary: Serve INTRA search from the TRACES NT simulator
parent: CDMS-1620
---

## Overview

Implement `findEuIntraCertificate` in the simulator so `GET /certificates/intras` can be tested end to end, applying the same paging and date-range behaviour delivered for CHED search.

INTRA paging parameters are optional at the gateway's REST layer following https://eaflood.atlassian.net/browse/CDMS-1394, but the SOAP call always supplies values, so the simulator must handle the defaults the gateway sends as well as explicit ones.

**Note:** This ticket is optional and low priority. Only CHED is required for the epic to deliver value, so this should not be started before the CHED path works end to end.

**Note:** The contracts for this port are already generated, so this ticket is implementation work only.

## User Story

**As a** tester writing Trade Gateway journey tests

**I want** the simulator to search INTRA certificates I control

**So that** I can test INTRA paging and date-range filtering deterministically

## Acceptance Criteria

**AC1 – Search filters on the update-date range**\
**Given** several INTRA certificates exist with different update timestamps\
**When** the gateway calls `findEuIntraCertificate` with an `UpdateDateTimeRange`\
**Then** only certificates updated within that range are returned

**AC2 – Search pages correctly**\
**Given** more matching certificates exist than the requested page size\
**When** the gateway calls `findEuIntraCertificate` with `offset` and `pageSize`\
**Then** the requested page is returned\
**And** paging through the full result set returns every match exactly once

**AC3 – Default paging parameters are handled**\
**Given** a request to `GET /certificates/intras` omits `offset` and `pageSize`\
**When** the gateway calls `findEuIntraCertificate` with its default values\
**Then** the simulator returns a valid page rather than an error

**AC4 – Empty result**\
**Given** no INTRA certificate matches the search criteria\
**When** the gateway calls `findEuIntraCertificate`\
**Then** an empty result is returned\
**And** `GET /certificates/intras` returns an empty items array rather than an error

## Testing Comments

- Reuse the boundary cases established for CHED search: offset beyond the result set, page size larger than the result set, exact multiples, and a range matching nothing
- `tests/Api.Tests/Samples/INTRA/FindEuIntraCertificateResponse.xml` shows the response shape to reproduce
- The empty-result case is a regression guard for https://eaflood.atlassian.net/browse/CDMS-1439 and should be asserted explicitly

## Technical Notes

- The gateway's call is in `src/TracesNT/Services/EuIntraCertificateService.cs`
- ADR-0003: Collection Query Endpoints describes the paging semantics the gateway layers on top of the TRACES find operation, and is the reference for what the simulator's results have to support

## Questions & Actions

1.
