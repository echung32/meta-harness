# Design: Stand up & secure the Omnigent control plane (sub-project A)

- **Date:** 2026-06-19
- **Status:** Approved design — pending spec review, then implementation planning
- **Source of truth for goals:** [PLAN.md](../../../PLAN.md); ops detail in [SETUP.md](../../../SETUP.md)
- **Method:** Superpowers brainstorming (6.0.3)

## Scope

This spec covers **sub-project A only**: bringing up the Omnigent control plane on this
single Ubuntu host and securing its public-facing web UI for a single operator.

**Out of scope** (deferred to sub-project B, its own spec → plan → implement cycle):
the cross-vendor writer↔reviewer loop, Polly/custom bundle, Superpowers wiring inside
sub-agents, and cost/ask-on-shell policies.

## Decisions (locked during brainstorming)

| Decision | Choice | Rationale |
|---|---|---|
| Execution plane | **This host itself**, registered via `omnigent host` | No devcontainer for now; matches PLAN.md "don't double-sandbox" |
| Server location | **Co-located** on this box | Single operator; simplest one-host runbook |
| Exposure | **Public IP + Caddy reverse proxy** on a domain | User has a public IP and a domain |
| Identity gate | **Self-hosted oauth2-proxy** (Google OIDC) in front | No third party in the request path |
| Users | **Just the operator** (you@gmail) | Subscription auth is ToS-fine for solo use |
| Edge assembly | **Single Docker Compose stack**; only Caddy published | Cleanest "what is exposed?" answer |
| Node | **Volta**-managed Node 22 | Volta already installed; Omnigent needs Node 22 LTS |

## Host baseline (verified 2026-06-19)

Ready: Ubuntu 24.04.4, Docker 29.5.3 + compose v5.1.4, Python 3.12.3, tmux 3.4,
`claude` + `codex` CLIs logged in (creds mode 0600), no `ANTHROPIC_API_KEY` /
`OPENAI_API_KEY` set. **Gap:** Node is v18 — upgrade to 22 via Volta.

## Architecture

Two deliberate entry paths into the server; only the human path is browser-gated.

```
                       ┌─────────────── this Ubuntu box (public IP) ───────────────┐
 Browser/phone ─443─▶ Caddy ──forward_auth──▶ oauth2-proxy (allowlist: you@gmail) │
 (Google login,      (TLS)  └──(if 2xx)──▶ omnigent-server (internal net) ──┐     │
  cookie)                                                              postgres    │
                       │                                            (internal net) │
                       │  omnigent host daemon (runs as ethan on host)             │
                       │    └─ connects to 127.0.0.1:6767 (token auth; bypasses    │
                       │       the browser gate) + opens outbound tunnel           │
                       │    └─ spawns claude/codex subprocesses using              │
                       │       ~/.claude + ~/.codex subscription creds             │
                       └─────────────────────────────────────────────────────────┘
```

### Two entry paths (the key structural decision)

- **Human path** — `browser → Caddy(TLS) → [forward_auth check → oauth2-proxy] →
  omnigent-server`. Caddy is the sole reverse proxy; it uses **`forward_auth`** to ask
  oauth2-proxy to authorize each request (and routes `/oauth2/*` login endpoints to
  oauth2-proxy). oauth2-proxy enforces Google OIDC + single-email allowlist. Only after
  it returns 2xx does Caddy proxy through to omnigent-server. Unauthenticated requests
  never reach the app. (oauth2-proxy runs in forward-auth mode, not inline-upstream mode.)
- **Machine path** — the local host daemon and `omnigent` CLI connect to
  **`127.0.0.1:6767`** using Omnigent's own bearer token. They cannot use the browser
  OAuth flow, so they bypass oauth2-proxy. Safe because the port is bound localhost-only
  and firewalled, and the token still authenticates the connection.

### Defense layers (outermost → innermost)

