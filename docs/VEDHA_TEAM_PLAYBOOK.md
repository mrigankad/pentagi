# Vedha: Team Playbook and Claude Code Prompts

Date: 2026-10-16
Project: Vedha (formerly PentAGI in this codebase)
Companion to `docs/BACKEND_ASSESSMENT_AND_ROADMAP.md` and `docs/PRODUCT_STRATEGY_AND_AZURE_PLAN.md`.

This playbook turns the roadmap into assignments. It is written for teams who work through Claude Code. Each team has a self-contained section: what to build, copy-paste Claude Code prompts, how to verify the result is correct, and the extra, non-coding steps they must do. A team lead can copy their one section and run with it.

Read Section 0 first, every team, before touching code.

---

## Section 0: Shared setup (every team does this once)

### 0.1 Conventions

- **Repo:** the existing Go backend lives in `backend/`. Vedha is the product name; the Go module stays `pentagi` for now unless Team Platform renames it in a planned migration.
- **Branches:** one feature branch per task, named `vedha/<team>/<short-task>`, for example `vedha/safety/scope-engine`. Never commit to the default branch directly.
- **Commits:** small, descriptive, imperative mood. One logical change per commit.
- **Pull requests:** open a PR per task against the integration branch. Keep PRs under about 600 changed lines where possible; split if larger.
- **Definition of done, for every task:** code compiles, `go vet` clean, new unit tests pass, the existing suite still passes, the feature is behind a config flag if it changes runtime behavior, and the change is documented in the PR body with how it was verified.

### 0.2 Baseline commands (know these before prompting Claude)

Run these from `backend/`:

```
go build ./...            # must succeed
go vet ./...              # must be clean
go test ./...             # existing + new tests
gofmt -l .                # must print nothing (all files formatted)
```

If the repo adds a linter (Team QA owns this), also run `golangci-lint run`.

### 0.3 The house-rules preamble (prepend to the first prompt of any session)

Every team pastes this block at the start of a Claude Code session so the agent shares context:

```
You are working on Vedha, an autonomous security-testing platform (Go backend
in backend/, module name "pentagi"). Read docs/BACKEND_ASSESSMENT_AND_ROADMAP.md
and docs/PRODUCT_STRATEGY_AND_AZURE_PLAN.md for context before changing code.

Rules for this codebase:
- Match existing patterns. Study a sibling file before writing a new one.
- Any change to runtime behavior goes behind a config flag in
  backend/pkg/config/config.go with a safe default (off, or the current behavior).
- Add unit tests for new logic. Do not weaken or skip existing tests.
- Keep changes minimal and in scope. Do not refactor unrelated code.
- After changing code, run: go build ./... && go vet ./... && go test ./...
  and gofmt. Report the actual output, do not claim success without running them.
- Never hardcode secrets. Read config from env via the existing config struct.
- Tell me your plan before editing, then implement.
```

### 0.4 Extra setup steps (every team, once)

1. Confirm you can build and test the backend locally with the commands above.
2. Create a root `CLAUDE.md` if the repo does not have one yet (Team QA owns the canonical version; see Appendix A). It gives every Claude Code session the shared rules automatically.
3. Agree the integration branch name with the other leads.
4. Agree the config-flag naming convention (suggested prefix `VEDHA_`).

---

## Team 1: Safety and Policy

**Mission.** Build the enforcement layer that makes Vedha safe to run autonomously. This is P0 and blocks real-world use. It is also the top product differentiator.

**What to build.**
- A scope engine: a per-flow policy object (allowed CIDRs, domains, ports, time window, deny list) enforced at the tool boundary for `terminal`, `pentester`, and `browser`.
- Budget and kill-switch: per-flow and per-user caps on tokens, dollars, wall-clock, and iterations, checked each loop, stopping the flow when exceeded.
- A human-approval gate for high-impact actions, reusing the existing `ask` tool mechanism.

