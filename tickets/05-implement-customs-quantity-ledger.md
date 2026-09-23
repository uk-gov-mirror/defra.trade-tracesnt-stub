---
project: CDMS
type: Story
summary: Implement the customs quantity ledger in the TRACES NT simulator
parent: CDMS-1620
key: CDMS-1662
---

## Overview

The customs CERTEX port is the most behavioural part of the simulator and has the subtlest contract. It exposes only two SOAP operations, but they back four Trade Gateway endpoints, and they are discriminated by field values rather than by operation name. Anyone implementing this from the WSDL alone will miss it. The discrimination is visible in `src/TracesNT/Services/CustomsChedService.cs`:

- `ProcessedChedRequest` with `QuantityManagementIndication` of `0` and an empty `CustomsDeclarationReferenceNumber` reads the quantity ledger
- `ProcessedChedRequest` with `QuantityManagementIndication` of `1`, an MRN and `CommodityDescriptionForChed` items reserves quantities
- `ChedClearanceRequest` with `GoodsClearanceInformationType.Item01` releases
- `ChedClearanceRequest` with `GoodsClearanceInformationType.Item02` deletes a reservation

Behind these the simulator must hold a genuine ledger. Reserve, release and delete have to affect each other and affect what a subsequent read returns, otherwise the endpoints delivered under https://eaflood.atlassian.net/browse/CDMS-1566, https://eaflood.atlassian.net/browse/CDMS-1568, https://eaflood.atlassian.net/browse/CDMS-1569 and https://eaflood.atlassian.net/browse/CDMS-1570 are not being tested as the stateful operations they are.

This story also closes a known gap. The fixtures under `tests/Api.Tests/Samples/CUSTOMS/` are synthetic, produced by round-tripping the generated types rather than captured from acceptance, as their README records and as ADR-0006 logs as an open question. Building the simulator is the natural moment to capture real responses.

**Note:** `chedInterventionRequest`, added under https://eaflood.atlassian.net/browse/CDMS-1567, is not included here and needs its own ticket once that work merges.

## User Story

**As a** tester working on customs quantity management

**I want** the simulator to hold a real quantity ledger

**So that** reserve, release and delete can be tested as the stateful operations they are

## Acceptance Criteria

**AC1 – Reading the ledger does not mutate it**\
**Given** a CHED with a quantity ledger\
**When** `ProcessedChedRequest` is called with `QuantityManagementIndication` of `0` and an empty `CustomsDeclarationReferenceNumber`\
**Then** the current ledger is returned and left unchanged\
**And** `GET /customs/cheds/{chedId}/quantities` returns the mapped model

**AC2 – Reserving reduces the available quantity**\
**Given** a CHED with available quantity\
**When** `ProcessedChedRequest` is called with `QuantityManagementIndication` of `1`, an MRN and commodity items\
**Then** the reservation is recorded against that MRN\
**And** a subsequent read shows the reduced available quantity

**AC3 – Release and delete produce different outcomes**\
**Given** a reservation exists against an MRN\
**When** `ChedClearanceRequest` is called with `GoodsClearanceInformationType.Item01`\
**Then** the ledger reflects a release\
**And** calling it with `GoodsClearanceInformationType.Item02` instead reflects a deleted reservation, distinguishable on a subsequent read

**AC4 – Invalid operations produce customs faults**\
**Given** a request reserves more than the available quantity, or references an unknown MRN or unknown CHED\
**When** the operation is processed\
**Then** the simulator returns a customs fault\
**And** Trade Gateway raises `CustomsFaultException` and returns 502

**AC5 – Header fields are handled as TRACES handles them**\
**Given** a request carries a `CertexHeaderType` with a `MessageId` and `UniqRequesterPrefix`, and a competent customs office reference number\
**When** the simulator processes it\
**Then** those fields are validated or echoed consistently with real TRACES behaviour

**AC6 – The ledger is controllable from the REST API**\
**Given** the control API\
**When** a test seeds a CHED with specific quantities, or reads back the current ledger\
**Then** it can do so without issuing SOAP requests

**AC7 – Synthetic fixtures are replaced with real captures**\
**Given** access to TRACES acceptance\
**When** representative customs responses are captured\
**Then** the synthetic fixtures under `tests/Api.Tests/Samples/CUSTOMS/` are replaced\
**And** the corresponding open question in ADR-0006 is closed

## Testing Comments

- Sequence tests carry the value here: read, reserve, read, release, read, asserting the ledger at every step
- `tests/Api.Tests/Services/CustomsChedServiceTests.cs` already parses the outbound SOAP body to prove a read sends `QuantityManagementIndication` of `0` and never requests PDF generation, so the simulator should be strict enough that a request getting this wrong fails rather than quietly succeeding
- Confirm the Customs account is required on this port and the Default account is rejected

## Technical Notes

- The SOAP actions are fully qualified, ending `CustomsCertexChedPort/ProcessedChedRequest` and `CustomsCertexChedPort/ChedClearanceRequest`
- The gateway sends `SendingDate` as the current UTC time and `MessageId` as an N-format GUID
- `UniqRequesterPrefix` is the configured `TracesNt:CustomsOfficeReferenceNumber`, one to eight alphanumeric characters
- Mapping behaviour is documented in `docs/mappings/customs-quantity-mappings.md`
- ADR-0006: Customs Quantity Management describes the behaviour being reproduced here and records the open question about the synthetic fixtures

## Questions & Actions

1. Capture real TRACES behaviour for over-reservation, double release, and releasing an already-deleted reservation rather than inferring it
2. Confirm whether `chedInterventionRequest` follows in its own ticket once https://eaflood.atlassian.net/browse/CDMS-1567 merges
