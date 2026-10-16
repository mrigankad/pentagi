# Vedha

Autonomous security-testing platform. Go backend in `backend/` (module `pentagi`).
Product name is Vedha.

## Build and test (run from backend/)
- `go build ./...`
- `go vet ./...`
- `go test ./...`
- `gofmt -l .`   (must print nothing)

## Conventions
- Feature branches: `vedha/<team>/<task>`. Never commit to the default branch.
- Any runtime-behavior change goes behind a config flag (prefix `VEDHA_`) in
  `backend/pkg/config/config.go` with a safe default (off, or current behavior).
- Add unit tests for new logic. Never weaken or skip existing tests.
- No secrets in code or config files; read from env via the config struct.
- Match existing patterns; study a sibling file before adding a new one.
- Keep changes in scope; do not refactor unrelated code.

## Key docs
- `docs/BACKEND_ASSESSMENT_AND_ROADMAP.md`  (hardening roadmap)
- `docs/PRODUCT_STRATEGY_AND_AZURE_PLAN.md` (strategy and Azure architecture)
- `docs/VEDHA_TEAM_PLAYBOOK.md`             (team assignments and Claude Code prompts)

## Architecture quick map
- `backend/pkg/controller`  flow/task/subtask orchestration and state
- `backend/pkg/providers`   LLM provider abstraction, per-role model routing
- `backend/pkg/tools`       agent tools (terminal, browser, search, memory, sub-agents)
- `backend/pkg/docker`      sandbox container lifecycle
- `backend/pkg/graph`       GraphQL API; `backend/pkg/server` has REST and auth
- `backend/pkg/database` + `backend/sqlc` + `backend/migrations`  data layer
- `backend/pkg/csum`        chain summarization (context management)

## Definition of done
Code builds, vet clean, gofmt clean, new tests pass, existing tests pass,
behavior change is flagged, PR body states how it was verified.