**Where in the code.** `backend/pkg/tools/executor.go` (dispatch boundary), `backend/pkg/tools/terminal.go`, `backend/pkg/tools/browser.go`, `backend/pkg/controller/flow.go`, `backend/pkg/providers/performer.go` (loop and cost), `backend/pkg/config/config.go`, new package `backend/pkg/tools/scope/`, a migration under `backend/migrations/sql/` for the scope column on `flows`.

**Claude Code prompts.**

Prompt 1, scope engine core:
```
Create a new package backend/pkg/tools/scope that defines a Scope type with
allowed CIDRs, allowed domains (with wildcard support), allowed ports, a deny
list, and an optional time window. Add a method Check(target string) that
returns allow/deny with a reason, where target can be an IP, host:port, or URL.
Write thorough table-driven unit tests covering IPv4/IPv6 CIDR membership,
wildcard domain matching, deny-list precedence over allow, port rules, and
malformed input. Do not wire it into tools yet. Run the tests and show output.
```

Prompt 2, enforce at the tool boundary:
```
Wire the scope.Scope check into tool dispatch. In backend/pkg/tools, before the
terminal, pentester, and browser handlers execute, extract the intended target
(for terminal, parse the command for host/IP/URL arguments conservatively; when
no target can be determined, fall back to the flow's default policy) and call
Check. On deny, return a hard tool error string to the model explaining the
target is out of scope, and do not execute. Put this behind a config flag
VEDHA_SCOPE_ENFORCEMENT (default off so existing flows are unchanged). Add tests
that a denied target never reaches the handler. Run go build, go vet, go test.
```

Prompt 3, budgets and kill-switch:
```
Add per-flow budget enforcement. Extend config with VEDHA_MAX_FLOW_COST_USD,
VEDHA_MAX_FLOW_ITERATIONS, and VEDHA_MAX_FLOW_WALLCLOCK_MINUTES (all default 0 =
unlimited). In the agent loop (backend/pkg/providers/performer.go) and the flow
controller, accumulate cost and iteration count (cost is already computed per
message chain) and stop the flow with a clear terminal status when a non-zero
cap is exceeded. Add tests for the stop condition. Show the loop change and test
output.
```

**How to verify it is correct.**
- `go test ./pkg/tools/scope/...` passes with the table-driven cases.
- With `VEDHA_SCOPE_ENFORCEMENT=true` and a scope of a single host, a flow asked to scan a different host gets a denial string and the handler is never called (assert via a test double).
- With a tiny `VEDHA_MAX_FLOW_ITERATIONS`, a flow stops at the cap.
- With the flags at their defaults, existing behavior and the existing suite are unchanged.

**Extra things this team must do.**
- Write the migration that adds a `scope` JSON column to `flows`, and extend the GraphQL and REST create-flow inputs so a scope can be supplied (coordinate with Team Platform on the schema).
- Document the policy format in `docs/`.
- Define what counts as a "high-impact action" requiring approval, with security input.
- Threat-model the target-parsing step: an attacker-influenced command must not evade the check by obfuscation. Prefer deny-on-uncertainty.

---

## Team 2: Sandbox, Execution, and Azure Infrastructure

**Mission.** Make the execution sandbox actually isolated, fix the terminal tool bugs, and own the Azure infrastructure-as-code.

**What to build.**
- Container hardening: drop capabilities, add back only `NET_RAW` (and `NET_ADMIN` only when enabled), set `no-new-privileges`, seccomp, read-only rootfs where feasible, non-root user, and resource limits (memory, CPU, PID).
- Remove or safely proxy the host Docker socket mount.
- Terminal fixes: make the timeout ceiling honest, track detached executions so output is retrievable and cancellable, surface real exit codes.
- Azure path: Bicep or Terraform to deploy the stack, and an adapter so tool execution can target Azure Container Apps Dynamic Sessions or Sandboxes instead of raw Docker.

**Where in the code.** `backend/pkg/docker/client.go`, `backend/pkg/tools/terminal.go`, `backend/pkg/tools/tools.go`, new `deploy/` directory for IaC.

