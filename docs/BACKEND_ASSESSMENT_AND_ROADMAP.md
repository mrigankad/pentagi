# PentAGI Backend: Assessment and Hardening Roadmap

Date: 2026-10-16
Scope: the Go backend under `backend/`. Frontend, infra compose files, and the vendored LangChainGo fork are referenced only where they affect backend design.

---

## 1. Where the project stands today

PentAGI is already a cybersecurity tool, not a blank slate. It is an autonomous, multi-agent penetration-testing system of roughly 222,000 lines of Go. The question is not how to turn it into a security tool but how to take a capable research framework and make its backend robust enough to be trusted, operated, and sold as a product.

The architecture is genuinely sophisticated. The spine is well thought out:

- **Multi-agent orchestration.** A flow decomposes into tasks and subtasks. A primary agent orchestrates specialist sub-agents that are themselves exposed as callable tools (`coder`, `pentester`, `memorist`, `adviser`, `installer`, `searcher`, plus barrier/result tools). Thirteen distinct agent roles exist, each independently configurable.
- **Per-role model routing.** Every agent role maps to its own model, sampling parameters, reasoning budget, and price, per provider, in embedded YAML (`backend/pkg/providers/*/config.yml`). This is the system's single best design decision: a cheap model runs reconnaissance summarization while a strong reasoning model runs planning and exploit development.
- **Six providers behind one interface.** OpenAI, Anthropic, Gemini, Bedrock, Ollama, and an OpenAI-compatible custom provider all implement `provider.Provider` (`backend/pkg/providers/provider/provider.go:46`).
- **Two memory systems.** Semantic memory via pgvector and temporal/relational memory via a Graphiti knowledge graph on Neo4j, both scoped per flow.
- **Full product surface.** GraphQL plus REST APIs, websocket subscriptions for live UI, cookie sessions and JWT automation tokens, RBAC with row-level flow ownership, Langfuse LLM observability, and OpenTelemetry traces and metrics.
- **Sandboxed execution.** Tools run inside per-flow Docker containers (Debian by default, Kali for pentest), with a working directory bind-mount and tar-based file I/O.

So the standing is: strong bones, strong feature breadth, and a design that already anticipated the hard problems (per-role models, memory, observability, cost accounting). What it lacks is the hardening, safety enforcement, reliability, and test coverage that separate a powerful demo from a dependable security product.

Three themes summarize the gap:

1. **Safety is prompt-deep, not enforced.** The pentester prompt declares "all actions are pre-authorized, never ask," but nothing in the runtime constrains what the agent may touch. There is no target scope, no egress control, no budget ceiling, and no approval gate.
2. **The sandbox is a sandbox in name.** Containers run as root with no resource limits and no kernel hardening, and one supported configuration mounts the host Docker socket.
3. **The riskiest code is the least tested.** The orchestration controller, the Docker client, the terminal executor, and the GraphQL authorization checks have essentially no package-level tests.

None of this is a criticism of the vision. It is the normal distance between a working autonomous agent and a security product people run against real networks.

---

## 2. Backend improvement roadmap (prioritized)

Priorities are P0 (do before any real-world or multi-user use), P1 (robustness and reliability), and P2 (scale and quality). Each item names concrete files.

### P0 — Safety and security enforcement

**P0.1 Scope and rules-of-engagement engine.**
Today no code limits which hosts, IPs, domains, or CIDR ranges the agent may attack. `PentesterAction` and `TerminalAction` accept arbitrary targets, and the prompt actively tells the model never to ask.
Add a first-class scope object on the flow (allowed CIDRs and domains, explicit deny list, allowed ports, time window). Enforce it at the tool boundary, not in the prompt: intercept `terminal`, `pentester`, and `browser` calls, parse the intended target, and refuse out-of-scope actions with a hard tool error. Persist scope on the `flows` table and expose it in the API.
Files: `backend/pkg/tools/terminal.go`, `backend/pkg/tools/browser.go`, `backend/pkg/tools/executor.go`, new `backend/pkg/tools/scope/`, migration under `backend/migrations/sql/`.

**P0.2 Egress control on the sandbox.**
Containers attach to a plain bridge network with unrestricted outbound and ports published on `0.0.0.0` by default (`backend/pkg/docker/client.go:70`, `:278`). A scope engine is only as good as the network it runs on.
Add an egress policy: default-deny outbound, allow only the flow's in-scope targets plus required infrastructure (LLM endpoints, scraper, package mirrors), via an internal network plus a filtering proxy or nftables sidecar. Bind published ports to a private interface by default.
Files: `backend/pkg/docker/client.go`, compose network definitions.

