# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A stand-in for the EU TRACES NT platform so Trade Gateway can be tested without reaching EU
acceptance. Three projects, one host process, split by URL path:

| Project | Role | Paths |
| --- | --- | --- |
| `src/Api.TradeTracesNTStub.Simulator` | CoreWCF SOAP simulator, plus the REST control API and its state | the five TRACES service paths, `/control/**` |
| `src/Api.TradeTracesNTStub.Mock` | Older WireMock stub of canned XML + proxy to EU acceptance | `/mock/**`, `/proxy/**` |
| `src/Api.TradeTracesNTStub.TestKit` | Packable builders + control-API client for other repos | — |
| `src/Api.TradeTracesNTStub` | Host: `Program.cs`, health, logging, Mongo, CDP boilerplate | `/health`, `/openapi/v1.json` |

The simulator is where new work goes; the WireMock stub is legacy and stays until journey tests move
across. They never overlap on a path, so both are live at once.

## Build and test

Restore needs a GitHub PAT with `read:packages` for the private DEFRA feed — `NuGet.config` reads it
from the environment, and the Dockerfile takes it as a BuildKit secret:

```bash
export DEFRA_NUGET_PAT=<token>
```

```bash
dotnet build
dotnet test --project tests/TradeTracesNTStub.Test/TradeTracesNTStub.Test.csproj

# Integration tests need the service already running on :8085
docker compose up --build -d      # or: dotnet run --project src/Api.TradeTracesNTStub
dotnet test --project tests/TradeTracesNTStub.IntegrationTests/TradeTracesNTStub.IntegrationTests.csproj \
  --filter-trait Category=IntegrationTest
```

`global.json` selects the Microsoft.Testing.Platform runner, so filters are xUnit v3 MTP flags, not
`--filter`:

```bash
dotnet test --project tests/... --filter-method '*GetChedCertificate*'
dotnet test --project tests/... --filter-class '*GatewayClientTests'
```

`TreatWarningsAsErrors` is on for every project (`Directory.Build.props`), targeting `net10.0` with
nullable and implicit usings enabled.

## Architecture notes

**Contracts are not generated here.** `TracesNT.WebServices` types come from the
`Defra.Trade.Gateway.TracesNT` NuGet package — the same package Trade Gateway generates its clients
into, which is what stops the two drifting. To pick up a TRACES schema change, regenerate in
`DEFRA/trade-gateway` (`scripts/update-webservices.sh`), publish, then bump the `PackageReference`
in **two** places: the simulator project and `tests/TradeTracesNTStub.IntegrationTests`.

**Service paths are a contract.** `TracesNtServices` holds the five path constants; the gateway
builds every endpoint as `{TracesNt:BaseUrl}/{ServicePath}`, so pointing a consumer at the simulator
is one setting (`TRACESNT__BASEURL`) and nothing else. Changing a path breaks that.

**WS-Security is genuinely validated**, in `WsSecurityMiddleware` ahead of CoreWCF rather than in a
dispatch inspector. `TracesNtServices.CredentialKeyByPath` maps each path to an account: the customs
port authenticates as `Customs`, the other four as `Default`. That separation is deliberate — a port
authenticating as the wrong account is the misconfiguration the simulator exists to catch. Rejections
return the untyped `env:Client` / `UnauthenticatedException` fault TRACES itself returns (the gateway
surfaces it as 502); typed per-certificate `PermissionDenied` faults are what produce a 403.

Credentials bind from `Simulator:Credentials:{Default|Customs}` with fake local defaults in
`appsettings.json`. No real TRACES credential belongs in this repository.

**Fault vocabulary.** Operations with a contract but no behaviour throw
`SimulatorFaults.NotImplemented(operation)` — a *receiver* fault, so it can't be mistaken for an
authentication failure (sender fault) or a genuinely missing certificate (typed `*NotFoundException`).

**CoreWCF binds only the schemes Kestrel listens on** (`SimulatorRegistrationExtensions`), because it
refuses to start otherwise. On CDP that is http alone; TLS terminates at the load balancer. Message
size limits are maxed to match the gateway's client bindings — smaller values truncate large
certificate responses.

**The WireMock stub runs in-process** on port 1080 (`WireMockHostedService`), reached through
`WireMockReverseProxyMiddleware`, which forwards `/mock/**` and `/proxy/**` to it. Its responses are
XML files under `Samples/**`, embedded as resources and looked up by their full resource name
(`Api.TradeTracesNTStub.Mock.Samples.CHED.<file>.xml`). Stubs are wired up in
`Extensions/WireMockStub/*`, matched on the `SOAPAction` header plus an XPath body matcher.

