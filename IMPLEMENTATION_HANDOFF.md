# Implementation handoff

## What the user wants

The user is a senior Java developer whose team works from Jira tickets. An SDET normally reads the ticket and PR diff, understands the affected feature, deploys the relevant services to QA/Sandbox/Stable through CKP, changes configuration as necessary, supplies inputs such as SMTP messages, inspects downstream behavior, and documents the result.

Build an application that can perform that same investigation and verification process. It must go beyond summarizing tickets, generating test cases, or merely running an existing CI suite. The requested foundation has five parts: Java integration/tool services, workflow backend APIs, an AI execution engine, internal service/library/tool context, and a frontend for configuration and verification.

The current repository is the planning handoff. The user asked to publish it for another session to continue; this does not mean the application has already been built or validated.

## Agreed constraints

1. Java is the backend language. Spring Boot, Maven, React/TypeScript, PostgreSQL and a durable workflow engine are proposed defaults, not all user-mandated selections.
2. MCP was discussed and deliberately made optional. Direct Java services exposed through an application tool registry are sufficient. Do not introduce MCP as a required foundation.
3. Tools work independently of AI. A dispatcher maps model-requested tool names and business arguments to authorized Java operations.
4. The application prompts for AD username/password. The backend obtains a user-scoped Delinea Secret Server token using the organization's supported authentication flow, including MFA if configured. Verify actual deployment behavior before implementing the exchange.
5. Connection configuration maps tools/environments to Delinea secret IDs and field identifiers. Connectors resolve credentials just in time. Do not introduce separate tool-password onboarding as the primary flow.
6. The AI module never exchanges, retrieves, chooses, stores or sees secrets or tokens. It has no secret-reading tool. Backend identity and connector infrastructure handle authentication. AI-provider authentication likewise belongs to transport infrastructure.
7. The initiating user's Delinea permissions constrain retrieval. A retrieved secret may represent a shared downstream account; preserve this distinction in auditing and authorization.
8. Broad API coverage is explicitly requested. Build a versioned endpoint catalog and track implementation/test coverage; an initial vertical slice is not completion of that goal.
9. Expected behavior comes from requirements and contracts. Do not let the implementation under test define its own correctness.
10. Reports distinguish PASS, FAIL, BLOCKED, NOT_TESTED and NEEDS_REVIEW, and link conclusions to evidence. Cleanup has its own recorded outcome.

## Confirmed environment context

- Source: Bitbucket/Stash; tickets: Jira; artifacts: JFrog Artifactory.
- Documentation: a wiki, product not confirmed. Confluence is a conditional integration, not an established fact.
- Observability: Grafana; actual data sources are unknown.
- CKP hosts Kubernetes services and has a configuration UI. kubectl and Helm are used. CKP-specific API details and configuration ownership are unknown.
- Services include AuthService, KeyService, SMTPReceiver, CLAP and RDA. Names do not establish their contracts or message flow.
- SMTP input can use Swaks or an existing CLI believed to be based on NioSmtpClient. Obtain its source/help before claiming command support.
- Private-host installations generally expose product APIs, but versions, editions, extensions and access differ. Match public specifications to installed products.
- Tickets range from small fixes/configuration changes to new features. Story points are not a reliable proxy for verification risk.

## End-to-end coverage details to make concrete

For representative tickets, capture prerequisites, fixture creation, cross-service behavior, expected final outcomes, evidence queries and cleanup. Define how change impact selects negative, boundary, authorization, regression, resilience and performance scenarios when relevant. Specify permitted failure injection for retry/timeout scenarios. Logs alone should not replace a directly observable business outcome where one is available.

Use three walkthroughs with an SDET: a configuration change, a reproducible bug fix, and a new feature spanning services. For each, trace every manual action to a planned tool and every expected result to an assertion and evidence source.

## Unknowns that must stay explicit

- Product versions, wiki identity, Grafana data sources, CI/build system and artifact-to-commit mapping.
- CKP endpoints, CRDs, environment ownership, deployment procedures and authoritative configuration path.
- Delinea version/hosting model, AD domain format, MFA, expiry/refresh/revocation, secret templates, approval/checkout policies.
- Internal service APIs, business rules, dependencies, fixtures, libraries and SMTP CLI behavior.
- Approved AI provider/runtime, API authentication and data-handling policy. Consumer subscription credentials are not assumed interchangeable with API credentials.
- Accepted pilot metrics, available staffing and final framework/runtime versions.

Do not request real passwords or token values in chat. Collect endpoint metadata and references through approved configuration, with secrets kept in Delinea. Public examples must remain synthetic.

## Suggested first implementation task

Read this handoff and the full implementation plan. Inspect the repository again for subsequent changes. Establish the versioned API catalog and architecture decisions, then scaffold the Java backend modules and frontend using company-compatible versions.

Build a fixture-backed vertical slice: ticket submission -> context retrieval -> structured plan -> authorized tool request -> sanitized tool result -> evidence report. Include a Delinea adapter interface and fake implementation for local development. Clearly label fake integrations and simulated evidence. Add the real login/secret-resolution path only against confirmed contracts and a test instance.

Next connect real read-only ticket/PR/artifact operations, then one Sandbox deployment/configuration/SMTP scenario with cleanup. Preserve a separate backlog for all remaining documented API families. Do not start real environment changes until the user has authorized the target and scope.

## Demonstration and acceptance

- Run an existing deterministic tool without AI.
- Let the model request the same tool through the dispatcher.
- Reject an unauthorized request before executing it.
- Prove synthetic credential markers never enter model requests, UI responses, logs, workflow history or evidence.
- Recover a run after worker interruption without duplicating side effects.
- Handle token expiry and logout without privileged fallback; report blocked cleanup.
- Produce criterion-to-evidence reports with exact build/configuration context.
- Keep documentation truthful about what is mocked, implemented, contract-tested and live-validated.

No private endpoints or live product integrations have been inspected in creating these documents. The full plan contains the official public documentation baseline and staged acceptance gates.