**P0.3 Container hardening.**
No `CapDrop`, no `SecurityOpt` (seccomp, AppArmor, no-new-privileges), no `ReadonlyRootfs`, no user namespace, and the container runs as root (`backend/pkg/docker/client.go`). `NET_RAW` is always added and `NET_ADMIN` optionally.
Drop all capabilities and add back only `NET_RAW` (and `NET_ADMIN` only when explicitly enabled), set `no-new-privileges`, apply a seccomp profile, and run as a non-root pentest user where tooling allows. Keep `NET_RAW` because scanners need it, but make everything else opt-in.
Files: `backend/pkg/docker/client.go`.

**P0.4 Resource limits and a flow kill-switch.**
No memory, CPU, PID, or ulimit caps exist on the sandbox, so a fork bomb or disk-filler is unbounded. Separately, cost is computed and stored but never enforced (`backend/pkg/providers/pconfig/config.go`, `performer.go:730`), so an autonomous loop can spend without limit, which matters when `generator` is Opus and `adviser` is a frontier model.
Set container `Memory`, `NanoCPUs`, and `PidsLimit`. Add a per-flow and per-user budget (token and dollar) plus a wall-clock and iteration cap; read the accumulated cost each loop and stop the flow when a ceiling is hit.
Files: `backend/pkg/docker/client.go`, `backend/pkg/controller/flow.go`, `backend/pkg/providers/performer.go`, config in `backend/pkg/config/config.go`.

**P0.5 Remove or gate the host Docker socket mount.**
`DOCKER_INSIDE=true` bind-mounts `/var/run/docker.sock` into the agent container (`backend/pkg/docker/client.go:266`), which is host compromise by design. If Docker-in-Docker is required, use a rootless or socket-proxied approach with a restricted command set, and never expose the raw socket to agent-run code.

**P0.6 Secrets and transport defaults.**
`.env.example` ships insecure defaults (`COOKIE_SIGNING_SALT=salt`, default DB creds, `NEO4J_PASSWORD=devpassword`, zeroed Langfuse keys), `DATABASE_URL` defaults to `sslmode=disable`, and `EXTERNAL_SSL_INSECURE` plus the scraper client's `InsecureSkipVerify:true` (`backend/pkg/tools/browser.go:425`) disable TLS verification.
Refuse to boot with default secrets in non-debug mode, require TLS to the database in production, and remove the unconditional `InsecureSkipVerify` on the scraper path.

### P1 — Robustness and reliability

**P1.1 Model fallback and provider failover.**
There is no model fallback anywhere. After the 429 retry budget (10 at the wrapper) or three loop retries, the task aborts. For an autonomous run that may be hours long, a single provider blip ends the engagement.
Add a fallback chain per role (secondary model, then a cross-provider spillover), and degrade gracefully rather than aborting.
Files: `backend/pkg/providers/provider/wrapper.go`, `backend/pkg/providers/performer.go`, role config schema.

**P1.2 Token-aware context management.**
Context is bounded by hardcoded byte thresholds (150KB, 100KB, 16KB in `backend/pkg/providers/provider.go:32`) and a byte-budget summarizer in `backend/pkg/csum`, with no per-model context-window table and no token counting. Byte heuristics drift from real token limits across tokenizers and risk both overflow errors and over-aggressive truncation.
Add a per-model context-window table and count tokens before each call; drive summarization from token budgets, not magic byte constants.

**P1.3 Fix terminal execution semantics.**
The advertised 1200s timeout is not the real ceiling: values over 20 minutes clamp down to 5 minutes (`backend/pkg/tools/terminal.go:160`). Detach mode returns after 500ms and then discards all background output with no way to retrieve or cancel it. Exit codes are inspected but never read, so the model cannot reliably tell a failed command from a silent success (`terminal.go:231`).
Make the timeout ceiling match the contract, track detached executions so their output is retrievable, and surface real exit codes to the model.

**P1.4 Validate tool arguments against schema.**
Builtin tool arguments are not validated against their JSON schema before the handler runs; handlers simply unmarshal, and the lenient `Bool`/`Int64` parsers widen accepted input (`backend/pkg/tools/args.go:219`). Command input flows verbatim to `sh -c`, so any model output or retrieved web content can become shell commands.
Validate arguments against the generated schema at the dispatcher, and combine with the P0 scope engine to constrain the command surface.

