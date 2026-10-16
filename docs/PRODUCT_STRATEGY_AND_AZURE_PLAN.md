# PentAGI: Competitive Strategy, Build-From-Scratch Plan, and Azure Architecture

Date: 2026-10-16
Companion to `docs/BACKEND_ASSESSMENT_AND_ROADMAP.md`. That document covers hardening the current code. This one covers the market, how to win in it, what to build if you started clean, and how to run the whole thing on Azure.

A note on the research: the competitor figures below come from vendor-adjacent blogs and comparison pages, and the sources disagree on funding amounts, round types, and rankings. Treat specific numbers as approximate and verify against primary sources (company press releases, Crunchbase, the original benchmark papers) before using them in a board deck or a fundraise.

---

## 1. The competitive landscape

The autonomous and AI-assisted pentesting market got real in 2025 and 2026, with large raises and a billion-dollar leader. It splits along four axes. Knowing which quadrant each rival sits in is how you find the open ground.

### Axis 1: what kind of attacker the tool automates

- **Application and API attacker** (web app flaws, injection, auth bypass, business logic). Players: **XBOW**, **RunSybil**, **Terra Security**, **Mindfort**, **Strix** (open source), **Ethiack** (agent "Hackian"), **Simbian AI Pentest Agent**. XBOW is the reference product here and reportedly topped the HackerOne US leaderboard in 2025.
- **Network and infrastructure attacker** (foothold, credential abuse, lateral movement, Active Directory, cloud, Kubernetes). Players: **Horizon3.ai NodeZero**, **Pentera**. These are the mature enterprise exposure-validation platforms.

Almost every competitor is strong on one side and weak or absent on the other. XBOW itself is described as leaving network, cloud, Active Directory, and mobile to separate tooling.

### Axis 2: continuous vs point-in-time

Enterprise buyers increasingly want always-on validation that re-tests on every deploy, not a once-a-quarter scan. NodeZero and Pentera lean continuous. XBOW is criticized by rivals as point-in-time, though that is a competitor's framing, not an established fact.

### Axis 3: fully autonomous vs human-in-the-loop

Fully autonomous (XBOW, NodeZero) maximizes speed and coverage. Human-in-the-loop (Terra Security, Cobalt, the pentest-as-a-service crowd) trades speed for fewer false positives and more trust. A December 2025 Stanford benchmark (cited second-hand, unverified) reportedly found the best fully autonomous agent missed a critical remote-code-execution flaw that 80% of human testers caught. Whether or not that exact figure holds, the general point is real: fully autonomous agents still miss things and still hallucinate findings, which is the single biggest trust problem in the category.

### Axis 4: SaaS vs self-hosted and open source

Nearly all the funded players are SaaS. You point them at your systems and your attack data lives in their cloud. **PentAGI and Strix are the notable open-source, self-hostable exceptions.** This is a real and underexploited wedge.

### Where PentAGI sits today

PentAGI is open-source, self-hostable, and already has the pieces to do **both** the application attacker and the network attacker, because it drives a full Kali toolchain inside a sandbox and also has a browser and web tools. Most rivals pick one side; PentAGI's architecture spans both. It also already has multi-provider model routing, two memory systems, and an observability stack. What it lacks is the productization: safety enforcement, verified findings, continuity, and the trust artifacts enterprises require.

---

## 2. How to tackle them: the differentiation thesis

Do not try to beat XBOW at raw web-app bug-bounty throughput; they have a huge head start and a billion dollars. Win on the dimensions the funded SaaS players structurally cannot or will not match.

1. **Own "both attackers, one platform."** Unify application and network and cloud and Active Directory testing with shared memory and a shared attack-path graph, so a web foothold flows into lateral movement and domain compromise in one engagement. This is the thing XBOW explicitly does not do and NodeZero does not do from the app side. PentAGI's sandbox-plus-Kali plus browser design is already positioned for it.

2. **Win on trust, not just speed.** The category's weak point is false positives and missed findings. Make verified exploitation a first-class output: every finding carries reproducible proof (the exact command chain, captured evidence, a replayable session), a confidence score, and an optional human-approval gate before anything intrusive runs. Publish your own eval benchmark and your false-positive rate. Trust is how you sell to regulated buyers.

3. **Be the sovereign, self-hosted option.** Banks, defense, healthcare, and governments cannot send live exploitation data and internal network maps to a third-party SaaS. Open-core plus fully self-hostable (including air-gapped and local-model) is a wedge XBOW and Horizon3 cannot follow without abandoning their model. This is also why the Azure-native path in Section 4 matters: "runs entirely in your own Azure tenant" is a sales line by itself.

4. **Make it continuous and CI/CD-native.** Trigger engagements on deploy, diff findings against the last run, and integrate into the developer loop (pull-request checks, ticketing). Point-in-time is a disadvantage you can design past from day one.