**Claude Code prompts.**

Prompt 1, hardening and limits:
```
In backend/pkg/docker/client.go, harden container creation. Set HostConfig
resource limits from new config values (VEDHA_SANDBOX_MEMORY_MB,
VEDHA_SANDBOX_NANO_CPUS, VEDHA_SANDBOX_PIDS_LIMIT). Drop all Linux capabilities
and add back NET_RAW always and NET_ADMIN only when DOCKER_NET_ADMIN is true. Add
SecurityOpt no-new-privileges. Keep everything behind sensible defaults that do
not break the current pentest toolchain. Do not change networking yet. Explain
each HostConfig field you set and why. Run go build and go vet.
```

Prompt 2, terminal correctness:
```
Fix backend/pkg/tools/terminal.go. (1) Make the timeout honor the documented
1200s ceiling instead of silently clamping values over 20 minutes down to 5.
(2) For detach mode, keep a registry of background executions keyed by an id so
their output can be retrieved later and they can be cancelled, instead of
discarding output after 500ms. (3) Read the exec exit code from
ContainerExecInspect and include it in the result so the model can distinguish
failure from silent success. Add unit tests for the timeout clamp logic and exit
-code reporting using a fake exec interface. Run go test ./pkg/tools/...
```

Prompt 3, Azure IaC (separate session, infra not Go):
```
Create a deploy/ directory with Bicep modules to stand up the Vedha stack in an
Azure tenant: AKS or Container Apps for the backend, Azure Container Apps
dynamic session pool for the sandbox, Azure Database for PostgreSQL Flexible
Server with the pgvector extension, Azure AI Search, Key Vault, Entra ID app
registration placeholders, VNet with NSG and Azure Firewall for egress, and
Azure Front Door with WAF. Parameterize everything and write a README with the
deploy commands. Do not include any real secrets. Validate the Bicep with
`az bicep build`.
```

**How to verify it is correct.**
- A container started with the new code runs the existing Kali tools (nmap SYN scan works because `NET_RAW` is retained), but `capsh --print` inside shows a reduced capability set.
- A fork bomb or memory hog inside the sandbox is bounded by the limits.
- A command with a 30-minute timeout is allowed to run that long; a detached command's output can be fetched afterward; a failing command reports a non-zero exit code.
- `az bicep build` succeeds on the IaC with no errors.

**Extra things this team must do.**
- Coordinate egress enforcement with Team 1: the scope allowlist must be reflected in the Azure Firewall rules and the NSG, so scope is enforced at the network, not only in code.
- Decide the Docker-in-Docker policy: remove it, or replace with a socket proxy with a restricted command set. Document the decision.
- Write a runbook for operating the sandbox fleet.

---

## Team 3: Providers, Models, and Reliability

**Mission.** Upgrade the models, add fallback, make context management token-aware, and add proper rate limiting.

**What to build.**
- Updated per-role model configs across all providers (see the model section of the assessment doc).
- Model fallback chains and cross-provider failover.
- A per-model context-window table and token counting that drives summarization.
- Client-side concurrency and token-rate limiting with jittered backoff.
- Embedding model upgrade with vector versioning.

**Where in the code.** `backend/pkg/providers/*/config.yml`, `backend/pkg/providers/*/models.yml`, `backend/pkg/providers/provider/wrapper.go`, `backend/pkg/providers/performer.go`, `backend/pkg/providers/pconfig/config.go`, `backend/pkg/csum/`, `backend/pkg/providers/embeddings/embedder.go`.

**Claude Code prompts.**