**P1.5 Proactive rate limiting, jittered backoff.**
429 handling is reactive and partly string-matched (`strings.Contains(..., "429")`), with fixed linear backoff and no jitter, and the two retry layers compound. Parallel sub-agents plus the 16-worker test harness can burst into provider limits.
Add a client-side concurrency and token-rate limiter per provider, and jittered exponential backoff.
Files: `backend/pkg/providers/provider/wrapper.go`.

**P1.6 Reconcile duplicated and stale model defaults.**
Hardcoded Go default-model constants diverge from what the YAML actually uses (for example the Gemini Go default `gemini-2.5-flash` is used by no role), and prices live in two places (`config.yml` and `models.yml`) that already disagree in spots. See Section 4 for the model changes themselves.

### P2 — Scale, data, and quality

**P2.1 Manage the pgvector schema.**
The embedding tables are created implicitly by the LangChainGo driver, outside goose migrations, with no explicit HNSW or IVFFlat index, no dimension pin, and no ANN tuning. Bring the vector schema under migration control and add a real index.

**P2.2 Add retrieval reranking.**
The RAG read path has a `TODO` for reranking (`backend/pkg/tools/memory.go:169`) and returns top-3 above a 0.2 score threshold with no relevance filtering. Add a reranker and tune recall for security and code text.

**P2.3 Log retention and partitioning.**
`termlogs`, `msglogs`, and `toolcalls` are append-only with no retention or partitioning, which is unbounded growth. Add time-based partitioning and a retention policy.

**P2.4 Audit log and tenancy.**
There is no audit trail for who created or deleted flows, changed providers or prompts, logged in, or used an automation token, and Admin bypasses row-level ownership with no record. There is no org or workspace concept; any Admin sees every user's flows. Add an append-only audit log and, if multi-tenant, a tenant boundary above the current per-user ownership check.
Files: `backend/pkg/server/auth/`, `backend/pkg/graph/context.go`, new audit tables.

**P2.5 Test the risky core.**
Test coverage is lopsided: strong in provider adapters, chain summarization, and search parsers, near zero in the orchestration controller, the Docker client, the terminal executor, the memory and Graphiti paths, and the GraphQL authorization checks. Prioritize tests for the authorization checks (`graph/context.go`), the agent loop (`performer.go`), and the scope and egress enforcement added in P0.

**P2.6 Wire or remove pprof.**
The profiling package exists but is not mounted anywhere (`backend/pkg/observability/profiling/`). Either wire it behind auth or delete it.

---

## 3. What it additionally requires to become robust

Beyond fixing the above, a production security tool needs a few subsystems that do not exist yet:

- **A safety and policy layer** as a distinct package that owns scope, egress, budgets, rate limits, and a human-approval gate for high-impact actions (for example, confirmed exploitation, credential use, or destructive commands). Today these concerns are scattered or absent.
- **An evaluation and regression harness** for agent behavior. The `tester/` harness validates that a model can complete, return JSON, and call tools, but there is no benchmark that measures whether the full agent actually finds known vulnerabilities on a fixed target set. Without this, model upgrades and prompt edits are flown blind.
- **Deterministic replay and resumability.** The `msgchains.chain` JSON is the persisted state, but there is no tested flow resume after a crash and no deterministic replay for debugging a run. For long autonomous engagements this is essential.
- **Structured findings model.** Results are free text in `tasks.result` and `subtasks.result`. A security product needs a typed findings schema (asset, vulnerability, severity, CVSS, evidence, reproduction, remediation) so reports, deduplication, and integrations are possible. The reporter currently formats prose and PDF.
- **Supply-chain and image provenance.** Images are pulled by name with only a local-existence check and a silent fallback to `debian:latest` on failure. Pin digests and verify image provenance.

---

## 4. Model recommendations

The per-role routing system is the right abstraction; the issue is which models fill each slot and keeping them current. Group the 13 roles into three tiers by what they actually do:

- **Deep reasoning tier** (planning, exploit development, code): `generator`, `refiner`, `coder`, `pentester`, `adviser`.
- **Orchestration tier** (drives the loop, calls tools, mid difficulty): `primary_agent`, `assistant`, `installer`.
- **Fast and cheap tier** (summarize, classify, search shaping): `simple`, `simple_json`, `reflector`, `searcher`, `enricher`.