5. **Lead on safety and governance.** A hard scope engine, egress control, full audit trail, and policy gates are not just hardening; they are enterprise purchase requirements and a marketing message. "It physically cannot act outside the authorized scope" is something no prompt-only competitor can claim.

6. **Keep the open-source community flywheel.** The open-source posture gives you contributors, auditability, and distribution that the closed players pay dearly for. Monetize with a managed and enterprise tier (SSO, multi-tenant, support, hosted sandboxes, premium integrations) on top of the open core.

The one-line positioning: **the open, self-hostable, safety-governed autonomous red team that chains the web attacker and the network attacker into one verified, continuous engagement.**

---

## 3. If you built PentAGI from scratch: the end-to-end plan

This assumes you keep the good ideas from the current codebase (per-role model routing, sub-agents-as-tools, dual memory, observability) and design in the things that are currently missing or bolted on.

### 3.1 Architectural pillars to add from day one

These are the structural decisions that are painful to retrofit, so build them in first:

- **A safety and policy layer as its own service.** Owns scope (allowed CIDRs, domains, ports, time windows), egress enforcement, budgets (token, dollar, wall-clock, iteration), rate limits, and the human-approval gate. Every tool call passes through it. Nothing executes outside it.
- **A typed findings and evidence model.** Not free text. Asset, vulnerability class, severity and CVSS, confidence, the exact reproduction chain, captured evidence, and remediation. Everything downstream (reports, dedup, diffing, integrations) depends on this existing early.
- **A unified attack-path graph.** One graph spanning assets, credentials, findings, and relationships across web and network and cloud, so the planner reasons about chains, not isolated bugs. This is the "both attackers, one platform" differentiator made concrete.
- **An evaluation harness and benchmark corpus.** A fixed set of vulnerable targets with known findings, run on every model and prompt change, measuring detection rate, false-positive rate, cost, and time. Without this you upgrade models blind.
- **Deterministic replay and resumable flows.** Persist agent state such that a run can be resumed after a crash and replayed deterministically for debugging. Essential for multi-hour engagements and for trust.
- **A clean provider abstraction with fallback.** Keep per-role routing, but design fallback chains and cross-provider failover in from the start rather than as a patch.

### 3.2 Feature set, by layer

- **Orchestration:** planner and specialist agents (recon, web exploit, network and AD, cloud, post-exploitation, reporter), sub-agents as tools, token-aware context management, loop and cost caps, resumable state.
- **Execution:** microVM-isolated per-engagement sandboxes, hardened and non-root, with managed egress control; a full offensive toolchain; a headless browser for web; detached long-running jobs that are trackable and cancellable.
- **Memory:** hybrid semantic search (vector plus keyword) with reranking, a temporal and relational attack graph, and cross-engagement institutional memory that respects tenant boundaries.
- **Safety and governance:** scope engine, egress firewall, approval gates, budgets, full audit log, content and action policies.
- **Findings and reporting:** typed findings, verified proof-of-exploit, run-to-run diffing, PDF and structured exports, and integrations (Jira, DefectDojo, ServiceNow, Slack, SIEM).
- **Platform:** SSO and enterprise identity, multi-tenant isolation, RBAC, REST plus GraphQL plus webhooks, CI/CD triggers, usage metering and billing.
- **Observability:** LLM tracing, metrics, cost dashboards, and a security audit trail distinct from the execution telemetry.

### 3.3 Phasing, with rough targets

Estimates below are planning ballparks for a senior team, not commitments. Adjust to your funding and risk appetite.

| Phase | Duration | Team | Goal | Success target |
|---|---|---|---|---|
| 0: Foundations | ~1 month | 2-3 | Provider abstraction, sandbox, scope and safety layer skeleton, findings model, eval harness scaffold | A single agent runs one safe, scoped, evidenced finding end to end |
| 1: MVP web attacker | 2-3 months | 3-5 | Recon plus web-exploit agents, verified findings, basic UI and API, continuous trigger | Detect and prove the OWASP Top 10 on a known-vulnerable app with low false positives |
| 2: Network and chaining | 2-3 months | 4-6 | Network and AD and cloud agents, unified attack-path graph, chained engagements | Chain a web foothold into lateral movement and a domain-level finding |
| 3: Enterprise and trust | 3-4 months | 5-8 | SSO, multi-tenant, audit, integrations, approval gates, published benchmark | Pass a design partner's security review; measurable false-positive rate |
| 4: Scale and GA | ongoing | 8+ | Hardening, performance, managed cloud tier, marketplace | Paying customers, SLA, continuous engagements at scale |

Roughly 9 to 12 months to a credible enterprise GA with a team scaling from about 3 to about 8 engineers.

### 3.4 What it will cost, in shape

Three cost centers, so plan all three:

