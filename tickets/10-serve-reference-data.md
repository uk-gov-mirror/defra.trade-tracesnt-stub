---
project: CDMS
type: Story
summary: Serve reference data from the TRACES NT simulator
parent: CDMS-1620
---

## Overview

Implement the four reference data operations the gateway calls: `getClassificationSections`, `getClassificationTree`, `getClassificationTreeNodeDetail` and `getMetadatas`.

Reference data is slow-moving and read-only, so unlike the certificate ports this is largely static content rather than live state. It should be seeded from responses captured from TRACES acceptance rather than synthesised, because hand-made reference data will not exercise the code-list mappers realistically, with a documented way to refresh it when the EU data changes.

**Note:** This ticket is optional and low priority. Only CHED is required for the epic to deliver value, so this should not be started before the CHED path works end to end.

**Note:** The contracts for this port are already generated, so this ticket is implementation work only.

**Note:** `getClassificationTrees`, `getClassificationTreeUpdates`, `getCertificateModel` and the laboratory test operations are generated but not called by the gateway, and are out of scope.

## User Story

**As a** tester writing Trade Gateway journey tests

**I want** the simulator to serve realistic TRACES reference data

**So that** reference data endpoints and code-list mapping can be tested without EU access

## Acceptance Criteria

**AC1 – All four operations are implemented**\
**Given** the simulator is running\
**When** the gateway calls each of the four reference data operations\
**Then** each returns data in the TRACES response schema\
**And** the corresponding `/reference-data/**` endpoints return their mapped JSON models

**AC2 – The data the gateway's tests need is present**\
**Given** the seeded reference data\
**When** the classification trees and metadata types used by Trade Gateway's tests are requested\
**Then** `INTRA_TRADE`, `CHEDA` and `ACCOMPANYING_DOCUMENT_TYPE` are present and complete as a minimum

**AC3 – Code list identifiers are preserved**\
**Given** a coded value is returned by the simulator\
**When** the gateway maps it\
**Then** the source data carries the list identifier the mapping needs\
**And** `CodedValue.urlId` resolves to a full `https://traces-codelists.ec.europa.eu/{listId}` URI

**AC4 – Unknown identifiers**\
**Given** an unknown tree or node identifier is requested\
**When** the operation is called\
**Then** the simulator responds as TRACES does\
**And** the gateway returns the agreed not-found response

**AC5 – Seed data can be refreshed**\
**Given** TRACES reference data changes\
**When** a developer follows the documented process\
**Then** the seed data can be re-captured from acceptance and updated

## Testing Comments

- Run `tests/Api.Tests/Endpoints/ReferenceDataEndpointsTests.cs` against the simulator and confirm the Verify snapshots still match
- Capture seed data from acceptance rather than writing it by hand, since the value of this story is realistic mapping coverage
- The existing samples under `tests/Api.Tests/Samples/REFERENCE_DATA/` show the response shapes and the trees and metadata types already in use

## Technical Notes

- The gateway's calls are in `src/TracesNT/Services/ReferenceDataService.cs`
- Mapping behaviour is documented in `docs/mappings/reference-data-mappings.md`

## Questions & Actions

1. Confirm which classification trees and metadata types the journey tests need beyond those the existing integration tests use
