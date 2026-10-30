# Multi-stage acceptance suites (local, test, integration, prod)

One test suite, many stages. Test code is **identical** for every stage; only the per-stage configuration changes (base URL, auth mechanism, expected values, input parameters).

## Stage catalogue

| Stage      | Group  | Notes                                                        |
|------------|--------|--------------------------------------------------------------|
| `local`    | local  | Disposable local app or Compose stack (default).             |
| `test`     | QA     | Authorized shared test environment.                          |
| `integration` | QA  | Authorized integration environment.                           |
| `prod`     | prod   | Read-only smoke checks only, explicit confirmation required. |

`local` and `prod` are separate stages. "QA" means `test` + `integration`. If the project uses different stage names, map them to this catalogue and record the mapping in the README; do not invent extra stages.

## Interactive scope selection

Run **before generating tests**, only when `<e2e dir>/README.md` has no `Stage scope` section (see below). Use `ask_user` with explicit choices.

1. **Stages to cover now:**
   - `local only`
   - `local + QA` (local, test, integration)
   - `all` (local, test, integration, prod)
2. **Remaining stages** (only if some stage is not covered by answer 1; skip when answer 1 was `all`):
   - `create all remaining now`
   - `choose which remaining to create now` (follow-up multi-choice; the unchosen ones are recorded as `deferred`)
   - `none, and do not ask again` (remaining stages are recorded as `declined`)
3. Create only the selected stages' config files. Future stages reuse the same pattern: add `stages/<stage>.toml` plus the stage's secrets as environment variables; no test code changes.

## Persisting the decision

After the questions, write/update `<e2e dir>/README.md` (in English) with this section:

```markdown
## Stage scope

<!-- e2e-stage-scope:v1 -->
- Created: local, test, integration
- Deferred: (none)
- Declined: prod

No further questions about additional stages are needed: the stages listed under "Declined" were deliberately excluded.
Do not ask again about them unless the user explicitly asks to add one.
To add a stage later, copy `stages/test.toml` to `stages/<stage>.toml`, adjust values, and export its credentials; test code does not change.
```

Rules for later invocations:
- Section present and `Deferred` empty: **do not ask anything about stages**.
- `Deferred` not empty: offer those stages once, only when the user extends the suite; move answers to `Created` or `Declined`.
- Never ask about `Declined` stages unless the user explicitly requests them.

## Layout

```
<e2e dir>/
  README.md
  conftest.py
  stages/
    local.toml
    test.toml
    integration.toml
    prod.toml        # only if created
  test_<journey>.py
```

`stages/<stage>.toml` is committed and contains **no secrets**:

```toml
base_url = "https://<host>"            # real value from the environment owner
[auth]
kind = "bearer"                         # real mechanism discovered in the project: bearer | basic | oidc-client-credentials | none ...
token_env = "E2E_TEST_TOKEN"            # name of the env var, never the value
[expect]                                # stage-specific expected values used by assertions
# e.g. tenant = "..."
[params]                                # stage-specific inputs (seed ids, users, ...)
```

Do not guess auth kinds, hosts or expected values: derive them from the project or ask the user. Prefer `tomllib` (Python 3.11+); on older runtimes use an equivalent loader already present in the project.

## Selector and safe failure

Select the stage with `E2E_STAGE` (default `local`). `conftest.py` must:
- Fail with a clear message when the stage is unknown, its file is missing, or a referenced env var is unset (no silent skip).
- Expose a `stage` fixture (config) and an authenticated client built from it; tests read URLs, expected values and params only from there.
- Never log or print credential values.

```python
import os, tomllib
from pathlib import Path
import httpx, pytest

STAGES_DIR = Path(__file__).parent / "stages"
SAFE_METHODS = {"GET", "HEAD", "OPTIONS"}

@pytest.fixture(scope="session")
def stage():
    name = os.environ.get("E2E_STAGE", "local")
    path = STAGES_DIR / f"{name}.toml"
    if not path.is_file():
        pytest.exit(f"Unknown or missing stage config: {name}", returncode=2)
    cfg = tomllib.loads(path.read_text(encoding="utf-8"))
    cfg["name"] = name
    return cfg

def pytest_collection_modifyitems(config, items):
    if os.environ.get("E2E_STAGE", "local") != "prod":
        return
    if os.environ.get("E2E_CONFIRM_PROD") != "prod":
        pytest.exit("Prod run requires explicit user confirmation (E2E_CONFIRM_PROD=prod)", returncode=2)
    skip = pytest.mark.skip(reason="only @pytest.mark.readonly tests run on prod")
    for item in items:
        if "readonly" not in item.keywords:
            item.add_marker(skip)

@pytest.fixture(scope="session")
def api_client(stage):
    auth = stage.get("auth", {})
    headers = {}
    if auth.get("kind") == "bearer":
        token = os.environ[auth["token_env"]]  # KeyError message names the missing variable
        headers["Authorization"] = f"Bearer {token}"
    hooks = {}
    if stage["name"] == "prod":
        def block_mutations(request):
            if request.method not in SAFE_METHODS:
                raise RuntimeError(f"{request.method} is forbidden on prod")
        hooks = {"request": [block_mutations]}
    with httpx.Client(base_url=stage["base_url"], headers=headers, timeout=10.0, event_hooks=hooks) as client:
        yield client
```

Register the `readonly` marker in `pytest.ini`/`pyproject.toml`.

## Prod policy

- Prod is optional and only if chosen in the scope question.
- Only tests marked `readonly` (read-only/smoke, no create/update/delete) may run; mutations are blocked by the client.
- Ask the user for explicit confirmation **before every prod execution** (`ask_user`); only then set `E2E_CONFIRM_PROD=prod` for that run. A "no" is reported as unrun, never as a pass.

## Executing against a stage

Command pattern (Windows): `$env:E2E_STAGE="test"; py -m pytest <e2e dir> -q`. Non-local stages (`test`, `integration`, `prod`) require the user's confirmation to execute against that environment, per the engineering workflow gates. Redact credentials in all reports.
