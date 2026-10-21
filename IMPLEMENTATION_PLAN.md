# AI verification workspace — implementation plan

Status: proposed plan, not an implemented system. Prepared 20 October 2026.

## 1. Product objective

Build an internal application where an authenticated user connects workplace tools, supplies a Jira ticket and target environment, reviews a verification plan, and starts an AI-assisted verification run. The application gathers requirements and PR changes, prepares the environment, executes tests, investigates outcomes, restores changes, and produces evidence-backed results and documentation drafts.

The backend is Java. Integrations are ordinary application services; MCP is not required. Services must also be usable from deterministic workflows and integration tests without an AI model.

Initial environments are QA, Sandbox, and Stable through CKP. Known deployed services include AuthService, KeyService, SMTPReceiver, CLAP, and RDA. Their responsibilities must come from internal documentation rather than assumptions based on names.

## 2. Architecture and proposed defaults

Use a modular Java backend with separate execution workers, rather than deploying a microservice for every connector initially.

Proposed stack, subject to company standards and version compatibility:

- Company-supported Java LTS, Spring Boot, Maven multi-module build.
- React and TypeScript frontend.
- PostgreSQL for users, connection metadata, plans, workflow records, and evidence metadata.
- Delinea Secret Server for tool credentials; configuration/database stores secret IDs and field mappings only. AD-backed login obtains a user-scoped Delinea token held on the backend.
- Object storage for reports, test outputs, traces, and evidence attachments.
- Durable workflow engine with Java support; evaluate against existing infrastructure before selecting one. Requirements include persisted state, timers, cancellation, retries, and compensating cleanup.
- Isolated container workers inside the corporate network for Helm, kubectl, SMTP CLI, and generated test execution.
- AI provider adapter with one provider implemented first and a stable internal tool-call contract for additional providers.

Logical flow:

```text
Frontend -> Application API -> Durable verification workflow
                                   <-> AI provider adapter
                                   -> Authorized tool dispatcher
                                   -> Java integration services
                                   -> Isolated execution workers
                                   -> Evidence and report storage
```

The AI proposes actions. The dispatcher validates and authorizes them. Integration services execute them. The workflow owns run state and recovery. Secrets never enter model prompts or tool definitions.

Suggested backend modules:

```text
application-api, identity, delinea-secrets, connections, integration-clients,
tool-registry, tool-execution, workflow, ai-providers,
knowledge, evidence, reporting, worker-runtime
```

### AD login and Delinea Secret Server credential flow

User requirement: on entry, prompt for AD username/password (and domain or MFA challenge where required). The Java backend authenticates against the organization's AD-integrated Delinea Secret Server deployment and obtains a token for that user. This replaces the earlier generic SSO/pasted tool-credential onboarding assumption. Validate the exact grant, username/domain format, MFA and endpoint against the installed deployment; an AD login alone does not automatically produce a Delinea token.

```text
User enters AD credentials over HTTPS
  -> Java authentication service calls approved Delinea authentication endpoint
  -> Delinea authenticates user and returns token
  -> Backend binds token to verified user and application session
  -> Browser receives only an opaque application-session cookie

Authorized tool invocation
  -> Backend resolves configured tool/environment/secret ID
  -> Secret resolver uses that user's Delinea token
  -> Delinea enforces permission to retrieve the secret
  -> Connector extracts configured credential fields and calls approved tool host
  -> Sanitized result returned to workflow/AI; credentials are discarded after use
```

Example configuration (illustrative IDs and field names, not actual secrets):

```yaml
tools:
  bitbucket:
    baseUrl: https://bitbucket.corp.example
    authentication:
      provider: delinea
      secretId: 12345
      type: bearer
      fieldMappings:
        token: api-token
  artifactory:
    baseUrl: https://artifactory.corp.example
    authentication:
      provider: delinea
      secretId: 23456
      type: basic
      fieldMappings:
        username: username
        password: password
```

Actual authentication types and secret-template field identifiers are selected per installation. Profiles may be user-, team-, or environment-specific, but retrieval always uses the initiating user's authorized Delinea session. AI provider API keys may use the same secret-reference mechanism when provisioned in Delinea.

