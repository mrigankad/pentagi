# Vedha: UI/UX Team Handover

Date: 2026-10-16
Audience: the product design team (UX research, interaction, visual).
Companion to the strategy, roadmap, playbook, and tracker in `docs/`.

This document gives the design team everything needed to start: who the users are, what actually hurts for them, how Vedha solves it, what belongs on their screens, and how to proceed. It is deliberately product-first, not pixel-first. The goal is shared understanding before anyone opens Figma.

The one idea to hold onto: Vedha is an autonomous agent that acts on a real network. The entire design challenge is turning that black box into something a professional can trust, steer, and stop. Transparency, control, and evidence are not features here. They are the product.

---

## 1. Who the end user is

Vedha has five distinct users. They do not want the same screens, and designing one dashboard for all of them is the first trap to avoid.

| Persona | Role | What they do with Vedha | How often |
|---|---|---|---|
| The Operator | Security engineer / penetration tester | Launches engagements, watches the agent work, validates findings, intervenes when needed | Daily, hands-on |
| The Manager | AppSec lead / security manager | Oversees coverage, triages and assigns findings, tracks team throughput and risk posture | Daily, oversight |
| The Developer | Engineer receiving findings | Reads an assigned finding, reproduces it, fixes it, marks it resolved | Occasionally, task-driven |
| The Executive | CISO / security director | Sees risk reduction over time, compliance posture, cost, and program ROI | Weekly or monthly |
| The Service Provider | MSSP consultant / pentest firm | Runs engagements on behalf of many client tenants, white-labels reports | Daily, multi-tenant |

The Operator and the Manager are the primary design targets for v1. The Developer experience is mostly delivered through integrations (a good Jira ticket matters more than a Vedha screen). The Executive needs one well-designed analytics view. The Service Provider is a tenancy and white-label layer on top, not a separate app.

A sixth stakeholder, the compliance or GRC officer, does not log in daily but drives hard requirements: scope attestation, an audit trail, and exportable engagement history. Design for their artifacts even though they are not a daily user.

---

## 2. What their problem actually is

Design from the pain, not from the feature list. Here is what each persona is really struggling with today, before Vedha.

**The Operator** is drowning in manual, repetitive work. Reconnaissance and tool-wrangling eat hours before any real testing starts. They context-switch across dozens of CLI tools and keep state in their head. When they finally automate something, they cannot see what the automation did or why, so they do not trust it. Their deepest fear with any autonomous tool is that it will act outside the authorized scope or do something destructive, and that fear is what kills adoption. They also hate writing reports.

**The Manager** cannot answer the questions their boss asks: are we actually covered, is our risk going up or down, and which findings matter most right now. Testing is point-in-time, so between engagements they are blind. Findings arrive as a noisy PDF with no prioritization and no way to track whether anything got fixed. They cannot measure their team's throughput or the tool's accuracy.

**The Developer** gets handed a vulnerability with a scary severity label, a vague description, and no reliable way to reproduce it. Half the time it is a false positive, so they learn to distrust and deprioritize the whole queue. They want a precise, reproducible, actionable ticket in the tool they already use, not a login to yet another security dashboard.

**The Executive** cannot connect security spend to risk reduction. They need a defensible number for the board and evidence for auditors and regulators, and today that is assembled by hand from scattered reports.

**The Service Provider** needs strict isolation between client tenants, fast engagement setup from templates, and branded reports, and they live in fear of data from one client leaking into another's view.

The thread through all of it: **trust and visibility.** Autonomous security tooling fails in the market not because the agent is weak but because humans cannot see what it did, cannot prove a finding is real, and cannot be sure it stayed in bounds.

---

## 3. How Vedha solves it, and what that demands from design

Map each solution to the interaction it requires. These are the design commitments.