Prompt 1, model config update:
```
Update the per-role model assignments. Goal: each role uses the current best-fit
model for its tier (deep-reasoning roles generator/refiner/coder/pentester/adviser
on a frontier reasoning model; orchestration roles primary_agent/assistant/
installer on a mid model; cheap roles simple/searcher/enricher/reflector on a
fast model). Apply this to backend/pkg/providers/anthropic/config.yml,
openai/config.yml, gemini/config.yml, and bedrock/config.yml. For Bedrock,
introduce real tiering instead of one model for every role, and add reasoning
blocks for thinking-capable models. Reconcile the hardcoded default-model
constants in each provider's .go file with the YAML. List every model ID you set
and flag any you are unsure still exists so I can verify it against the provider.
```

Prompt 2, fallback:
```
Add a fallback mechanism to the provider layer. Extend the role config schema to
allow an ordered list of fallback models (and optionally a fallback provider).
In the call path, when a call fails after its retry budget with a retryable or
capacity error, try the next fallback before aborting the task. Add tests with a
fake LLM that fails the primary and succeeds on the fallback. Keep it behind
VEDHA_MODEL_FALLBACK (default off). Run go test ./pkg/providers/...
```

Prompt 3, token-aware context:
```
Replace the hardcoded byte-size context limits with token-aware management. Add a
per-model context-window table (tokens) and a token-counting helper. Drive the
csum summarizer and the prompt size caps from token budgets derived from the
active model's window, not the magic byte constants in provider.go. Preserve
current behavior when the model is unknown by falling back to the old byte caps.
Add tests. Show the before/after of the size-limit logic.
```

Prompt 4, embeddings:
```
Change the default embedding model from text-embedding-ada-002 to a current
model (text-embedding-3-large), configurable via EMBEDDING_MODEL. Add a stored
embedding-model version tag on vector records and a guard that refuses to mix
vectors from different embedding models in one similarity search, with a clear
error pointing to a re-embedding step. Do not silently switch. Add tests for the
version guard.
```

**How to verify it is correct.**
- Each provider's config loads and validates (`go test ./pkg/providers/...`), and `testProvider`/`testAgent` pass for the new models with real keys in a staging run.
- With fallback on and a fake primary that returns a 429 past its budget, the task completes on the fallback model.
- The token-aware path keeps a long conversation under the model's real window and does not overflow.
- Switching embedding models surfaces the version guard instead of returning garbage matches.

**Extra things this team must do.**
- Verify every model ID against the live provider before merging; preview IDs churn. Do not pin preview names in production configs.
- Own the single source of truth for prices: reconcile `config.yml` and `models.yml` so cost accounting is correct, and document how prices get updated.
- Coordinate the re-embedding migration with Team 4.

---

## Team 4: Memory, Data, and Findings Model

**Mission.** Bring the vector schema under control, add reranking, build the typed findings model and the attack-path graph, and add retention.

**What to build.**
- Managed pgvector schema (migrations, explicit index, dimension pin).
- A reranking step in the RAG read path, optionally via Azure AI Search semantic ranker.
- The typed findings and evidence model (the product-critical one).
- The unified attack-path graph schema on top of Graphiti or Neo4j.
- Retention and partitioning for the high-volume log tables.

**Where in the code.** `backend/migrations/sql/`, `backend/sqlc/models/`, `backend/pkg/tools/memory.go`, `backend/pkg/tools/search.go`, `backend/pkg/graphiti/`, new findings package.

**Claude Code prompts.**

Prompt 1, findings model (do this first, everything depends on it):
```
Design and implement a typed findings model. Create a migration adding a
findings table (asset, title, vulnerability class, severity, CVSS vector,
confidence, status, reproduction steps, evidence refs, remediation, flow/task/
subtask refs, timestamps) and an evidence table (type, blob ref, metadata). Add
sqlc queries and the Go model, following the existing sqlc patterns in
backend/sqlc and backend/pkg/database. Add a code path for agents to record a
finding via a new barrier tool "report_finding" with a validated schema. Do not
remove the existing free-text result fields yet. Add tests for the queries.
```