Implement `DelineaAuthenticationService`, `DelineaSecretResolver`, and `CredentialFieldMapper`. These are backend-only components, never AI-callable secret-reading tools. Resolve approved mappings on the server; tool-call arguments must not choose secret IDs, user identities, or arbitrary credential destinations.

Explicit module boundary: the AI planning/execution module never performs secret exchanges. It has no Delinea client, secret-resolver dependency, secret-reading tool, or access to credential/session stores. Its inputs and outputs contain only permitted tool definitions, business arguments, and sanitized results. The identity service performs the login/token exchange; connector infrastructure resolves and uses tool credentials internally. Secret retrieval is an implementation detail of executing an authorized tool, never a model decision or a step delegated to the model.

```text
AI module -> tool name + business arguments -> authorized dispatcher
  -> connector -> internal secret resolver -> Delinea
  -> connector authenticates to target system
  -> sanitized result -> AI module
```

Enforce this separation through Java module dependencies, restricted credential-store access and network controls appropriate to the deployment. Include architecture tests proving the AI module cannot depend on Delinea/credential-resolution components, and execution tests proving authentication failures return sanitized status rather than secret material. AI-provider transport authentication, when needed, is likewise handled by backend transport infrastructure; provider credentials are never model-visible or managed through AI tool calls.

Credential lifecycle and authorization requirements:

- AD passwords are transient authentication inputs: never persist them or record them in logs, traces, analytics, workflow history or exception bodies. Clear the form after submission and minimize in-memory lifetime; do not promise guaranteed Java memory erasure.
- Keep Delinea access/refresh tokens in a protected backend session store with expiry; encrypt any persisted session secrets. Never return them to the frontend or store them in browser local storage. Use Secure, HttpOnly, appropriately scoped SameSite cookies and CSRF protection for application sessions.
- Fetch tool credentials just in time; default to no cross-operation plaintext credential cache. Never put them in application config, model context, audit payloads, evidence, queue messages, CLI arguments or durable workflow state. Trusted execution code necessarily handles credentials transiently; “not exposed” means absent from these user/model/storage surfaces, not absent from runtime memory.
- Apply both application permissions and Delinea secret permissions. A secret may contain a shared downstream account, so the downstream identity is not necessarily the AD user. Audit the initiating user, connection/secret reference and downstream identity metadata without recording credential values.
- Bind secrets to approved tool hosts and environments. Control mapping edits and reject arbitrary hosts/redirects that could receive credentials. A secret ID is a reference, not authorization.
- Honor applicable secret checkout, approval, access-reason and check-in requirements. If access is pending or denied, report the operation as blocked; never substitute a privileged token.
- Refresh tokens only when supported and permitted. Expiry, revocation or rotation must invalidate cached credentials; pause dependent work for reauthentication when required. Reconcile prior side effects before retrying an operation.
- Browser closure alone does not invalidate a still-authorized run. Explicit logout invalidates the application/Delinea session and revokes remotely where supported; stop new credential-dependent actions. If cleanup cannot run, mark it pending/blocked, retain environment coordination state and notify the user for reauthentication or an authorized operator. No silent account fallback.
- Workers receive only the minimum operation-specific credential material through a protected short-lived channel; never the user's AD password or general Delinea token. Minimize its lifetime and remove temporary credential material after execution.

Acceptance tests: successful AD/Delinea login including configured MFA; invalid credentials; forbidden secret ID; two-user isolation; host/mapping authorization; template-field mismatch; expired/revoked token; rotation; logout during execution; cleanup requiring reauthentication; and verification that synthetic credential markers never appear in frontend responses, logs, model requests, workflow records or evidence.

Delinea documents token-based authentication, including user prompts and 2FA; its SDK includes domain-password authorization. The exact supported flow must be verified against the private installation, not assumed from public examples. See the Delinea sources in section 11.

## 3. Workstream A — Java integration services and full API coverage

### Define the coverage boundary first