- **Autonomy that is observable (the "glass box").** The agent runs itself, but the Operator can watch its reasoning, its commands, and its decisions live, and scrub back through them. Design demand: a real-time activity stream that makes a long autonomous run legible, not a spinner. The backend already streams this (agent logs, terminal I/O, screenshots, search logs over websocket subscriptions), so the data exists.
- **Findings you can trust.** Every finding carries reproducible proof: the exact command chain, captured evidence, and a confidence score, with a clear verified-versus-suspected distinction. Design demand: an evidence-first finding detail view where proof is the hero, not an afterthought, and confidence is always visible.
- **Control and safety that are always present.** Scope is defined up front and enforced; the Operator can pause, approve a high-impact action, adjust the budget, or kill the run at any moment. Design demand: persistent, unmissable controls and a live budget and scope indicator on every engagement screen. The safety layer (scope, budgets, approval gates) is being built by the safety team; the UI is its face.
- **Continuous, not point-in-time.** Engagements can run on a schedule or on deploy, and results diff against the last run. Design demand: a time dimension everywhere. New, fixed, and still-open are first-class states, and trend over time is a primary view.
- **Findings that reach the right person.** Verified findings flow into Jira, DefectDojo, Slack, and the SIEM. Design demand: the developer experience lives mostly outside Vedha, so design the outbound ticket and notification payloads as carefully as any in-app screen.
- **Sovereign and multi-tenant.** It runs in the customer's own environment with strict tenant isolation. Design demand: a tenant switcher and visual tenant context that make it impossible to act in the wrong client's space.

---

## 4. KPIs and what belongs on each screen

Different personas need different numbers. Putting the Executive's risk score on the Operator's live screen is noise; putting the Operator's iteration count on the board report is noise. Split them.

### Operator, live engagement screen (the signature screen)

The Operator is watching a run happen. The metrics are about progress, safety, and cost right now.

- Engagement status and phase (recon, exploitation, post-exploitation, reporting)
- Elapsed time and current iteration count
- Live budget burn against the cap (dollars and tokens), as a meter that turns warning colors as it approaches the limit
- Findings discovered so far, by severity, updating live
- Current agent action and target, with the scope indicator showing in-bounds
- Pending approvals (count and a clear call to action)
- Targets covered versus targets in scope

### Manager, findings and coverage dashboard

The Manager is triaging and reporting. The metrics are about the backlog, accuracy, and coverage.

- Open findings by severity and by status (new, triaged, assigned, fixed, accepted-risk)
- Mean time to remediate, trended
- New versus fixed versus still-open across the last N runs
- False-positive rate (the trust metric; this is also a sales metric, so make it prominent and honest)
- Attack surface coverage (assets tested versus known assets)
- Engagement throughput (runs completed, findings per run) and cost per engagement

### Executive, risk and program view

One clean view, export-ready. The metrics are about risk and ROI.

- Overall risk posture and its trend over time
- Risk reduction since last period (findings fixed, exposure closed)
- Compliance and scope attestation status
- Total cost and cost per finding or per engagement
- Coverage of the critical asset inventory

### Compliance, evidence view

- Scope attestation per engagement (what was authorized, what ran)
- Full audit trail (who launched what, when, against what)
- Exportable engagement history and evidence packages

A design rule for all four: progressive disclosure. The Executive view is the top of a funnel that drills into a finding, which drills into its evidence, which drills into the raw agent trace. The same truth at four depths, never four disconnected products.

---

## 5. The screens to design (information architecture)

A suggested map for v1. Validate and reshape it in discovery.

1. **Engagement list / home.** All engagements, their status, last run, and findings summary. Entry point for the Operator and Manager.
2. **New engagement wizard.** The most safety-critical screen in the product. The user defines scope (allowed targets, excluded targets, ports, time window), picks a profile (web, network, full), sets a budget, and chooses continuous or one-shot. Scope is the thing that keeps the agent safe and legal, so this screen must make scope definition clear, hard to get wrong, and reusable via templates.
3. **Live engagement view.** The glass box. Agent activity stream, live terminal output, the emerging attack-path graph, findings as they appear, and the persistent control bar (pause, approve, adjust budget, stop) with the live budget and scope indicators.
4. **Findings inbox.** Triage queue with filtering, sorting by severity and confidence, bulk actions, assignment, and status management.
5. **Finding detail.** Evidence-first. Severity and CVSS, confidence and verified status, affected asset, the reproduction command chain, captured evidence (screenshots, output), remediation guidance, status workflow, and push-to-integration actions.
6. **Attack-path graph.** Visualizes how the agent chained a foothold into deeper compromise across web, network, and cloud. This is the "both attackers, one platform" differentiator made visible, and a strong demo asset.
7. **Analytics and trends.** The Manager and Executive dashboards described in Section 4.
8. **Reports.** Generated reports, diff views between runs, and export.
9. **Assistant (interactive mode).** A conversational mode already exists in the backend; design it as a focused chat where a user can ask the agent to investigate or explain, distinct from a full autonomous engagement.
10. **Settings.** Model and provider configuration (the per-role model routing is powerful but needs a sane default so most users never touch it), scope templates, integrations, team and roles, and tenant management.