**The control API is the simulator's way in.** `Control/` holds a JSON API on `/control`;
`getChedCertificate` serves what it wrote. `ChedCertificateBuilder` constructs the whole
`SPSCertificateType` from the control model — there are no templates, and nothing is inherited from a
captured document. There is no read-back endpoint by design; the SOAP face is the only way to read
state, so the two cannot drift.

**The rule the whole design turns on:** the client specifies everything a submitter specifies, and
the simulator fills in only what TRACES fills in. Do not add a default for something a submitter
would send — a request must read as exactly the certificate it produces. The one deliberate exception
is the supporting document, which is fabricated when absent because every retrieved CHED has one.

**That boundary is documented, not guessed.** DG SANTE publish it per element in
`TNT-UN-CEFACT-Mappings-CHED-V2.xlsx` (sheet `CHED`, column `Issue`): `M`/`O`/`C` belong to the
submitter, `N` to TRACES. Check it before deciding which side of the line a new field sits on — the
sample submissions are misleading on their own, because they carry values TRACES ignores and
regenerates.

**Enrichment is load-bearing, not cosmetic.** TRACES returns codes carrying the display names it
looked up (`<StatusCode name="To be done (New)">1</StatusCode>`), and trade-gateway copies those into
`CodedValue.name` without ever deriving them. `Control/Lookups` supplies them from seed JSON holding
only the codes the tests use; an unknown code throws `UnknownCodeException` → 400 naming
the file to edit. Never make an unknown code silently produce a blank name. An empty seeded name is
different: it means the code genuinely has no label (package type `NA`), so the attribute is omitted.

**Two escape hatches worth knowing.** A party with a `name` and no `identifier` skips the operator
registry — TRACES calls it an operator created on the fly. And `PATCH /control/cheds/{id}` merges,
because notes, clauses and classifications are objects keyed by their code rather than lists; that is
why applying a decision is a short patch instead of restating the certificate.

**The generated types are stricter than they look.** Code values are enums whose members are named
`Item1`, `Item42` etc. with the real value in an `XmlEnumAttribute` — use `XmlEnums.Parse<T>(code)`,
never a hard-coded member. Some of those enums declare two members for one wire value. Several
wrappers are arrays where a single value reads naturally (`SPSNoteType.ContentCode`,
`SPSTradeLineItemType.OriginSPSCountry`) and singletons where an array does (`SPSPartyType.Name`);
check against the package's `TracesNT.xml` doc or IL rather than guessing.

**Two wire differences from real TRACES are expected and harmless.** The generated types' constructors
default `listID`/`listVersionID`, so the simulator emits attributes real TRACES omits — a consumer
deserialising with the same types sees identical values either way. And datetimes round-trip through
`DateTime`, so they come back at the host's UTC offset rather than the EU server's: same instant,
different rendering.

**A port with constructor dependencies must be registered** (`AddTransient<ChedCertificateSimulator>`).
CoreWCF only falls back to a parameterless constructor; the other four ports are stateless and rely on
that fallback.

**Mongo and the CDP platform boilerplate** are wired in `Program.cs` but nothing uses them yet — state
is deliberately in memory, so a restart is a reset.

## Testing approach

`GatewayClientTests` drives the simulator with Trade Gateway's *own* generated WCF clients, bound
exactly as the gateway binds them. This is the test that proves a consumer needs no code change, so
keep new simulator behaviour covered there rather than only through raw HTTP.

Integration tests hit `http://localhost:8085` directly (`IntegrationTestBase`) — there is no
`WebApplicationFactory`, so the service must be up first. WireMock stub responses are asserted with
Verify snapshots (`*.verified.xml` alongside each test).

`ChedRetrievalTests` covers the control-API-to-SOAP round trip, which is the only proof state is
actually persisted rather than echoed. `TestKitTests` drives the same path through the shipped
builders, as a consuming repository would. `ChedCertificateBuilderTests` has one test per derived
rule — issuer party, party name and role, CN expansion, baseport names, clause display text — which
matters more now that nothing is inherited from a captured document: a rule that stopped firing would
reach a consumer as a blank label rather than failing loudly.

## Conventions

`.editorconfig` is the authority. The ones that bite: private static fields are `s_camelCase`, other
private/internal fields are `_camelCase`. Simulator code is formatted to ~120 columns; the older Mock
project is not — match whichever file you're in.