- **Engineering.** The dominant early cost. A team of 3 to 8 senior engineers over the first year.
- **LLM inference.** The dominant run-cost at scale, and the reason budgets and the kill-switch are not optional. A single deep autonomous engagement can run many thousands of reasoning-model calls; `generator` and `pentester` on a frontier model are the expensive roles. Design cost controls (the cheap and deep tiering, caching, budget gates) as product features, not afterthoughts.
- **Infrastructure.** Sandboxes, databases, vector and graph stores, observability. Modest next to inference early on, but the sandbox fleet grows with concurrency.

The strategic cost lever is the per-role model tiering you already have: keep the frontier reasoning model on the two or three roles that set the plan, and push everything else down to cheap and fast models. That one discipline is the difference between a viable unit economic and a ruinous one.

---

## 4. A completely Azure-based architecture, end to end

This maps every component to a managed Azure service, chosen so the whole system can run inside a customer's own Azure tenant, which is itself the sovereign selling point. It also resolves the biggest gaps in the current code (sandbox isolation, egress control, enterprise identity, reranking) with managed services instead of hand-rolled ones.

### 4.1 Models and inference

- **Azure AI Foundry** as the model plane. Its catalog includes the GPT-5 family via **Azure OpenAI** and, through Foundry Models, managed access to other frontier models including Anthropic Claude. Keep the provider abstraction so a role can point at any catalog model.
- Apply the per-role tiering from the companion doc: a frontier reasoning model for `generator`, `refiner`, `coder`, and `pentester`; a mid model for the orchestration roles; a cheap and fast model for summarization, search, and classification.
- Use **Azure AI Content Safety** on inputs and outputs as a policy checkpoint, and Foundry's agent governance features for action logging.

### 4.2 Agent orchestration and the backend

- Run the Go backend on **Azure Kubernetes Service (AKS)** for full control, or **Azure Container Apps** for a lighter operational load. Container Apps is the faster path; AKS is the choice if you need fine-grained network policy and scale.
- Optionally adopt the **Foundry Agent Service** for parts of the agent runtime, since it is built on Container Apps and provides per-session sandboxes natively. Keep your own orchestration if you want provider independence.

### 4.3 The execution sandbox (this is the big win)

- Replace the raw Docker-socket sandbox with **Azure Container Apps Dynamic Sessions** or **Container Apps Sandboxes**, which reached GA in September 2026 with **hardware-isolated microVM (Hyper-V) isolation**, pre-warmed pools to avoid cold starts, and snapshot and resume for stateful runs.
- This directly fixes the current P0 problems: isolation is managed and microVM-grade, the host is never exposed, and you get a clean per-engagement teardown. Pair each session with a **Network Security Group** and **Azure Firewall** egress policy so the scope engine's allowlist is enforced at the network, not just in code.

### 4.4 Memory and data

- **Azure Database for PostgreSQL Flexible Server** with the **pgvector** extension for semantic memory, bringing the vector schema under managed, indexed, backed-up control (fixing the unmanaged-schema gap).
- **Azure AI Search** as the retrieval layer when you want hybrid (vector plus keyword) search and its built-in **semantic ranker**, which gives you the reranking the current RAG path is missing, as a managed service.
- For the attack-path and temporal graph, either **Neo4j Aura** on the Azure Marketplace (keeps Graphiti compatibility) or **Azure Cosmos DB for Apache Gremlin** if you prefer fully native.
- **Azure Blob Storage** for evidence artifacts (screenshots, captures, PCAPs, reports).

### 4.5 Identity, secrets, and networking

- **Microsoft Entra ID** for SSO, OIDC, SAML, and conditional access, replacing the current Google-and-GitHub-only OAuth and giving enterprises the identity integration they require.
- **Azure Key Vault** with **Managed Identity** for all secrets, replacing the insecure env-file defaults; no API key or DB credential lives in plaintext config.
- **Virtual Network** with **Private Endpoints** for the database, search, and model endpoints; **Azure Firewall** for the egress allowlist; **Azure Front Door** with **WAF** in front of the API and UI.

### 4.6 Observability, audit, and cost control

- **Azure Monitor** and **Application Insights** with OpenTelemetry for traces and metrics, and **Microsoft Sentinel** as the SIEM for the security audit trail, which is distinct from execution telemetry.
- Keep **Langfuse** (self-hosted on Azure) for LLM-specific tracing if you want its depth, or consolidate into Foundry's own tracing.
- **Azure Cost Management** budgets and alerts wired into the application-level kill-switch, so a runaway engagement trips both an app budget and a cloud budget.

### 4.7 Delivery and IaC

- **Azure Container Registry** for images, **GitHub Actions** or **Azure DevOps** for CI/CD, and **Bicep** or **Terraform** for infrastructure as code so a customer can stand up the whole stack in their tenant from a template. A one-command tenant deployment is the concrete form of the "sovereign, self-hosted" pitch.