“All supported functionality” means all documented operations for each agreed installed product, edition, version, enabled module, and internal extension. It does not mean every historical or future API version. Public documentation is a baseline; private hosts may have different versions, plugins, and configuration.

The end goal is complete client coverage within that boundary. Delivery is incremental, starting with ticket verification operations. Administrative operations remain in the coverage backlog and are permission-controlled; they are not silently excluded from scope.

Create a versioned API catalog with one row per operation:

```text
product, edition, version, module, operationId, method/path or command,
documentation source, request/response schema, authentication,
permissions, side effects, pagination, asynchronous behavior,
client implementation status, test status, AI exposure policy
```

Track documented, implemented, contract-tested, and live-validated counts separately. Do not label generated methods as fully verified. Any unavailable/licensed-out operation remains visible with its reason; scope exceptions require explicit agreement.

### Integration inventory

| Integration | API families to inventory | First verification slice |
|---|---|---|
| Delinea Secret Server | Version-specific authentication, token lifecycle, secret retrieval, templates and applicable access workflows | AD-backed user session and just-in-time secret resolution by configured ID; backend-only |
| Jira | Issues, search, fields, comments, attachments, links, transitions, projects, agile modules, permissions and administration | Read ticket, acceptance criteria, linked work; draft/publish verification summary under policy |
| Bitbucket/Stash | Projects, repositories, refs, commits, source, diffs, PRs, reviews, build/deployment metadata, hooks and permissions | Read PR/diff and files at pinned commits; associate artifacts |
| Artifactory | Repositories, artifacts, metadata/properties, checksums, search/AQL, build info, transfer/promotion, permissions and administration | Resolve immutable artifact and build provenance |
| Wiki | Product-specific search, pages/content, versions, attachments, spaces, permissions and administration | Retrieve relevant documentation with source revision |
| Grafana | Dashboards, folders, data sources/querying, annotations, alerting, organizations and permissions | Resolve service dashboards and gather relevant observations |
| Kubernetes | Discovered API groups/resources, workloads, configuration, services, events, logs, watches, RBAC, installed CRDs | Inspect environment, patch permitted configuration, monitor rollout |
| Helm | Version-specific CLI commands and flags for repositories/registries, charts, releases, values, status, history, upgrades and rollback | Deploy pinned chart/artifact, inspect release, restore where possible |
| CKP | Internal API contracts, configuration ownership, deployment abstractions and any custom resources | Use authoritative configuration/deployment path |
| SMTP tooling | Actual NioSmtpClient-based CLI commands/options and Swaks capabilities if retained | Controlled SMTP submission, transcript and message identifiers |
| Internal services/libraries | OpenAPI, async/message schemas, library interfaces and test utilities | Execute scenarios for selected service |

Grafana configuration APIs do not automatically cover every underlying observability backend. Inventory actual log, metric, and trace data sources and add their query interfaces where necessary. Do not assume Loki, Prometheus, or Tempo are installed.

### Implementation approach

1. Obtain version-matched specifications and vendor docs. Generate Java clients where usable machine-readable specifications exist; review generated behavior and fill gaps manually.
2. Keep vendor DTOs inside clients; expose stable domain operations through Java services.
3. Support configurable approved base URLs, context paths, corporate CA certificates, proxies, pagination, limits, bounded retries and structured errors.
4. Use a compatible Kubernetes Java client; discover installed APIs and custom resources. Run Helm through a controlled CLI adapter unless an established internal API owns the action.
5. Invoke CLIs with validated argument arrays, isolated working directories and scoped credentials, not model-generated shell strings.
6. Return operation IDs for long-running work, with status and evidence retrieval methods.
7. Review retry behavior per operation. SMTP sends and deployments may have uncertain outcomes; reconcile before retrying.
8. Keep connector credentials scoped to the initiating user and approved operation. Never fall back to administrator credentials.

Deliverables: API catalog, Java clients/services, configuration schemas, contract tests, integration validation records, and coverage report.

## 4. Workstream B — application backend and durable execution

Core entities:

- User, DelineaSessionReference, ConnectionProfile, SecretReference (ID and field mapping only), EnvironmentProfile. Token values are kept in the protected session store, outside workflow records.
- KnowledgeDocument and revision, ServiceDefinition, ToolDefinition and revision.
- VerificationRun, PlanRevision, AcceptanceCriterion, TestScenario.
- ToolInvocation, ExternalOperation, EnvironmentReservation, ChangeRecord.
- EvidenceItem, AssertionResult, VerificationReport, PublicationRecord.

Each run stores initiating identity, ticket snapshot, PR/commit, artifact digest, environment, configuration baseline, service versions, knowledge revisions, model identifier/settings, tool revisions, and budgets.

Proposed application API families:

```text
/auth                        AD/Delinea login, configured MFA, status, reauthentication, logout
/connections                 manage approved hosts/secret-ID mappings and test access; no secret readback
/environments                permitted targets and capabilities
/knowledge                   versioned service/library/tool documentation
/api-coverage                integration inventory and implementation status
/verification-runs           create and inspect runs
/verification-runs/{id}/plan review, revise and approve plan
/verification-runs/{id}/events live progress and completed actions
/verification-runs/{id}/cancel request cancellation and cleanup
/verification-runs/{id}/evidence authorized evidence access
/verification-runs/{id}/report retrieve/export report
/verification-runs/{id}/publish publish selected report under policy
```

Proposed lifecycle:

```text
CREATED -> COLLECTING_CONTEXT -> PLANNING -> WAITING_FOR_INPUT/REVIEW
-> PREPARING_ENVIRONMENT -> EXECUTING -> EVALUATING
-> CLEANING_UP -> COMPLETED
```

Record verification verdict and cleanup status separately. A completed run can be failed or blocked; a passed test run with failed cleanup still requires attention.

Reliability requirements:

- Browser closure does not stop a run.
- Duplicate submissions have idempotency keys.
- Persist decisions and tool results before advancing; avoid repeating model decisions or side effects after recovery.
- Reserve shared environments or use isolated namespaces; detect drift and concurrent modifications.
- Record changes before mutation. Cleanup uses conditional restoration and never blindly overwrites another operator's changes.
- Cancellation, worker failure and expired credentials have defined recovery paths.
- Cleanup is compensating work, not a guaranteed transaction rollback. Migrations or irreversible side effects require an explicit supported procedure.
- Enforce per-run time, cost, tool-call and mutation budgets.

## 5. Workstream C — AI planning and verification brain

Implement one agent loop initially, with separate structured planning and evaluation stages. Multiple agents are not necessary for the foundation.

Loop:

1. Resolve ticket, PR and immutable build; flag ambiguous associations.
2. Retrieve relevant authorized service documentation, library context and test guides.
3. Produce a structured plan linking acceptance criteria to scenarios, inputs, expected outcomes, actions and evidence requirements.
4. Identify ambiguity, missing access or unsupported operations before execution.
5. Give the model the relevant permitted tool definitions.
6. Validate each proposed tool call against schema, run scope, identity, permissions and operation policy.
7. Execute using Java services and workers; redact results and persist evidence.
8. Feed results to the model for bounded investigation or the next scenario.
9. Evaluate assertions, identify coverage gaps, and prepare the report.
10. Complete workflow-owned cleanup and report its actual outcome.

Tool contracts include name/version, description, argument schema, result schema, permission requirements, side effects, timeout, retry/idempotency policy and evidence behavior. Identity and credential references are supplied by the backend, never model-selectable parameters.

The provider adapter translates these contracts and tool results to each supported provider. Provider differences in streaming, tool schemas and cancellation require adapter tests. Do not assume consumer subscription tokens are interchangeable with API credentials. Exact provider authentication is an implementation decision to validate against official docs and company access.

Verification rules:

- Expected results originate from approved requirements/contracts, not solely from implementation code.
- Prefer existing executable tests; retain reviewed generated tests as reusable assets.
- Run generated code in restricted workers with explicit resource/network limits.
- Use deterministic assertions where possible. Label subjective or unresolved interpretation as requiring review.
- Criteria statuses: PASS, FAIL, BLOCKED, NOT_TESTED, NEEDS_REVIEW.
- Overall PASS requires all required criteria passed and no unresolved mandatory coverage gaps.
- Distinguish observed product defects from environment, tool and permission failures.
- For bug fixes, attempt fail-before/pass-after validation where reproducible and feasible.
- Treat repository text, tickets, wiki pages and logs as data, not instructions that can expand permissions.

## 6. Workstream D — internal knowledge and context

Maintain reviewed Markdown plus machine-readable contracts in source control. Indexing must preserve document access permissions and source revisions.

```text
knowledge/
  services/{AuthService,KeyService,SMTPReceiver,CLAP,RDA}/
    overview.md
    deployment.md
    configuration.md
    verification.md
    troubleshooting.md
    contracts/
  libraries/<name>/overview.md
  testing-tools/<name>/usage.md
  environments/<name>.md
  business-rules/
```

Service template: owner, applicable versions, purpose, business rules, dependencies, endpoints/protocols, events, deployment/artifact mapping, readiness, configuration ownership/reload behavior, fixtures, expected outcomes, evidence queries, cleanup, known limitations and last review date.

Library template: purpose, version compatibility, public interfaces, consumers, behavioral contracts and test fixtures. Testing-tool template: source/location, version, exact CLI syntax, parameters, outputs, exit codes, supported scenarios and cleanup.

Begin with structured metadata and keyword search; add semantic retrieval only if measured retrieval gaps justify it. Pin selected context revisions for each run. Stale or contradictory guidance must be surfaced.

Deliverables: initial service guides completed with SDETs, internal tool documentation, dependency/environment mappings and knowledge retrieval tests.

## 7. Workstream E — frontend

Screens:

1. Sign-in and profile: AD username/password prompt, configured domain/MFA handling, backend Delinea authentication, roles, provider selection and reauthentication state. No tool credential entry is required for Delinea-managed connections.
2. Connections: approved host, product/version, auth method, Delinea secret ID, field mappings, connection test and access/expiry status. Show metadata only, with no secret-value reveal or readback. Restrict configuration edits by role.
3. Environments: accessible environments, namespaces, service versions and reservation status.
4. New verification: ticket, environment, build selection, notes and execution limits.
5. Plan review: acceptance criteria, planned scenarios, changes, missing information and editable plan revisions.
6. Live run: durable timeline, operation status, sanitized evidence, questions and cancel control.
7. Results/history: criterion coverage, failures, reproduction steps, evidence links, cleanup status, export and authorized publication.
8. Knowledge: service/library/tool documents, owners, versions and review state.
9. API coverage/admin: implementation matrix, access policies, worker health and usage.

Do not expose raw implementation details unless needed to make a decision. Keep a technical evidence view available for SDETs.

## 8. Evidence and documentation output

Each evidence item records run/scenario, source tool, acquisition time, relevant correlation IDs, artifact/config context, storage location and content hash. Preserve authorized original outputs according to retention policy and redact model-facing/user-facing copies as appropriate. Evidence permissions must not broaden source access.

Report includes scope, provenance, criterion-to-test mapping, expected/observed outcomes, source evidence, product versus infrastructure failures, untested areas, reproduction procedure, and cleanup result.

Provide HTML/Markdown and structured JSON initially. Other document formats can follow. Wiki updates and bug reports are drafts unless publication is explicitly authorized by the run/team policy. Preserve earlier reports when a run is repeated or a reviewer changes a conclusion.

## 9. Delivery phases and acceptance gates