---

## 6. Design principles for this product

Five principles to hold every decision against.

1. **Trust is the product.** If a user cannot tell whether a finding is real or whether the agent stayed in scope, nothing else matters. Surface confidence, evidence, and scope state relentlessly.
2. **Control is always one click away.** Pause, approve, and stop are never buried. A user must be able to halt an autonomous agent instantly from anywhere in an engagement.
3. **Make the long run legible.** A multi-hour autonomous engagement must read as a comprehensible narrative, not a loading state. This is the hardest and most important interaction problem in the product.
4. **Progressive disclosure.** Executive summary down to raw agent trace, same truth at every depth. Never force a developer through an executive dashboard or an executive through a terminal log.
5. **Meet users where they work.** Developers live in Jira and Slack, not Vedha. The best finding UX is often a ticket that never requires opening the app.

Two craft notes: security professionals strongly prefer a dark, information-dense interface, so treat dark mode as the default, not an option. And accessibility is a procurement requirement for enterprise and government buyers, so design to WCAG AA from the start rather than retrofitting.

---

## 7. How to proceed

A phased plan that fits alongside the engineering roadmap.

**Phase 1: Discovery (about 2-3 weeks).**
- Interview three to five people per primary persona, ideally from design-partner organizations. Build jobs-to-be-done statements, not feature wishlists.
- Study the competitors from the strategy doc to learn the category's conventions and where they frustrate users, so Vedha can do better rather than merely match.
- Review what data the backend already exposes. The GraphQL schema and its live subscriptions (`backend/pkg/graph/schema.graphqls`) tell you what the live engagement screen can show today, and the findings model (built by the data team) tells you what the finding detail view will carry. Design to real data, not imagined data.
- Output: validated personas, journey maps, a jobs-to-be-done list, and a prioritized problem inventory.

**Phase 2: Structure (about 2 weeks).**
- Information architecture and the core user flows: define a scope and launch an engagement, watch and steer a live run, triage and assign a finding, and read the trend dashboard.
- Low-fidelity wireframes for the ten screens, pressure-tested against the KPIs in Section 4.
- Output: IA, flows, and wireframes, reviewed with engineering for feasibility against the API.

**Phase 3: Validate (about 2 weeks).**
- Clickable prototype of the two signature flows: the new-engagement scope wizard and the live glass-box engagement view.
- Usability test with operators and managers. The two questions to answer: can they define a safe scope without error, and can they understand what the agent is doing while it runs.
- Output: tested prototype and a prioritized fix list.

**Phase 4: Build-ready design (ongoing).**
- High-fidelity designs and a design system, dark-mode-first, WCAG AA.
- Component-level handoff to the frontend team (the app is React and TypeScript in `frontend/`), aligned to the engineering phases so design lands just ahead of each build.
- Output: design system, annotated specs, and a living component library.

**Sequencing with engineering.** The finding-centric screens depend on the data team's findings model, and the safety controls depend on the safety team's scope and budget work, so design those screens in parallel with that backend work and ship the UI as each lands. The live engagement view can be prototyped immediately against the existing subscription streams.

---

## 8. The short version

- Five users, but design v1 for the Operator and the Manager; the Developer is served through integrations and the Executive through one analytics view.
- Their real problem is trust and visibility, not a missing feature. Autonomous security tools fail when humans cannot see what the agent did, prove a finding is real, or be sure it stayed in scope.
- The signature screen is the live glass-box engagement view, and the most safety-critical screen is the scope wizard. Get those two right first.
- Put progress, budget, and scope on the Operator's screen; backlog, accuracy, and coverage on the Manager's; risk and ROI on the Executive's. Same truth at every depth.
- Proceed discovery, structure, validate, build-ready, sequenced so design lands just ahead of each engineering phase, designing against the real API and the real findings model.
