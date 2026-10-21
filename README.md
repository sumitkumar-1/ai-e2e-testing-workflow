# AI end-to-end ticket verification

An internal application that helps SDETs verify Jira tickets by understanding requirements and PR changes, deploying microservices, exercising behavior, collecting evidence, and restoring the environment.

**Current status: planning only. No application, live integration, or automated verification has been implemented.**

## Start here

1. [Implementation handoff](IMPLEMENTATION_HANDOFF.md): user intent, agreed constraints, unknowns, and a suggested starting task for another session.
2. [Implementation plan](IMPLEMENTATION_PLAN.md): architecture, integration coverage, workflow, frontend, delivery phases, and official documentation sources.
3. [Service context template](docs/SERVICE_CONTEXT_TEMPLATE.md): information to collect for each internal service.

## Intended experience

Sign in with AD credentials through the backend's Delinea integration, choose an environment, and enter a Jira ticket. The application retrieves requirements and code changes, proposes a verification plan, executes authorized tests, and produces a report linking each conclusion to evidence. It tracks cleanup separately from test results.

## Core decisions

- Java backend with ordinary integration services; MCP is not required.
- AI selects operations through tool calling. Backend services perform the operations.
- AD/Delinea authentication and secret exchanges happen entirely outside the AI module.
- Tool configuration contains Delinea secret IDs and field mappings, never credential values.
- Full integration API coverage is a goal scoped to actual installed versions; it is distinct from which operations a particular run may invoke.
- Service knowledge and business expectations must be supplied explicitly. Do not invent internal APIs or service behavior.

Known platforms: Jira, Bitbucket/Stash, JFrog Artifactory, wiki, Grafana, CKP/Kubernetes, Helm, kubectl, and SMTP testing through an existing NioSmtpClient-based CLI or Swaks. Known environments: QA, Sandbox, Stable.

Examples in this repository use placeholders. Do not commit credentials, private hostnames, production data, or raw internal test evidence. There are no build/run instructions yet because implementation has not started.
