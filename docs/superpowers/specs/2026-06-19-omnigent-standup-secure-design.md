# Design: Stand up & secure the Omnigent control plane (sub-project A)

- **Date:** 2026-06-19 (rev 2 — rebased on the real Omnigent deploy after discovery)
- **Status:** Approved design — in implementation
- **Source of truth for goals:** [PLAN.md](../../../PLAN.md)
- **Discovered facts:** [deploy/DISCOVERED.md](../../../deploy/DISCOVERED.md)
- **Method:** Superpowers brainstorming (6.0.3)

## Scope

Sub-project A only: bring up the Omnigent control plane on this single Ubuntu host and
secure its public-facing web UI for a single operator.

**Out of scope** (sub-project B, its own spec→plan→implement cycle): the cross-vendor
writer↔reviewer loop, Polly/custom bundle, Superpowers-in-subagent wiring, and
cost/ask-on-shell policies.

## Decisions (locked during brainstorming + discovery)

| Decision | Choice | Rationale |
|---|---|---|
| Execution plane | **This host itself** (local runner) | No devcontainer; PLAN.md "don't double-sandbox" |
| Server location | **Co-located** on this box | Single operator; one host |
| Exposure | **Public IP + Caddy TLS** on a domain | User has a public IP + domain |
| Front identity gate | **Self-hosted oauth2-proxy**, GitHub OAuth (`--github-user`) | Need a single-account allowlist; native Omnigent OIDC restricts by domain only. (GitHub chosen over Google: simpler app registration, exact single-user allowlist.) |
| Inner auth | **`OMNIGENT_AUTH_ENABLED=0` (local mode)** | Single operator; oauth2-proxy is the sole gate; loopback runner works with no header plumbing |
| Compose | **Build on the shipped official stack** + a small gate override | Don't reinvent compose/Caddy |
| Server port | **8000** | Real Omnigent port (NOT 6767) |
| Node | **Volta**-managed Node 22 | Volta installed; Omnigent needs Node 22 |

## Host baseline (verified 2026-06-19)

Ready: Ubuntu 24.04.4, Docker 29.5.3 + compose v5.1.4 (≥2.24 for `!reset` ✓),
Python 3.12.3, tmux 3.4, `claude` + `codex` logged in (creds 0600), no `*_API_KEY` env.
**Gap:** Node v18 → upgrade to 22 via Volta.

## Architecture

Two entry paths into the server; the public one is identity-gated, the loopback one is the
single local user. Omnigent runs **without its own login** (`AUTH_ENABLED=0`), so its port
must never be reachable except via the gate or localhost.

```
                ┌──────────────────── this Ubuntu box (public IP) ────────────────────┐
 Browser/phone ─443─▶ caddy ──▶ oauth2-proxy ──upstream──▶ omnigent:8000 ──┐          │
 (GitHub login,     (TLS)      (GitHub OAuth,            (docker net,      │          │
  cookie)                       --github-user=you)       AUTH_ENABLED=0)  postgres   │
                │                                                    (docker net)      │
                │   omnigent local runner (runs as ethan on host)                     │
                │     └─ connects to 127.0.0.1:8000  →  treated as the "local" user   │
                │        (no header needed; port is localhost-only + firewalled)      │
                │     └─ spawns claude/codex subprocesses using ~/.claude + ~/.codex  │
                └──────────────────────────────────────────────────────────────────────┘
```

### Two entry paths (the key structural decision)

- **Human path** — `browser → caddy(TLS) → oauth2-proxy(GitHub OAuth, single-account
  allowlist) → omnigent:8000`. oauth2-proxy runs as a **full reverse proxy**
  (`--upstream=http://omnigent:8000`); unauthenticated requests never pass it.
- **Machine path** — the local runner connects to **`127.0.0.1:8000`** (a localhost-only
  published port). With `AUTH_ENABLED=0` the server treats every request as the single
  `local` user, so the loopback runner needs no credential. Safe because that port is bound
  to `127.0.0.1` and firewalled — only local processes reach it.

Both the gated browser user and the loopback runner resolve to the **same single operator**,
which is exactly the single-user model.

### Defense layers (outermost → innermost)

1. **UFW** — default-deny inbound; allow only 22/80/443.
2. **Caddy** — Let's Encrypt TLS; the only service published to `0.0.0.0`.
3. **oauth2-proxy** — GitHub OAuth; `--github-user` = your account(s) (comma-separated
   `OAUTH2_PROXY_GITHUB_USER` allows your own personal + work logins — all still resolve to
   the one `local` Omnigent user); strips client-supplied `X-Forwarded-*`.