1. **UFW** — default-deny inbound; allow only 22/80/443.
2. **Caddy** — TLS termination (Let's Encrypt HTTP-01); only service published to `0.0.0.0`.
3. **oauth2-proxy** — Google OIDC, `OAUTH2_PROXY_AUTHENTICATED_EMAILS` = your single email.
4. **Omnigent auth** — `OMNIGENT_AUTH_ENABLED=1` + its own OIDC; admin account only, no
   open self-signup.
5. (Sub-project B) Omnigent **policies** as a runtime backstop.

A flaw in any single layer does not open the door by itself.

## Components & file inventory

**Compose services** (base on Omnigent's `deploy/docker/`, extended):

| Service | Published port | Reachable from internet? |
|---|---|---|
| `caddy` | `0.0.0.0:80`, `0.0.0.0:443` | Yes — the only public surface |
| `oauth2-proxy` | none (internal net) | No |
| `omnigent-server` | `127.0.0.1:6767` (localhost only) | No |
| `postgres` | none (internal net) | No |

**Host-level (not in Compose):**
- Node 22 via Volta; Python 3.12 (present); tmux (present).
- `omnigent` CLI + host daemon, run as `ethan`, registered against `http://127.0.0.1:6767`,
  inheriting `~/.claude` + `~/.codex`.

**Secrets — generated locally, never committed:**
- `deploy/.env`: Postgres password, `OMNIGENT_OIDC_COOKIE_SECRET`, oauth2-proxy
  `OAUTH2_PROXY_COOKIE_SECRET` (32-byte), Google `client_id`/`client_secret`, allowlisted
  email, `OMNIGENT_ACCOUNTS_INIT_ADMIN_PASSWORD`.
- `.gitignore`: `.env`, `*.credentials*`, Caddy data/volumes.
- `bootstrap.sh` generates what it can; we add the oauth2-proxy + Google values.

**Alpha caveat:** exact env var names, the compose base, and CLI verb spellings
(`omnigent` vs `omni`, `server start` vs `run`) are pinned by reading the cloned
`deploy/docker/` and `--help`, **not** trusted from doc pages (which drift).

## Bring-up sequence (each step verified before the next)

1. Node 22 via Volta; confirm `node -v`.
2. Clone `omnigent-ai/omnigent`; **pin a commit/tag** (alpha — avoid surprise breakage).
3. Google OAuth client; redirect `https://YOUR-DOMAIN/oauth2/callback`.
4. DNS A record → public IP; confirm propagation.
5. Author Compose stack + `Caddyfile` + oauth2-proxy config; run `bootstrap.sh`; fill `.env`.
6. `docker compose up -d`; complete `needs_setup` / set admin password.
7. UFW default-deny inbound; allow 22/80/443; verify 6767 unreachable externally.
8. Install `omnigent` CLI on host; `omnigent login` + `omnigent host` → `127.0.0.1:6767`.
9. Creds sanity: assert no `*_API_KEY` in the daemon env; `claude /status` = subscription.
10. Smoke test from phone browser.

## Error handling / known failure modes

- Caddy cert issuance fails → check DNS + port 80 open for ACME (HTTP-01).
- oauth2-proxy redirect loop → cookie-domain / whitelist-domain must match `YOUR-DOMAIN`.
- Daemon can't reach server → it must target `127.0.0.1:6767`, not the gated public URL.
- Subscription silently billed as API → `*_API_KEY` must be unset in the daemon's env.
- Omnigent CLI/server verb mismatch → resolve via `--help` against the pinned version.

## Success criteria (definition of done for sub-project A)

1. `https://YOUR-DOMAIN` from a phone browser forces Google login; a non-allowlisted
   account is rejected.
2. After login the Omnigent UI loads and a session can start on this host.
3. A trivial task (create `/tmp/hello.txt`) executes on this box via the **subscription**
   route, not an API key.
4. From an external network, only 443/80/22 are reachable; `6767` is not.
5. No secrets in git; admin password rotated after first login.

## Open questions to resolve during implementation

- Exact Compose base + env var names for the pinned Omnigent version.
- Whether the host daemon registers cleanly against a localhost server URL (expected: yes,
  outbound tunnel + token).
- Confirm oauth2-proxy cookie flow works from a mobile browser (expected: yes, standard
  cookie session).
