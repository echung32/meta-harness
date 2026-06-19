# Discovered values (read from cloned Omnigent repo)

Pinned against the local clone at `~/omnigent`.

- **OMNIGENT_REF (pinned, detached HEAD):** `918c153`
- **Image:** `ghcr.io/omnigent-ai/omnigent-server:${OMNIGENT_IMAGE_TAG:-latest}`
  (pin `OMNIGENT_IMAGE_TAG=sha-<short>` for reproducibility). GHCR package may be
  **private** during alpha → `docker login ghcr.io` with a `read:packages` token first.
- **Host image (sandboxes only, not needed here):** `ghcr.io/omnigent-ai/omnigent-host`.

## Ports / services (CORRECTS the plan — was 6767/SQLite)

- **Server port: `8000`** (web UI at `http://localhost:8000`). NOT 6767.
- **DB: Postgres** (`postgres:16-alpine`), `DATABASE_URL=postgresql+psycopg://...`.
- **Compose service names:** `omnigent` (server), `postgres`. Optional `caddy` from overlay.
- Compose project name: `omnigent`.

## Official deploy files already shipped (use these, don't reinvent)

- `~/omnigent/deploy/docker/docker-compose.yaml` — server + postgres; publishes
  `${OMNIGENT_PORT:-8000}:8000`.
- `~/omnigent/deploy/docker/docker-compose.https.yaml` — **Caddy auto-HTTPS overlay**;
  `!reset []` drops the direct 8000 publish, Caddy proxies `omnigent:8000` over the docker
  net, publishes 80/443. Requires Compose **2.24+** for `!reset` (host has v5.1.4 ✓).
- `~/omnigent/deploy/docker/Caddyfile` — 3-line `{$OMNIGENT_DOMAIN} { reverse_proxy omnigent:8000 }`.
- `~/omnigent/deploy/docker/.env.example` — full env walkthrough.
- `~/omnigent/deploy/docker/bootstrap.sh` — idempotent; mints `POSTGRES_PASSWORD` +
  cookie secret into `.env`.

## Auth model (three modes; the single-Gmail constraint matters)

- `OMNIGENT_AUTH_ENABLED` master switch (default `1`).
- **accounts** (default): built-in username/password; admin auto-created, password in
  `docker compose logs omnigent` + `/data/admin-credentials`.
- **oidc** (set `OMNIGENT_OIDC_ISSUER`): server runs `/auth/login|callback|logout` itself.
  Restriction is **domain-level only** (`OMNIGENT_OIDC_ALLOWED_DOMAINS`) →
  **cannot safely gate a single personal Gmail** (`gmail.com` would admit all Gmail users).
- **header** (`OMNIGENT_AUTH_PROVIDER=header`): trusts `X-Forwarded-Email` from a front
  proxy (oauth2-proxy named); rejects requests without it. Proxy must strip client-supplied
  copies.
- **AUTH_ENABLED=0**: single-user "local" mode — every request is the shared `local` user,
  no login. Docs: "local dev only, never for shared deploys." Safe ONLY if the sole way to
  reach the port is already identity-gated + network-isolated.
- Redirect URI is derived from `OMNIGENT_DOMAIN` → `https://<domain>/auth/callback` (no
  direct `OMNIGENT_OIDC_REDIRECT_URI` knob on the domain path).

## Runner / host model

- Web UI prints the CLI command to launch a local runner. Documented form:
  `omnigent run <agent.yaml> --server http://localhost:8000`.
- Host tunnel endpoint `/v1/hosts/{host_id}/tunnel` (WebSocket, outbound from host).
  Authenticates owner from the WS handshake before accept. `OMNIGENT_LOCAL_SINGLE_USER`
  enables single-user loopback host re-own across auth-mode flips.
- CLI verbs: `omnigent run` confirmed in docs; `omnigent host` referenced in PLAN/older
  docs. **Confirm exact verbs via `omnigent --help` at install time (Task 7).**

## Implications for the plan

1. Replace all `6767` → `8000`.
2. Build on the **official compose + https overlay**; our override only adds oauth2-proxy,
   a localhost-only publish for the runner, and the auth-mode choice.
3. Single-Gmail restriction must come from **oauth2-proxy** (`--authenticated-emails-file`),
   not native OIDC.
4. The previous spec's "Omnigent native auth as an inner layer" needs revisiting — see the
   design fork raised with the operator.
