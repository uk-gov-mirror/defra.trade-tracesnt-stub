---
project: CDMS
type: Epic
summary: Build a TRACES NT simulator for Trade Gateway testing
key: CDMS-1620
---

## Overview

Trade Gateway integrates with the EU TRACES NT platform over SOAP. The only end-to-end test target today is the EU acceptance environment, which we cannot write to, cannot reset, and cannot reach without real EU credentials. We cannot create the certificates a test needs, we cannot drive a state transition such as a CHED status change, and the data underneath us moves without warning.

`DEFRA/trade-gateway-journey-tests` and `DEFRA/trade-gateway-local-environment` are both held at read-only smoke tests because of this.

This epic delivers a TRACES NT simulator in `DEFRA/trade-tracesnt-stub` with two deliberately different faces. The SOAP face is wire-compatible: every TRACES operation Trade Gateway calls is reproduced faithfully enough that the generated WCF clients reach the simulator by changing `TRACESNT__BASEURL` and nothing else. The server side is generated from the same `scripts/master.wsdl` the gateway's clients come from, so the two cannot drift. The control face is a plain REST API with clean JSON models for creating and mutating test data — TRACES has no such API and the simulator does not pretend otherwise.

The simulator holds real state, so what the control API writes the SOAP face reads back. The customs quantity ledger in particular behaves like a ledger: reserve, release and delete affect each other and affect what a subsequent read returns.

Required scope is CHED, across both the ports that serve it: `ChedCertificateServiceV2` behind `/certificates/cheds/**`, and `CustomsCertexChedServiceV06` behind `/customs/cheds/**`. That is what the first delivery must cover.

**Note:** The INTRA, DOCOM and reference data ports, and fault injection, are optional and low priority. They are captured as stories in this epic so the shape of the whole is visible, but they are not required for the epic to deliver value and should not be started before the CHED path is working end to end.

**Note:** This does not replace the in-process WireMock stubs in `tests/Api.Tests`, which stay for fast unit-level coverage of mapping and error handling. TRACES operations the gateway does not call, the TRACES web UI, and any production use are all out of scope.

## User Story

**As a** developer or tester working on Trade Gateway

**I want** a TRACES NT simulator I can seed, mutate and reset

**So that** I can run deterministic end-to-end tests without EU acceptance access

## Acceptance Criteria

**AC1 – Journey tests run against the simulator**\
**Given** the simulator is running\
**When** the `DEFRA/trade-gateway-journey-tests` suite is executed against it\
**Then** the suite passes\
**And** Trade Gateway requires no code change, only a different `TRACESNT__BASEURL`

**AC2 – No EU credentials are required**\
**Given** no TRACES NT acceptance credentials are present in the environment\
**When** the full journey test suite is run\
**Then** it completes successfully

**AC3 – The simulator is part of the local environment**\
**Given** a developer runs `docker compose up` in `DEFRA/trade-gateway-local-environment`\
**When** the environment starts\
**Then** the simulator starts alongside Trade Gateway, MongoDB and Floci\
**And** Trade Gateway is configured to use it by default

**AC4 – Both CHED ports are covered**\
**Given** the simulator is complete for the required scope\
**When** the CHED operations are exercised\
**Then** `getChedCertificate`, `findChedCertificate`, `ProcessedChedRequest` and `ChedClearanceRequest` all return realistic responses\
**And** `/certificates/cheds/**` and `/customs/cheds/**` can both be tested end to end

**AC5 – Optional ports do not block delivery**\
**Given** the INTRA, DOCOM and reference data ports are not yet implemented\
**When** the CHED journey tests are run\
**Then** they pass\
**And** the epic is deliverable without those stories

## Testing Comments

- The definitive check is the journey test suite passing with every TRACES NT credential removed from the environment
- Per story, run the matching Trade Gateway integration tests under `tests/Api.Tests/Endpoints/` against the simulator instead of WireMock and confirm the Verify snapshots still match

## Technical Notes

- Endpoints are built as `{TracesNt:BaseUrl}/{ServicePath}` in `src/TracesNT/TracesNtConfig.cs`, so one `TRACESNT__BASEURL` change redirects all five services
- The five service paths are `ChedCertificateServiceV2`, `EuIntraCertificateServiceV1`, `DocomCertificateRetrievalServiceV1`, `ReferenceDataServiceV1` and `CustomsCertexChedServiceV06`
- `scripts/master.wsdl` aggregates the five live EU WSDLs; `scripts/update-webservices.sh` is the generation pattern to mirror on the server side
- Authentication is WS-Security `UsernameToken` with `PasswordDigest` using two distinct accounts, Default and Customs — see `src/TracesNT/ClientBehaviours/WsSecurityHeader.cs`
- `tests/Api.Tests/Samples/` holds a ready-made seed corpus, though `Samples/CUSTOMS/` is synthetic and needs replacing
- The stories are sequenced so a working vertical slice arrives early: the control API has to exist before CHED retrieval has anything to serve, so it comes first
- Required stories, in order: stand up the simulator, add the control API, serve CHED retrieval, serve CHED search, implement the customs quantity ledger, run the journey tests against the simulator
- Optional and low priority, in no particular order: INTRA retrieval, INTRA search, DOCOM retrieval, reference data, fault injection
- Contracts are generated for all five services and all five paths are hosted, because generation is driven by the aggregated WSDL in a single pass and costs nothing extra per service. Only the CHED operations are implemented; everything else returns an explicit not-implemented fault
- Each optional story is therefore implementation work only, with no regeneration and no risk of the shared data contracts shifting underneath the CHED implementation

## Questions & Actions

1. Confirm whether the simulator is deployed to CDP dev and test, or run only locally and in ephemeral test environments
2. Confirm whether any consumer beyond the journey tests and the local environment needs designing for