Prompt 2, vector schema:
```
Bring the pgvector schema under migration control. Add a goose migration that
creates the embedding table explicitly with a pinned vector dimension matching
the configured embedding model, and an HNSW index with sensible parameters.
Ensure the application uses this managed table rather than the implicitly driver
-created one. Document the dimension/model coupling. Verify the migration applies
cleanly up and down.
```

Prompt 3, reranking:
```
Add a reranking step to backend/pkg/tools/memory.go after the similarity search,
behind VEDHA_RERANK (default off). Provide two implementations behind an
interface: a local cross-encoder-style score and an Azure AI Search semantic
ranker client. Rerank the top-N candidates and return the best M. Remove the
TODO about missing reranking. Add tests with a fake reranker asserting reorder.
```

**How to verify it is correct.**
- Migrations apply up and down with no errors against a fresh Postgres.
- A finding recorded by the new tool round-trips through the DB with all typed fields intact and is retrievable via the API.
- With reranking on, a retrieval returns results in the reranker's order, not raw vector order.
- The vector index exists and similarity queries use it (check the query plan).

**Extra things this team must do.**
- Coordinate the embedding-model version tag with Team 3 and write the re-embedding migration.
- Define the findings taxonomy (vulnerability classes, severity scale) with security input.
- Decide the retention windows for `termlogs`, `msglogs`, `toolcalls` with the ops lead and implement partitioning.

---

## Team 5: Platform, API, Identity, and Tenancy

**Mission.** Make Vedha enterprise-ready: SSO, multi-tenant isolation, audit logging, and tested authorization.

**What to build.**
- Microsoft Entra ID (OIDC and SAML) auth alongside the existing providers.
- A tenant or workspace boundary above the current per-user ownership.
- An append-only audit log for security-relevant actions.
- Tests for the GraphQL authorization and ownership checks (currently untested).

**Where in the code.** `backend/pkg/server/auth/`, `backend/pkg/server/oauth/`, `backend/pkg/graph/context.go`, `backend/pkg/graph/schema.resolvers.go`, `backend/migrations/sql/`, new audit package.

**Claude Code prompts.**

Prompt 1, authorization tests first (lock in current behavior before changing it):
```
Write thorough tests for the existing authorization logic in
backend/pkg/graph/context.go (validatePermission, validatePermissionWithFlowID)
and backend/pkg/server/auth/permissions.go. Cover: a user cannot access another
user's flow, admin bypass works, the *.admin auto-escalation regex behaves, and
missing permissions are denied. Do not change the logic; just characterize it
with tests. Run go test ./pkg/graph/... ./pkg/server/auth/...
```

Prompt 2, Entra ID:
```
Add a Microsoft Entra ID OIDC provider alongside the existing Google and GitHub
OAuth in backend/pkg/server/oauth. Follow the existing oauth client pattern.
Support OIDC now; leave a clean seam for SAML. Map Entra groups to Vedha roles
via config. Gate behind config (OAUTH_ENTRA_* vars). Add tests for the callback
and claim mapping with a fake token. Do not break the existing providers.
```

Prompt 3, tenancy and audit:
```
Introduce a tenant/workspace concept. Add a tenants table and a tenant_id on
users and flows (migration), default everyone to a single tenant for backward
compatibility. Enforce tenant isolation in the GraphQL ownership checks so admins
see only their tenant's flows unless they are a super-admin. Separately, add an
append-only audit_log table and record flow create/delete, provider/prompt
changes, logins, and automation-token use. Add tests for tenant isolation and
that audited actions produce audit rows.
```

**How to verify it is correct.**
- The characterization tests pass against current code, then still pass (or are updated deliberately) after the tenancy change.
- An Entra ID login creates or links a user and assigns the mapped role.
- A user in tenant A cannot see tenant B's flows, asserted by test.
- Every audited action produces exactly one audit row with actor, action, and target.

**Extra things this team must do.**
- Coordinate the scope-on-flow schema with Team 1 and the findings schema with Team 4 so migrations do not collide; agree a migration ordering.
- Decide the super-admin model and document it.
- Plan the data migration for existing single-tenant installs.

