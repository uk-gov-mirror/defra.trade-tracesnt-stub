---
project: CDMS
type: Story
summary: Run the journey tests against the TRACES NT simulator
parent: CDMS-1620
key: CDMS-1663
---

## Overview

The story that realises the epic's value. Add the simulator to `DEFRA/trade-gateway-local-environment`, point Trade Gateway's local and CDP configuration at it, and convert `DEFRA/trade-gateway-journey-tests` to set up its own data through the control API rather than depending on whatever happens to exist in EU acceptance.

Until this lands, the simulator is a service nobody uses. After it, the journey test suite runs from a clean slate on any machine with no EU credentials.

**Note:** The suite covers the CHED path only, across `/certificates/cheds/**` and `/customs/cheds/**`. INTRA, DOCOM and reference data journeys follow when those optional stories are picked up.

## User Story

**As a** developer or tester working on Trade Gateway

**I want** the journey tests to run against the simulator by default

**So that** the suite is deterministic and needs no EU acceptance access

## Acceptance Criteria

**AC1 – The simulator is part of the local environment**\
**Given** a developer runs `docker compose up` in `DEFRA/trade-gateway-local-environment`\
**When** the environment starts\
**Then** the simulator starts alongside Trade Gateway, MongoDB and Floci\
**And** Trade Gateway's `TRACESNT__BASEURL` points at it by default

**AC2 – Journey tests manage their own data**\
**Given** the journey test suite\
**When** a test runs\
**Then** it creates the data it needs through the control API, exercises the gateway, and resets afterwards\
**And** the suite passes when run repeatedly and in any order

**AC3 – No EU credentials are required**\
**Given** no TRACES NT acceptance credentials are present in the environment\
**When** the full journey test suite is run\
**Then** it completes successfully

**AC4 – Error paths are covered**\
**Given** the simulator can be made to return a not-found response, a permission-denied fault and a customs fault\
**When** the journey tests run\
**Then** each of those paths is asserted end to end against the gateway's HTTP response

**AC5 – Documentation is updated**\
**Given** a developer new to the repositories\
**When** they read the README in Trade Gateway and in the journey tests repository\
**Then** they can see how to run against the simulator\
**And** they can see when it is still appropriate to run against EU acceptance

## Testing Comments

- Running the suite twice in succession without an intervening restart is the check that reset is working
- Confirm the suite fails clearly, rather than passing vacuously, if the simulator is not running
- Keep at least one documented path for running against acceptance, since the simulator cannot prove the real EU contract has not changed

## Technical Notes

- Trade Gateway's compose setup is in `compose.yml` and `compose/config/trade-gateway.env`, and TRACES configuration is passed as `TRACESNT__*` environment variables
- The gateway's `.env.example` and the Docker Compose section of `README.md` both list the TRACES variables and will need updating to describe the simulator defaults
- Journey test principals are configured in Trade Gateway and need authorisation entries, so the `Permissions` and `Principals` split described in `README.md` applies; ADR-0004: Dual-Issuer Authentication (Cognito + STS) and ADR-0005: Fine-Grained Authorisation are the reference for granting them access

## Questions & Actions

1. Decide whether a scheduled contract-check job should still run the suite against EU acceptance to detect upstream schema changes
2. Confirm whether CDP dev and test point at the simulator or continue to use acceptance
