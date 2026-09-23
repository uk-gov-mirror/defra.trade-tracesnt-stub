---
project: CDMS
type: Story
summary: Stand up the TRACES NT simulator
parent: CDMS-1620
key: CDMS-1621
---

## Overview

Create the simulator service in `DEFRA/trade-tracesnt-stub` and get it to the point where Trade Gateway can talk to it: SOAP endpoints hosted at the paths the gateway expects, contracts generated from the real WSDL, and WS-Security validated. No TRACES operations are implemented — this is the foundation the CHED stories build on.

Trade Gateway builds every endpoint as `{TracesNt:BaseUrl}/{ServicePath}` in `src/TracesNT/TracesNtConfig.cs`, so hosting all five paths under one base URL makes `TRACESNT__BASEURL` the only switch a consumer needs. Contracts are generated from the same `scripts/master.wsdl` the gateway's clients come from, which is what stops the two drifting apart; the aggregated WSDL has to be generated in one pass for shared types such as `SPSPartyType` to be produced once, so all five services are generated whether or not they are implemented.

Authentication must be genuinely checked rather than waved through. A permissive simulator would hide the misconfiguration we most want to catch — a port authenticating as the wrong account, which is what `tests/Api.Tests/TracesNtCredentialsTests.cs` exists to detect.

**Note:** Generating a contract is not implementing it. Only CHED gets behaviour, in its own stories; every other operation returns an explicit not-implemented fault so it fails loudly rather than confusingly.

## User Story

**As a** developer working on Trade Gateway

**I want** a deployable simulator the gateway can reach and authenticate against

**So that** the TRACES operations can be built out on top of it

## Acceptance Criteria

**AC1 – The gateway reaches the simulator with one configuration change**\
**Given** Trade Gateway is configured with `TRACESNT__BASEURL` pointing at the simulator\
**When** it calls any of the five TRACES services\
**Then** the call reaches the simulator and returns a well-formed SOAP response or fault\
**And** no Trade Gateway code has changed

**AC2 – Contracts come from the shared package**\
**Given** the simulator needs the TRACES contracts\
**When** it is built\
**Then** it consumes `Defra.Trade.Gateway.TracesNT`, the same package Trade Gateway's clients are generated into, so the two cannot drift\
**And** the process for picking up a TRACES schema update — regenerate in `trade-gateway`, publish, bump the version here — is documented

**AC3 – The gateway's clients can reach every port**\
**Given** the simulator is running\
**When** Trade Gateway's generated `*PortClient` types open a channel against each of the five endpoints\
**Then** each call returns a well-formed SOAP fault rather than a binding, address or serialisation error, proven by an automated test

**AC4 – WS-Security is validated and the two accounts are kept separate**\
**Given** the simulator is configured with distinct Default and Customs credentials\
**When** a request presents a `wsse:UsernameToken` with a valid digest for the right account\
**Then** it is processed\
**And** a wrong key, unknown username, missing `wsse:Security` header, expired `wsu:Timestamp`, or the wrong account for the port is rejected with the fault shape TRACES returns, which the gateway surfaces as 502

**Note:** an authentication failure is an untyped `env:Client` fault, so it falls through the gateway's typed `catch (FaultException<T>)` guards into `TracesCommunicationException` and comes out as 502. 403 is reachable only via the typed `*PermissionDeniedExceptionType` faults, which are per-certificate authorisation rather than authentication.

**AC5 – Unimplemented operations fail explicitly**\
**Given** an operation has no implementation behind it\
**When** the gateway calls it\
**Then** the simulator returns a SOAP fault naming the operation as not implemented\
**And** the fault is distinguishable from a missing certificate and from an authentication failure

**AC6 – The service runs locally and deploys to CDP**\
**Given** the repository is checked out\
**When** the service is started with `docker compose up`\
**Then** the container starts and its health endpoint returns 200\
**And** it starts with no TRACES NT credentials configured and makes no outbound call to the EU estate\
**And** it deploys successfully to CDP dev

## Testing Comments

- The channel-open test in AC3 is the one that earns its keep: driving the real `*PortClient` types is what catches contract and header mismatches a hand-written envelope would sail past
- Recompute the digest in a test the way `tests/Api.Tests/TracesNtCredentialsTests.cs` does, and exercise the cross-account case in both directions, so the simulator cannot pass by ignoring credentials entirely
- A round-trip serialisation test is no longer meaningful: sharing one package means simulator and gateway use the same types, so it would round-trip through the same code

## Technical Notes

- CoreWCF for SOAP hosting on `net10.0`; follow the CDP service template for the health endpoint, logging and Dockerfile
- Both HTTP and HTTPS must work, and message size limits must not be smaller than the gateway's `int.MaxValue`, or large certificate responses truncate — see `src/TracesNT/Extensions/ServiceRegistrationExtensions.cs`
- CoreWCF recognises the package's `System.ServiceModel` contract attributes by name and namespace, so the client-generated ports host server-side unchanged. `XmlSerializerFormat` is covered too, but a test pins it — a silent fallback to the DataContractSerializer would put every response on the wire in the wrong shape
- `WebServiceClientId` is declared in `.../sanco/tracesnt/base/v3` on the customs contract and `v4` on the other four, so match it on local name
- The security header the gateway sends is built in `src/TracesNT/ClientBehaviours/WsSecurityHeader.cs`: a `UsernameToken` with a `PasswordDigest` of `Base64(SHA1(nonce || created || authenticationKey))` and a two-minute `wsu:Timestamp`. SHA-1 is mandated by the WS-Security profile rather than chosen
- Credentials come from configuration with obviously-fake local defaults; no real TRACES NT credential belongs in the repository

## Questions & Actions

1. ~~Decide whether to generate contracts independently in both repositories or publish a shared NuGet package~~ — **answered:** shared package, `Defra.Trade.Gateway.TracesNT`
2. ~~Capture the exact TRACES fault envelope for an authentication failure from acceptance~~ — **answered:** `Samples/INTRA/UnauthenticatedException.xml` is a real acceptance capture; an `env:Client` fault with faultstring `UnauthenticatedException` over HTTP 500
3. ~~Confirm which CDP service template the repository should be based on~~ — **answered:** already on the CDP .NET backend template
4. ~~The simulator is pinned to a build from `trade-gateway` PR 48~~ — **done:** on the published `0.1.0`
