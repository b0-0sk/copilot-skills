# Disposable Docker Compose environment

Use this only if the application needs a reproducible stack; otherwise reuse the existing test environment.

1. Inspect existing compose files and service images first. Prefer `compose.e2e.yaml` or an e2e profile/override over modifying developer/prod defaults. Ensure the **real application binary/image** is started together with only required dependencies. Provide fake/test credentials solely for isolated test services.
2. Check ports, networks, hostnames, test DB initialization/migrations, and service healthchecks. `depends_on: condition: service_healthy` gates dependency startup, not the business journey; use independent HTTP readiness probes for the application.
3. Use a dedicated Compose project name that cannot collide with a developer's current stack (e.g. an explicit unique `COMPOSE_PROJECT_NAME`). Avoid binding ports publicly where unneeded. If test client runs on host, publish only the app test port. If inside Compose, use service DNS.
4. Example start (adjust to actual repo):
   `docker compose -p <unique-e2e-project> -f compose.e2e.yaml up -d --build --wait --wait-timeout 120`
   `--wait` waits for healthy/running services, subject to healthcheck configuration; it is **not** an acceptance test.
5. On startup failure: capture `docker compose ... ps` and scoped service logs, redact secrets, report `BLOCKED`, and clean up owned resources when safe.
6. Always record ownership before teardown. When using your own fresh isolated project, `docker compose -p <same-name> -f compose.e2e.yaml down` removes your project containers/networks. Never `down` someone else's stack. Never run blanket `docker system prune` or delete volumes by default; volume cleanup requires proven disposable ownership and user authorization.
7. Prefer a portable developer/CI workflow; do not commit generated test databases, secrets, or reports. Guard non-test endpoints and block production execution without explicit user authorization.

Official references:
- https://docs.docker.com/compose/how-tos/startup-order/
- https://github.com/docker/compose/blob/main/docs/reference/compose_up.md