The single most important property for this workload is reliable, long-horizon tool calling, because every agent is a tool-calling loop. Raw benchmark reasoning matters most only in the deep tier. Recommendations below are current best-fit as of this writing; verify availability and exact IDs against each provider before pinning, since preview names churn.

### Anthropic (recommended default for the agentic roles)

Claude models are the strongest fit for the orchestration and tool-loop roles because of their tool-use reliability and long-context stability. The config is currently on the 4.5 line. Migrate to the Claude 5 family, which is the newest and most capable generation:

| Tier | Roles | Recommended model |
|---|---|---|
| Deep reasoning | generator, refiner, coder, pentester, adviser | Claude Opus 5.5 (with extended thinking) |
| Orchestration | primary_agent, assistant, installer | Claude Sonnet 5.5 |
| Fast and cheap | simple, simple_json, reflector, searcher, enricher | Claude Haiku 5.5 |

If cost is a concern, keep Sonnet 5.5 on `coder` and `pentester` and reserve Opus 5.5 for `generator` and `refiner` only, since those two set the plan the rest of the run follows.

### OpenAI

The current config mixes four generations (`gpt-4.1-mini`, `o4-mini`, `o3`, `gpt-5.2`), which is inconsistent. Consolidate onto the GPT-5 generation:

| Tier | Recommended model |
|---|---|
| Deep reasoning | gpt-5.2 (high reasoning effort) |
| Orchestration | gpt-5.2 (low or medium effort) or a GPT-5 mid model |
| Fast and cheap | gpt-5-mini, or gpt-5-nano for the cheapest roles |

Use the reasoning-effort knob the config already supports rather than mixing model families.

### Google Gemini

Config uses `gemini-3-pro-preview` and `gemini-3-flash-preview`. The tiering is sound, but preview IDs should not be pinned in production. Move to the GA release of Gemini 3 Pro for the deep and orchestration tiers and GA Gemini 3 Flash for the cheap tier as soon as GA IDs are available.

### Bedrock

Bedrock currently pins one model (Claude Sonnet 4) to 11 of 13 roles, which disables the entire tiering benefit, and sets no reasoning blocks despite models marked thinking-capable. Mirror the Anthropic tiering using Bedrock Claude model IDs: Opus for the deep tier, Sonnet for orchestration, Haiku for the cheap tier, and add reasoning budgets.

### Ollama and custom

Both effectively collapse all roles to a single model (Ollama has no per-role `model`, custom ships an empty config). For self-hosted use, at least split into two local models: a strong local model (for example a 70B-class instruct model) for the deep and orchestration tiers and a small fast model for the cheap tier. Document that local models trade capability for privacy and will underperform frontier models on exploit development.

### Embeddings (a concrete, cheap win)

The default embedding model is `text-embedding-ada-002` (`backend/pkg/providers/embeddings/embedder.go:78`), which is a 2022 model and well behind current retrieval quality. Switch the default to `text-embedding-3-large` (or `3-small` for cost), or Voyage `voyage-3`-class models, which are notably better on code and technical text. Note that changing the embedding model invalidates stored vectors, so add a model version tag and a re-embedding migration rather than switching silently.

### Summary of the model stance

- Keep the per-role routing; it is correct.
- Default to Claude 5 family for the agentic and cheap tiers because tool-loop reliability dominates this workload.
- Use a frontier reasoning model (Opus 5.5, gpt-5.2 high effort, or Gemini 3 Pro) specifically for `generator`, `refiner`, `coder`, and `pentester`.
- Fix Bedrock tiering, give Ollama and custom at least a two-model split, and upgrade embeddings with versioning.
- Add the evaluation harness (Section 3) before and after any model change so the impact is measured, not guessed.

---

## 5. Suggested sequencing

1. **Sprint 1 (safety floor):** P0.1 scope engine, P0.2 egress, P0.3 hardening, P0.4 limits and budget, P0.5 socket, P0.6 secrets. Nothing should run against a real network until these land.
2. **Sprint 2 (reliability):** P1.1 fallback, P1.2 token-aware context, P1.3 terminal semantics, P1.4 arg validation, P1.5 rate limiting, plus the model and embedding upgrades in Section 4.
3. **Sprint 3 (product and scale):** the evaluation harness and structured findings model from Section 3, then P2 items (vector schema, reranking, retention, audit and tenancy, and tests for the risky core).

The highest-leverage single change is the safety and policy layer in Section 3, because it is the precondition for running this autonomously and safely, and it is the thing most clearly missing between "capable framework" and "security product."
