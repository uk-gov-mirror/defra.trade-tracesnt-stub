---
project: CDMS
type: Story
summary: Add fault injection to the TRACES NT simulator
parent: CDMS-1620
---

## Overview

Trade Gateway has a considerable amount of code devoted to what happens when TRACES misbehaves: the exception types in `src/TracesNT/Exceptions/` and their HTTP mapping in `src/Api/Utils/GlobalExceptionHandler.cs`. Against the real acceptance environment none of it can be exercised deliberately, because we cannot ask TRACES to fail on demand.

Let a test ask the simulator to misbehave on purpose: return a specific SOAP fault, deny permission, return malformed XML, delay past the client timeout, or return a server error. Injected behaviour is configured through the control API and scoped narrowly enough that one test's induced failure does not affect another's happy path.

**Note:** This ticket is optional and low priority. Two of the four fault paths can already be triggered without it — a permission-denied fault by presenting the wrong credentials, and a customs fault by over-reserving quantity — so the required CHED scope is deliverable without this story. What it adds is the malformed-response and timeout paths, which cannot otherwise be provoked.

## User Story

**As a** tester writing Trade Gateway journey tests

**I want** to make the simulator fail on demand

**So that** the gateway's error handling is tested rather than assumed

## Acceptance Criteria

**AC1 – Faults are configurable through the control API**\
**Given** the control API is available\
**When** a test configures a fault for a named operation or a specific certificate ID\
**Then** subsequent matching requests fail in the configured way\
**And** requests that do not match are unaffected

**AC2 – Permission denied maps correctly**\
**Given** a permission-denied fault is configured\
**When** the gateway calls the affected operation\
**Then** it raises `PermissionDeniedException` and the API returns 403

**AC3 – Customs faults map correctly**\
**Given** a customs fault is configured on the customs port\
**When** the gateway calls the affected operation\
**Then** it raises `CustomsFaultException` and the API returns 502

**AC4 – Malformed responses map correctly**\
**Given** the simulator is configured to return malformed SOAP\
**When** the gateway calls the affected operation\
**Then** it raises `InvalidSoapException` and the API returns 502

**AC5 – Timeouts and transport failures are inducible**\
**Given** a delay longer than the client timeout, or an HTTP server error, is configured\
**When** the gateway calls the affected operation\
**Then** it raises `TracesCommunicationException` and the API returns the agreed error response

**AC6 – Injected faults are cleared by reset**\
**Given** faults have been configured\
**When** the control API reset operation is called\
**Then** all injected behaviour is cleared and the simulator returns to normal responses

## Testing Comments

- Every fault type needs a test asserting the resulting Trade Gateway HTTP status, since the mapping is the thing under test rather than the fault itself
- Confirm that scoping works: a fault injected for one certificate ID must not affect a request for another
- Confirm the error response body does not leak TRACES implementation detail, which is the behaviour the exception handler exists to provide

## Technical Notes

- The gateway's fault types are `PermissionDeniedException`, `CustomsFaultException`, `InvalidSoapException` and `TracesCommunicationException` in `src/TracesNT/Exceptions/`
- The timeout case needs a delay longer than the WCF client's send timeout, so the configured delay must be adjustable

## Questions & Actions

1. Confirm the fault scoping granularity the journey tests need: per operation, per certificate ID, or a request count before failing
