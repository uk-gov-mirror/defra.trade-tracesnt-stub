---
project: CDMS
type: Story
summary: Add a REST control API to the TRACES NT simulator
parent: CDMS-1620
key: CDMS-1622
---

## Overview

This is the simple face of the simulator, and it is deliberately not TRACES-shaped. TRACES does have an API for creating a CHED or status changes, but it is too complex to be worth mimicking here, and there is no reason to inherit the XML schema's complexity. A test author should not have to build SOAP envelopes to set up a fixture.

Provide a REST API with clean JSON models designed for the person writing the test, documented with OpenAPI, hosted on a path prefix separate from the SOAP service paths. What the control API writes, the SOAP face reads back.

Mapping the simple REST model onto the full TRACES XML inside the simulator is the point of the story: it is what lets the control surface stay small while the SOAP surface stays faithful.

This comes before CHED retrieval in the sequence deliberately: there is no other way to get a CHED into the simulator's state for that story to serve.

**Note:** Only CHED needs to be supported here. The models should be designed so that INTRA and DOCOM can be added later without reshaping the API, but those certificate types are optional and are covered by their own stories.

**Note:** Configuring the simulator to fail on purpose is not included here and is covered by the fault-injection story.

## User Story

**As a** tester writing Trade Gateway journey tests

**I want** a simple REST API to create and manipulate TRACES test data

**So that** I can set up exactly the scenario I need without crafting SOAP requests

## Acceptance Criteria

**AC1 – A documented REST API on its own prefix**\
**Given** the simulator is running\
**When** a developer opens the OpenAPI document\
**Then** every control operation is listed with a request and response example\
**And** the control API is served on a path prefix that cannot collide with the five SOAP service paths

**AC2 – CHEDs can be created and updated**\
**Given** the control API is available\
**When** a CHED is created or updated through it\
**Then** the operation succeeds and returns the stored representation\
**And** the request models are plain JSON designed for readability rather than copies of the TRACES XML schema

**AC3 – State can be reset**\
**Given** a test has created data\
**When** it calls the reset operation\
**Then** the simulator returns to an empty state or to a named fixture set\
**And** tests can be run in any order without interfering with each other


## Testing Comments

- Test create and update through the control API alone at this point, asserting on the stored representation returned in the response — there is no read-back endpoint, and the SOAP side isn't implemented yet. The end-to-end round trip, proving the data was actually persisted and can be read back, belongs to the CHED retrieval story, which depends on this one
- Cover reset explicitly, including that it clears data created by a previous test, since order-dependent tests are the failure this story exists to prevent
- Fixture sets should be plain files in the repository so a scenario can be shared and reviewed

## Technical Notes

- Keep the control models independent of the generated TRACES types, so the REST surface can stay simple as the SOAP surface grows
- Include deletion so a test can clear specific fixtures. State is stored as TRACES XML internally, so a read-back endpoint would need to map XML back to the control JSON model — deliberately not included; the SOAP face is the only way to read state back

## Questions & Actions

1. Agree the shape of the control models: how much of a certificate a test must supply and what the simulator defaults
2. Decide whether state is held in memory, which is simplest and resets on restart, or persisted