---

## Team 6: Findings, Reporting, and Integrations

**Mission.** Turn typed findings into trusted outputs and connect Vedha to the tools customers already use.

**What to build.**
- Verified proof-of-exploit: attach the exact command chain and captured evidence to each finding, with a confidence score.
- Run-to-run diffing (new, fixed, still-open findings).
- Reports (PDF and structured JSON export) built from the typed findings model.
- Integrations: Jira, DefectDojo, Slack, and a generic webhook and SIEM export.

**Where in the code.** `backend/pkg/templates/prompts/reporter.tmpl` and the reporter path, the findings package from Team 4, new `backend/pkg/integrations/`.

**Claude Code prompts.**

Prompt 1, verified findings and reports:
```
Depends on Team 4's findings model. Update the reporter to build reports from the
typed findings table instead of free-text results. For each finding, include the
reproduction command chain and evidence refs, and a confidence score. Produce
both a structured JSON export and the existing PDF. Add a test that a report
generated from a set of fixture findings contains every finding with its
evidence. Keep the old reporter path available behind a flag during transition.
```

Prompt 2, diffing:
```
Add run-to-run finding diffing. Given two flows against the same target scope,
compute new, fixed, and still-open findings by a stable finding fingerprint
(asset + vuln class + location). Expose it via the API and include it in reports.
Add tests for the fingerprint stability and the diff categories.
```

Prompt 3, integrations:
```
Create backend/pkg/integrations with a common interface (PushFinding, PushReport)
and implementations for Jira, DefectDojo, Slack, and a generic signed webhook.
Each is configured per tenant and gated by config. Do not call external services
in tests; use httptest servers and assert the outbound payloads. Implement Jira
and the webhook first.
```

**How to verify it is correct.**
- A report built from fixture findings contains every finding, its evidence, and its confidence.
- Diffing two runs where one bug was fixed and one is new reports exactly one fixed and one new.
- Each integration posts the expected payload to an `httptest` server; no live calls in CI.

**Extra things this team must do.**
- Define the finding fingerprint with Team 4 so it is stable across runs.
- Write customer-facing docs for each integration (setup, field mapping).
- Get a security reviewer to confirm the "verified" label is only applied when exploitation actually succeeded, not merely suspected.

---

## Team 7: Evaluation, QA, and CI

**Mission.** Make quality measurable. Build the benchmark that every model and prompt change is measured against, and raise the test floor on the risky core.

**What to build.**
- An evaluation harness and a benchmark corpus of vulnerable targets with known findings.
- Metrics: detection rate, false-positive rate, cost, and time per engagement.
- Tests for the orchestration controller, the agent loop, the Docker client, and the memory path (the currently untested core).
- CI pipeline: build, vet, lint, test, and a nightly eval run.

**Where in the code.** New `backend/eval/` or `cmd/eval/`, `backend/pkg/controller/` (tests), `.github/workflows/`.

**Claude Code prompts.**

Prompt 1, eval harness:
```
Build an evaluation harness under cmd/eval. It runs Vedha against a configurable
set of target environments (start with a docker-compose of known-vulnerable apps
like a deliberately vulnerable web app), collects the findings Vedha reports,
compares them to a ground-truth file per target, and outputs detection rate,
false-positive rate, total cost, and wall-clock time as a JSON report. Make it
runnable locally and in CI. Start with one target and one ground-truth file.
```

Prompt 2, controller tests:
```
Add unit tests for the orchestration controller in backend/pkg/controller,
starting with the flow lifecycle (create, status transitions, stop, finish) and
the agent loop stop conditions. Use fakes for the provider and database
interfaces following any existing test doubles in the repo. Do not change
production logic; if something is untestable, note it for refactor rather than
changing behavior silently. Report coverage before and after.
```

