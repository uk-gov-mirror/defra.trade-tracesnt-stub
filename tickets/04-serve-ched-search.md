---
project: CDMS
type: Story
summary: Serve CHED search from the TRACES NT simulator
parent: CDMS-1620
key: CDMS-1661
---

## Overview

Implement `findChedCertificate` in the simulator so `GET /certificates/cheds` can be tested end to end.

Search carries the contract most likely to be got wrong. The gateway sends `offset`, `pageSize` and an `UpdateDateTimeRange` of `From` and `To`, and the endpoint's behaviour at the boundaries — an offset past the end of the results, a page size larger than the result set, an empty match — is exactly what a real search against acceptance cannot be made to exercise reliably, because we do not control the data.

## User Story

**As a** tester writing Trade Gateway journey tests

**I want** the simulator to search CHED certificates I control

**So that** I can test paging and date-range filtering deterministically

## Acceptance Criteria

**AC1 – Search filters on the update-date range**\
**Given** several CHEDs exist with different update timestamps\
**When** the gateway calls `findChedCertificate` with an `UpdateDateTimeRange`\
**Then** only certificates updated within that range are returned

**AC2 – Search pages correctly**\
**Given** more matching CHEDs exist than the requested page size\
**When** the gateway calls `findChedCertificate` with `offset` and `pageSize`\
**Then** the requested page is returned\
**And** paging through the full result set returns every match exactly once with none duplicated or skipped

**AC3 – Paging boundaries behave**\
**Given** a set of matching CHEDs\
**When** the gateway requests an offset beyond the end of the results, or a page size larger than the result set\
**Then** the simulator returns an empty page or the full set respectively, without error

**AC4 – Empty result**\
**Given** no CHED matches the search criteria\
**When** the gateway calls `findChedCertificate`\
**Then** an empty result is returned\
**And** `GET /certificates/cheds` returns an empty items array rather than an error

**AC5 – Newly created certificates are searchable**\
**Given** a CHED is created through the control API with a known update timestamp\
**When** a search covering that timestamp is run\
**Then** the new certificate appears in the results

## Testing Comments

- Boundary cases deserve explicit tests: offset beyond the result set, page size larger than the result set, exact multiples of the page size, and a range matching nothing
- `tests/Api.Tests/Samples/CHED/FindChedCertificateResponse.xml` shows the response shape to reproduce
- Run the search cases in `tests/Api.Tests/Endpoints/ChedEndpointsTests.cs` against the simulator and confirm the Verify snapshots still match

## Technical Notes

- The SOAP action literal is `findChedCertificate`; the gateway's call is in `src/TracesNT/Services/ChedCertificateService.cs`
- `updatedFrom` and `updatedBefore` are required at the gateway's REST layer while `offset` and `pageSize` are optional, so the simulator must handle the defaults the gateway supplies
- ADR-0003: Collection Query Endpoints describes the paging semantics the gateway layers on top of the TRACES find operation, and is the reference for what the simulator's results have to support

## Questions & Actions

1. Confirm how real TRACES orders search results, since paging is only stable if the ordering is deterministic
