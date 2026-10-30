# Python browser acceptance (pytest-playwright)

- Use `pytest-playwright` and the `page` fixture; configure target base URL and auth from the per-stage config selected by `E2E_STAGE` (see [Stages](stages.md)); on `prod` only `readonly` journeys (no form submissions that persist data) may run. Prefer `get_by_role`, `get_by_label`, `get_by_test_id` when the product exposes stable attributes. Avoid brittle CSS selectors tied to styling.
- A real user journey includes interaction and observable outcome, for example: sign in with a **test** account -> submit a form -> navigate to list/detail -> confirm saved state. Choose real flows from the approved task.
- Make tests independent: fixture-owned accounts/data, unique IDs and teardown. For authentication, use a supported sandbox identity/test credentials; do not bypass auth to call it E2E.
- Use Playwright auto-wait/expect rather than `sleep`, with bounded deadlines. For async workflows, poll on visible outcome.
- Capture trace/screenshot on failure where configured; scrub PII/secrets and restrict artifact retention.
- Only generate UI tests if the app actually has a browser UI. For REST-only Spring microservices prefer httpx.
- Official docs: https://playwright.dev/python/docs/intro and https://playwright.dev/python/docs/api-testing