4. **Network isolation** — omnigent + postgres + oauth2-proxy have no public port; the
   server's only host port is `127.0.0.1:8000`. `AUTH_ENABLED=0` is acceptable *only*
   because of layers 1–3 + this binding.

A flaw in any single outer layer doesn't expose the app by itself; reaching the app at all
requires either a GitHub-authenticated session as you, or local process access on the box.

## Components & file inventory

**Reused from the cloned repo** (`~/omnigent/deploy/docker/`, pinned ref):
- `docker-compose.yaml` (server `omnigent` + `postgres`), `docker-compose.https.yaml`
  (Caddy auto-TLS overlay), `bootstrap.sh`, `.env.example`.

**Authored in this repo** (`/home/ethan/meta-harness/deploy/`):
- `docker-compose.gate.yaml` — override that (a) adds the `oauth2-proxy` service, (b) points
  Caddy at oauth2-proxy via a custom Caddyfile, (c) re-adds `127.0.0.1:8000:8000` on
  `omnigent` for the loopback runner, (d) sets `OMNIGENT_AUTH_ENABLED=0`.
- `Caddyfile` — `{$OMNIGENT_DOMAIN} { reverse_proxy oauth2-proxy:4180 }`.
- `.env` — all secrets + `OMNIGENT_DOMAIN` + `OAUTH2_PROXY_GITHUB_USER`, passed via
  `--env-file` (gitignored). Single-account allowlist is `--github-user` (env), no file.
- `.gitignore` — `.env`, `caddy_*`.

**Host-level:** Node 22 (Volta); `omnigent` CLI + local runner run as `ethan`, inheriting
`~/.claude` + `~/.codex`.

**Run command:**
```
docker compose --env-file deploy/.env \
  -f ~/omnigent/deploy/docker/docker-compose.yaml \
  -f ~/omnigent/deploy/docker/docker-compose.https.yaml \
  -f deploy/docker-compose.gate.yaml up -d
```

**Alpha caveats (pin from the clone, not docs):** exact CLI verbs (`omnigent run` vs
`omnigent host`) confirmed via `--help`; GHCR image may be private → `docker login ghcr.io`
or build locally.

## Bring-up sequence (each step verified before the next)

1. Node 22 via Volta; `node -v`.
2. Pin the cloned repo to a fixed ref.
3. GitHub OAuth App (callback `https://YOUR-DOMAIN/oauth2/callback`); DNS A record → IP.
4. Author `deploy/` gate files + `.env` (secrets, `OMNIGENT_DOMAIN`, GitHub creds, username).
5. `docker compose ... up -d`; verify `127.0.0.1:8000` bound localhost-only; Caddy cert issued.
6. UFW default-deny; allow 22/80/443; verify 8000 unreachable externally.
7. Install `omnigent` CLI; launch the local runner per the UI's printed command.
8. Creds sanity: no `*_API_KEY`; `claude /status` = subscription.
9. Smoke test from phone browser.

## Error handling / known failure modes

- Caddy cert fails → DNS + port 80 open for ACME (HTTP-01).
- oauth2-proxy redirect loop → cookie-domain / redirect-url must match `YOUR-DOMAIN`.
- App reachable without login → confirm omnigent has NO public port and Caddy points at
  oauth2-proxy, not `omnigent:8000` directly.
- Runner can't reach server → it must target `127.0.0.1:8000`.
- Subscription billed as API → `*_API_KEY` unset in the runner's env.
- GHCR pull denied → `docker login ghcr.io` or `--build`.

## Success criteria (definition of done)

1. `https://YOUR-DOMAIN` from a phone browser forces GitHub login; a non-allowlisted
   account is rejected.
2. After login the UI loads and a session can start on this host.
3. A trivial task (create `/tmp/hello.txt`) runs on this box via the **subscription** route.
4. From an external network only 443/80/22 are reachable; `8000` is not.
5. No secrets in git; if any admin/cookie secrets exist, they're stored safely.

## Open questions to resolve during implementation

- Exact CLI verb to register/run the local runner (`omnigent run --server` vs `omnigent
  host`) — confirm via `--help`.
- Whether the GHCR server image is public yet or needs auth/local build.