Prompt 3, CI:
```
Create a GitHub Actions workflow that on every PR runs: go build ./..., go vet
./..., gofmt check, golangci-lint, and go test ./... with the race detector.
Add a separate nightly workflow that runs the eval harness against the benchmark
corpus and uploads the JSON report as an artifact. Cache the Go build. Make the
PR workflow required.
```

**How to verify it is correct.**
- The eval harness produces a JSON report with the four metrics against at least one real vulnerable target.
- Controller test coverage rises from near zero to a meaningful floor (agree a target, for example 50 percent of the flow lifecycle paths).
- The PR workflow fails on a deliberately broken build and passes on a clean one.

**Extra things this team must do.**
- Own the canonical `CLAUDE.md` (Appendix A) and keep it current as conventions evolve.
- Grow the benchmark corpus over time and publish the methodology (this is also a marketing asset per the strategy doc).
- Set the coverage and lint gates and enforce them in CI.

---

## Section 8: Sequencing and dependencies

Build order matters. Some teams block others.

1. **First, in parallel:** Team 1 (safety), Team 2 (sandbox hardening), Team 7 (CI and test scaffolding). Nothing runs against a real network until Team 1 and Team 2 land their P0 work.
2. **Team 4's findings model is a dependency** for Team 6 (reporting) and partly Team 7 (eval ground truth). Team 4 should ship the findings model early.
3. **Team 3** (models, fallback, context) can run in parallel throughout; coordinate the embedding change with Team 4.
4. **Team 5** (identity, tenancy, audit) can run in parallel but must agree migration ordering with Teams 1 and 4.
5. **Team 6** (reporting, integrations) starts once Team 4's findings model is merged.

Migration ordering is the main cross-team hazard: Teams 1, 4, and 5 all add migrations. Agree a numbering and merge order so they apply cleanly.

---

## Appendix A: canonical CLAUDE.md to drop in the repo root

```
# Vedha

Autonomous security-testing platform. Go backend in backend/ (module "pentagi").
Product name is Vedha.

## Build and test (run from backend/)
- go build ./...
- go vet ./...
- go test ./...
- gofmt -l .   (must be empty)

## Conventions
- Feature branches: vedha/<team>/<task>. Never commit to the default branch.
- Any runtime-behavior change goes behind a config flag (prefix VEDHA_) in
  backend/pkg/config/config.go with a safe default.
- Add unit tests for new logic. Never weaken or skip existing tests.
- No secrets in code or config files; read from env via the config struct.
- Match existing patterns; study a sibling file before adding a new one.
- Keep changes in scope; do not refactor unrelated code.

## Key docs
- docs/BACKEND_ASSESSMENT_AND_ROADMAP.md  (hardening roadmap)
- docs/PRODUCT_STRATEGY_AND_AZURE_PLAN.md (strategy and Azure architecture)
- docs/VEDHA_TEAM_PLAYBOOK.md             (team assignments and prompts)

## Architecture quick map
- backend/pkg/controller  flow/task/subtask orchestration and state
- backend/pkg/providers    LLM provider abstraction, per-role model routing
- backend/pkg/tools        agent tools (terminal, browser, search, memory, sub-agents)
- backend/pkg/docker       sandbox container lifecycle
- backend/pkg/graph        GraphQL API; server/ has REST and auth
- backend/pkg/database + sqlc + migrations  data layer
- backend/pkg/csum         chain summarization (context management)

## Definition of done
Code builds, vet clean, gofmt clean, new tests pass, existing tests pass,
behavior change is flagged, PR body states how it was verified.
```

## Appendix B: how a team runs a task with Claude Code (the loop)

1. Paste the house-rules preamble (Section 0.3) to start the session.
2. Paste the specific task prompt from your team's section.
3. Let Claude present its plan. Confirm or correct the plan before it edits.
4. After it implements, make it run `go build ./... && go vet ./... && go test ./...` and gofmt, and show the real output.
5. Check the result against your section's "how to verify it is correct" list.
6. Open a PR; CI (Team 7) must be green.
7. Get a review from another team member, merge to the integration branch.
