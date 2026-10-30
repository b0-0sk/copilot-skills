---
name: acceptance-e2e
description: Generate and run repeatable black-box end-to-end acceptance tests for an application, using Python pytest with HTTPX for APIs or Playwright for UIs; optionally provision an isolated local environment with Docker Compose. The same tests can target several stages (local, test, integration, prod) changing only per-stage configuration. Use after an approved engineering change or when explicitly asked to create acceptance tests. Avoid mocks of the system under test, production writes, and superficial health-only checks.
---

## GitHub Copilot compatibility

Use this skill in GitHub Copilot desktop, CLI, VS Code or JetBrains agent mode. Use only tools actually available in the current client. To invoke a related skill, discover and read its installed SKILL.md; its name alone is not an executable tool call. Resolve personal skills relative to ~/.copilot/skills and project skills from supported repository skill directories. Prefer the client-provided installed skill location when available. Use a native question tool when available; otherwise ask in chat and wait for the answer. Never infer an answer or approval from silence. On Windows use py and py -m for Python commands unless the project explicitly requires another interpreter.


# Acceptance E2E — black-box user journeys

## Scope and authorization

This skill owns **acceptance scenarios, test code, environment setup, execution and evidence**. It does not own prompt refinement, requirement approval or product implementation. Follow applicable `AGENTS.md` and existing test conventions.

- Run only after the user explicitly requests acceptance testing, or as part of an **already authorized** implementation workflow. Do not bypass a pending requirements gate.
- Before running external processes, creating containers or writing test files, check that tools/permissions are available and the target is a disposable **local or authorized test environment**. Never run against production by default; prod is only allowed as read-only smoke tests with explicit confirmation on every run (see [Stages](references/stages.md)).
- Do not infer API routes, UI selectors, payload formats, response codes, credentials or business invariants: verify them in source, contracts, docs or a running test instance. Ask for genuinely missing business assertions; otherwise proceed with a bounded plan.
- Prefer extending an existing black-box test framework rather than adding a parallel Python suite. Use Python when the user requests it or it provides clear cross-service value. Do not replace existing JUnit/Testcontainers tests.

## Decision procedure

1. **Identify user-visible acceptance criteria.** From the approved request and changed behavior, select 1–5 high-value scenarios: primary successful journey, important rejected input/authorization, observable state or side effect, and a relevant regression. Avoid reproducing unit tests at system level.
2. **Discover existing entrypoints.** Inspect public API/OpenAPI, UI routes, auth flow, existing compose/test config and affected components. Record exact contracts and how to seed disposable data. Never define a test merely to match current implementation if it contradicts approved behavior.
3. **Choose the cheapest sufficient execution path:**
   - API backend: Python `pytest` + `httpx`, black-box HTTP requests to a **running application**.
   - Browser UI: Python `pytest-playwright` and semantic locators, with backend present.
   - Already well-covered by repository-native acceptance framework: extend it and explain why a second Python suite is unnecessary.
4. **Environment:** Reuse a safe existing dedicated test stack when possible; otherwise add a narrowly scoped Compose override/file with the real application and essential infrastructure. Do not make Compose mandatory when a local app instance already provides equivalent isolation. Read [Compose isolation](references/compose-isolation.md) only if container orchestration is needed.
5. **Choose stages.** Read [Stages](references/stages.md). Unless `<e2e dir>/README.md` already has a `Stage scope` section, ask interactively which stages to cover (local only / local + QA / all) and, if stages remain, whether to create all, choose some, or none (and not ask again). Persist the decision in that README. Tests are written once; each stage only adds a `stages/<stage>.toml` and its credentials via environment variables. Never ask again about stages when the README says no further questions are needed.
6. **Generate committed, maintainable tests.** Location precedence: an existing e2e location of the project; otherwise `<project>/.copilot/tools/e2e/` (the default; do not use `tests/e2e/` unless the project already does). Before writing, run the bootstrap below. For HTTP API specifics read [API testing](references/api-testing.md); for browser flows read [UI testing](references/ui-testing.md). Use `conftest.py` fixtures for the stage config (base URL, auth, expected values, params), secrets through environment, data setup/cleanup, unique test IDs and real assertions. Prefer `pytest` tests over a one-off procedural `.py` that hides failures.
7. **Execute and validate:** Inspect the test environment, start only test-owned resources, wait for readiness with bounded timeouts, run focused tests, collect evidence, and tear down only test-owned resources. Treat infrastructure failure as `BLOCKED`, not `PASS`. Never claim success on generated-but-unexecuted tests.
8. **Report:** `PASS`, `FAIL` or `BLOCKED`; user journey; files added; run commands; actual assertions, duration/exit status; logs, reports and artifacts available; suspected product regression vs environment problem. Preserve enough diagnostics to reproduce without exposing credentials.

## Non-negotiable quality rules

- Exercise **public boundaries**; do not import the application service internals into Python tests.
- A health/readiness check is **setup**, not business acceptance. Assert response content/headers AND subsequent visible state or observable side effects when relevant.
- Use deterministic fixtures; isolation between tests; bounded polling for eventual consistency (never arbitrary long sleeps), explicit timeouts and meaningful error messages.
- Exercise the real application artifact with real relevant components. Stub external third parties only at documented test boundaries; identify any mocks in the final report so no one mistakes a partial system test for complete E2E coverage.
- Avoid modifying/deleting existing data, global Docker volumes or unrelated containers; never call `docker compose down -v` for an existing stack. Prevent accidental staging/commit of secrets or test result artifacts.
- Do not make changes to product behavior solely to get a test green unless separately authorized; when it fails, present evidence and ask or hand back to the owning implementation workflow.
- Minimize tokens by keeping the primary skill concise and reading only mode-specific references. Do not generate unnecessary Compose files, diagrams, test abstractions, CI jobs or broad suites.

## Project bootstrap: `.copilot/tools`

Run only after the user has confirmed generating or updating tests (see `engineering-workflow` section 5) and only when the default location is used.

1. Create `<project>/.copilot` and `<project>/.copilot/tools` if missing (both), plus `e2e/` when writing tests.
2. If the project is a git repository and this is the first time `.copilot/tools` is created (it is neither tracked nor listed in `.git/info/exclude`), ask with `ask_user`: **local only** or **include in the repo**.
   - **Local only:** append `.copilot/tools` to `.git/info/exclude`. Do not edit `.gitignore` or any shared file.
   - **Include in the repo:** ignore nothing and leave the files ready for commit. Never commit or push without an explicit order.
3. If it already exists and the decision is detectable (listed in `.git/info/exclude` or tracked), do not ask again. If the project is not a git repository, skip the question and say so.
4. Never write secrets into `.copilot/tools`; use environment variables and an `.env.example` without real values.

## Integration with the harness

`prompt-cycle` approves the request first. `engineering-workflow` detects API impact, asks for confirmation to generate/update and to execute, then performs the authorized checks. Invoke `acceptance-e2e` for changed public/user behavior, new endpoints, workflow integration, or other tasks whose acceptance cannot be established by unit/component tests. Report acceptance back to the engineering workflow. Never self-approve requirements or silently deploy to a remote environment.

## Example explicit invocations

- `acceptance-e2e Create and run black-box API acceptance tests for the newly added JWT issuer flow, using the repository's test stack.`
- `acceptance-e2e Generate pytest browser tests for the approved checkout flow; use Docker Compose only if the repo lacks an isolated test environment.`