| Phase | Deliverable | Acceptance gate |
|---|---|---|
| 0: Discovery | Installed product/version inventory, API catalog scope, environment ownership, representative tickets | Each integration has authoritative docs/spec source and unresolved gaps recorded |
| 1: Foundation | Java modules, application API, AD/Delinea login and secret-ID resolution, run persistence, tool dispatcher, basic UI | User can log in and invoke an allowed tool via its secret ID; forbidden secrets and cross-user access rejected; expiry/logout handled; credential-leakage checks pass |
| 2: Read-only analysis | Jira, Bitbucket, artifact and knowledge retrieval; structured plan | Representative tickets produce traceable plans and identify ambiguous builds/requirements |
| 3: Executable vertical slice | One service, Sandbox, deployment/config change, SMTP input and evidence | Complete a real scenario and restore environment; recover from worker failure and cancellation |
| 4: SDET pilot | Reports, observability, failure investigation, regression cases | SDETs compare results against known outcomes; measure false passes, false failures, coverage and cleanup reliability |
| 5: Broad API completion | Remaining operation families, services, environments and provider adapters | Every agreed catalog operation implemented/tested or explicitly recorded as a pending scope exception |
| 6: Operational rollout | Monitoring, recovery drills, ownership, retention and maintenance process | Agreed quality targets met for eligible ticket classes; ongoing API drift detection in place |

Track false passes, false failures, blocked runs, requirement coverage, reproduction success, environment conflicts, cleanup failures, SDET review time, runtime and AI cost. Establish thresholds with the team before allowing automatic verification; do not use model confidence as the acceptance gate.

Test strategy: schema/contract tests for clients, real-instance integration tests, permission and isolation tests, workflow failure injection, end-to-end seeded scenarios, and historical tickets with known outcomes. Model-dependent evaluations use fixed snapshots and record model/prompt/tool versions.

Planning estimate, not a commitment: a focused vertical slice and pilot may take roughly 8–12 weeks with two backend engineers, one frontend engineer, and part-time SDET/platform support, assuming access and usable test environments. Full API coverage is a separately estimated program after Phase 0 produces operation counts and complexity. Do not equate pilot completion with full coverage.

## 10. Required internal inputs

- Installed versions/editions of Jira, Bitbucket, Artifactory, Grafana, Kubernetes and Helm; wiki product.
- Approved private endpoints, network/proxy/CA requirements and authentication methods. Tool secrets remain in Delinea; configure secret IDs and field mappings only. AD credentials are transient login inputs, not documentation or saved application configuration.
- Delinea deployment/version, authentication endpoint, AD domain/username conventions, MFA requirements, token expiry/refresh/revocation behavior, secret templates/field identifiers, and checkout/access policies. Confirm whether secrets hold personal or shared downstream accounts.
- CKP API specification, extensions/CRDs, configuration ownership and deployment procedures.
- NioSmtpClient CLI source/help and existing SMTP fixtures.
- Helm charts, artifact naming, commit/build mapping and CI identity; CI system is currently unspecified.
- Initial service/library docs, observability backend identities, and representative tickets with SDET expected outcomes.
- Corporate model/data handling policy and provider credentials supported for application use.

These details refine implementation; the architecture, data contracts, documentation templates and API catalog structure can be prepared before access is available.

## 11. Official documentation baseline

Consulted 20 October 2026. Pin the matching installed versions during discovery; these links are not proof that the private installations support the latest API.

- Jira Data Center REST: https://developer.atlassian.com/server/jira/platform/rest-apis/
- Bitbucket Data Center REST: https://developer.atlassian.com/server/bitbucket/rest/v1000/api-group-project/
- JFrog API/specification entry point: https://docs.jfrog.com/integrations/docs/jfrog-api
- Grafana HTTP API: https://grafana.com/docs/grafana/latest/developer-resources/api-reference/http-api/
- Confluence Data Center REST, only if the wiki is Confluence: https://developer.atlassian.com/server/confluence/rest/
- Kubernetes API overview/discovery: https://kubernetes.io/docs/reference/using-api/
- Helm command reference: https://helm.sh/docs/helm/
- Delinea Secret Server token authentication (version-specific example; match installed version): https://docs.delinea.com/online-help/secret-server-11-7-x/api-scripting/authentication/script-token-auth/index.htm
- Delinea-maintained SDK with domain-password authorization examples (reference for behavior, not a requirement to use Python): https://github.com/DelineaXPM/python-tss-sdk

The integration-family inventory above is a planning baseline, not an exhaustive endpoint enumeration. That enumeration requires installed versions and internal extensions. No private APIs or environments have been inspected or validated in preparing this plan.