### 4.8 Azure component map (summary)

| Concern | Azure service |
|---|---|
| Models | Azure AI Foundry / Azure OpenAI (GPT-5, managed Claude) |
| Content and action policy | Azure AI Content Safety, Foundry governance |
| Backend orchestration | AKS or Azure Container Apps |
| Agent runtime (optional) | Foundry Agent Service |
| Execution sandbox | Container Apps Dynamic Sessions / Sandboxes (microVM) |
| Egress enforcement | NSG plus Azure Firewall |
| Relational and vector DB | Azure Database for PostgreSQL Flexible Server plus pgvector |
| Hybrid search and reranking | Azure AI Search (semantic ranker) |
| Attack-path graph | Neo4j Aura (Marketplace) or Cosmos DB Gremlin |
| Evidence and artifacts | Azure Blob Storage |
| Identity and SSO | Microsoft Entra ID |
| Secrets | Azure Key Vault plus Managed Identity |
| Edge and WAF | Azure Front Door plus WAF |
| Observability | Azure Monitor, App Insights, Sentinel, (Langfuse) |
| Cost control | Azure Cost Management budgets plus app kill-switch |
| Registry and IaC | ACR, GitHub Actions / Azure DevOps, Bicep / Terraform |

---

## 5. The short version

- The market splits into app-attacker tools (XBOW and peers) and network-attacker tools (NodeZero, Pentera), almost all SaaS, almost none open-source.
- Do not out-XBOW XBOW. Win on four things they structurally cannot match: both attackers in one platform, verified and trusted findings, sovereign self-hosting, and safety and governance by construction.
- Building from scratch, put the safety layer, the typed findings model, the attack-path graph, and the eval harness in first, because they are the expensive retrofits and the actual differentiators.
- Plan roughly 9 to 12 months and a 3-to-8 person senior team to enterprise GA, with LLM inference as the cost you must engineer against from day one via per-role model tiering and hard budgets.
- Azure can host all of it in the customer's own tenant, and doing so simultaneously fixes the sandbox-isolation, egress, identity, and reranking gaps with managed services, while making "runs entirely in your Azure" a sales line.

---

## Sources

Competitor and market research (vendor-adjacent; verify figures against primary sources):

- [XBOW Alternatives: Top AI Pentesting Tools (Simbian)](https://simbian.ai/blog/xbow-alternatives-ai-pentesting-tools)
- [Best AI Pentesting Tools 2026 (MindFort)](https://www.mindfort.ai/blog/best-ai-pentesting-tools)
- [Best AI Pentesting Tools in 2026 (Strix)](https://www.strix.ai/blog/best-ai-pentesting-tools)
- [AI Pentesting Agents 2026: 39+ Tools Tested (AppSecSanta)](https://appsecsanta.com/research/ai-pentesting-agents-2026)
- [Agentic AI security startups funding and M&A, RSAC 2026 (Software Strategies)](https://softwarestrategiesblog.com/2026/03/28/agentic-ai-security-startups-funding-mna-rsac-2026/)
- [XBOW Raises $120M Series C (TAMradar)](https://www.tamradar.com/funding-rounds/xbow-series-c-120m)
- [Best XBOW Alternatives in 2026 (Synack)](https://www.synack.com/?p=27304)
- [NodeZero Alternatives (CodeAnt)](https://codeant.ai/blogs/nodezero-alternatives)
- [Top Automated and Autonomous Pentest Platforms 2026 (Pentestas)](https://pentestas.com/blog/automated-autonomous-pentest-platforms-companies)

Azure architecture:

- [Running Foundry Agent Service on Azure Container Apps (Microsoft)](https://techcommunity.microsoft.com/blog/appsonazureblog/running-foundry-agent-service-on-azure-container-apps/4518598)
- [Hosted agents in Foundry Agent Service (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/hosted-agents)
- [Azure Container Apps Sandboxes reach GA with microVM isolation](https://windowsforum.com/news/azure-container-apps-sandboxes-reach-ga-with-microvm-isolation.445745/)
- [Safely execute AI-generated code with Container Apps Dynamic Sessions (Microsoft)](https://techcommunity.microsoft.com/blog/appsonazureblog/even-simpler-to-safely-execute-ai-generated-code-with-azure-container-apps-dynam/4499795)
- [Govern AI Agents with the Agent Governance Toolkit and Container App Sandboxes (Microsoft)](https://techcommunity.microsoft.com/blog/linuxandopensourceblog/govern-ai-agents-using-agent-governance-toolkit-and-azure-container-app-sandboxe/4526011)
- [What's new in Azure Container Apps at Build'26 (Microsoft)](https://techcommunity.microsoft.com/blog/appsonazureblog/whats-new-in-azure-container-apps-at-build26/4524184)
