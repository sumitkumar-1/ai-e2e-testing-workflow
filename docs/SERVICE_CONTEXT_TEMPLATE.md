# Service context template

Copy this template for each service. Mark unknowns as unknown rather than guessing. Do not include secrets, private production data, or unapproved internal material in a public repository.

## Ownership and applicability

- Service name, owner, repository reference:
- Applicable service/API/chart versions:
- Documentation revision and review date:

## Purpose and business contracts

- Responsibilities and explicit exclusions:
- Requirement sources and expected business outcomes:
- Consumers and downstream dependencies:

## Interfaces

- OpenAPI/message schema references:
- Protocols, endpoints, events and authentication requirements:
- Errors, retries, timeouts and idempotency behavior:

## Deployment and configuration

- Artifact/chart identification and commit/build mapping:
- Environment/namespace mapping (approved metadata only):
- Readiness and application-health checks:
- Configuration owner and supported update path:
- Relevant settings and reload/restart behavior:
- Restoration limitations:

## Verification

- Fixtures and setup operations:
- Happy, negative, boundary and regression scenarios:
- Cross-service sequence and final observable outcomes:
- Supported failure injection and recovery scenarios:
- Expected results and their authoritative sources:
- Existing reusable tests or internal testing libraries:

## Evidence and cleanup

- Correlation IDs, log/metric/trace query conventions:
- Direct business-outcome assertions:
- Fixture deletion and configuration/deployment restoration:
- Known limitations, ambiguities and escalation owner:
