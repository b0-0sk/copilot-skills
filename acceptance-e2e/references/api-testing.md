# Python API acceptance testing (pytest + httpx)

1. Locate the contract of the *actual* public API (OpenAPI, endpoint/controller, authorization, payload and status semantics), plus known fixtures.
2. Create `conftest.py` in the e2e directory with a session-scoped `httpx.Client` built from the stage config selected by `E2E_STAGE` (see [Stages](stages.md)); credentials come only from the environment variables named in that config. Do not hardcode credentials. Fail clearly when required configuration is missing; never silently `skip` broken setups.
3. Create `test_<journey>.py` with readable arrange/act/assert tests. Prefer test names like `test_authorized_user_can_download_document` or `test_expired_token_is_rejected`.
4. Verify application behavior, not generic transport: relevant status AND payload/schema AND observed business state (e.g. re-read created resource, check idempotency, verify rejection did not change state).
5. Use unique IDs per test run and cleanup with `yield` fixtures; avoid depending on execution order or shared state. Use idempotent/setup endpoints only if supported by the real application or seeded through a dedicated isolated environment.
6. Timeout every request. Await only eventual consistency with bounded polling; do not mask failing assertions with repeated retries.
7. `py -m pytest <e2e dir> -q --junitxml=artifacts/e2e-junit.xml` on Windows (`python -m pytest` on other systems) is an example once dependencies/config are provided. Produce a project-specific reproducible command; do not assert this path works before checking.
8. For auth scenarios, use short-lived **test** credentials/tokens supplied through a secure test provider and never persist token bytes in logs, snapshots or test files.

Reference `conftest.py` outline (stage selection, auth, prod guard): see [Stages](stages.md). Tests read expected values and parameters from the `stage` fixture instead of hardcoding them, so the same test runs on every stage.

Do **not** generate tests with made-up URLs or trivial `assert True`. Derive actual scenarios from the repo and acceptance criteria.
