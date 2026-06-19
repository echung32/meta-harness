# Design: Stand up & secure the Omnigent control plane (sub-project A)

- **Date:** 2026-06-19 (rev 3 — **as-built**: Cloudflare Tunnel edge)
- **Status:** Implemented & verified (pending the final UI smoke test)
- **Source of truth for goals:** [PLAN.md](../../../PLAN.md)
- **Discovered facts:** [deploy/DISCOVERED.md](../../../deploy/DISCOVERED.md)
- **Implementation plan:** [../plans/2026-06-19-omnigent-standup-secure.md](../plans/2026-06-19-omnigent-standup-secure.md)

## Scope

Sub-project A only: bring up the Omnigent control plane on this single Ubuntu host and
secure its public web UI for a single operator. **Out of scope** (sub-project B): the
cross-vendor writer↔reviewer loop, Polly/bundle, Superpowers-in-subagent wiring, policies.

## Decisions (as-built)

| Decision | Choice | Rationale |
|---|---|---|
| Execution plane | **This host itself** (local runner via `omnigent host`) | No devcontainer; PLAN.md "don't double-sandbox" |
| Server location | **Co-located** on this box | Single operator |
| Edge / exposure | **Cloudflare Tunnel (`cloudflared`)** — outbound-only, zero inbound ports | Campus host (UH) with Cloudflare-proxied DNS; sidesteps inbound firewall, hides origin IP, TLS at CF edge |
| Front identity gate | **oauth2-proxy**, GitHub OAuth (`--github-user`) | Single-account allowlist; native Omnigent OIDC restricts by domain only |
| Inner auth | **`OMNIGENT_AUTH_ENABLED=0`** (local mode) | Single operator; gate is the sole identity check; loopback runner works with no header plumbing |
| Compose | **Official base compose** + a gate override (oauth2-proxy + cloudflared) | Don't reinvent the server/DB compose |
| Server port | **8000** | Real Omnigent port |
| Node | **Volta**-managed Node 22 | Omnigent + harnesses need Node 22 |

## Host baseline (verified 2026-06-19)

Ubuntu 24.04.4, Docker 29.5.3 + compose v5.1.4, Python 3.12.3, tmux 3.4, Node 22.23.0
(Volta), `claude` + `codex` logged in on **subscriptions** (Claude `claudeAiOauth`; Codex
`auth_mode=chatgpt`), no `*_API_KEY` in env.

## Architecture (as-built)

No inbound ports. `cloudflared` dials out to Cloudflare; the only host-published port is the
loopback runner path. Omnigent runs without its own login (`AUTH_ENABLED=0`).

```
   Internet                         ┌──────────── this Ubuntu box (ics-gpu-1) ───────────┐
                                    │  (UFW: deny all inbound except 22/SSH)             │
 Browser/phone ─https─▶ Cloudflare edge ◀═outbound tunnel═ cloudflared                   │
 (GitHub login,         (TLS, cf-ray)                          │                          │
  cookie)                                                      ▼                          │
                                    │                      oauth2-proxy ──▶ omnigent:8000 │
                                    │                   (GitHub OAuth,    (docker net,     │
                                    │                    --github-user=    AUTH_ENABLED=0) │
                                    │                     echung32)            │           │
                                    │                                      postgres        │
                                    │   omnigent host daemon (runs as ethan):  (docker net)│
                                    │     └─ 127.0.0.1:8000  →  the single "local" user    │
                                    │     └─ spawns claude/codex via ~/.claude + ~/.codex  │
                                    └─────────────────────────────────────────────────────┘
```

### Two entry paths (key structural decision)

- **Human path** — `browser → Cloudflare edge (TLS) → cloudflared → oauth2-proxy (GitHub
  OAuth, single-account) → omnigent:8000`. oauth2-proxy is a full reverse proxy
  (`--upstream=http://omnigent:8000`); unauthenticated requests never pass it.
- **Machine path** — the local `omnigent host` daemon connects to **`127.0.0.1:8000`**. With
  `AUTH_ENABLED=0` the server treats every request as the single `local` user, so the
  loopback runner needs no credential. Safe because that port is bound to `127.0.0.1` and
  UFW denies inbound — only local processes reach it.

Both the gated browser user and the loopback runner resolve to the **same single operator**.

### Defense layers (outermost → innermost)

1. **No inbound surface** — `cloudflared` is outbound-only; UFW denies all inbound except 22.
2. **Cloudflare edge** — TLS termination; hides origin IP; optional WAF.
3. **oauth2-proxy** — GitHub OAuth; `--github-user` = your account(s) (comma-separated
   allows your own personal + work logins — all still resolve to the one `local` Omnigent
   user); strips client-supplied `X-Forwarded-*`.
4. **Network isolation** — omnigent/postgres/oauth2-proxy have no host port; omnigent's only
   host port is `127.0.0.1:8000`. `AUTH_ENABLED=0` is acceptable *only* because of layers 1–3
   + this binding.

## Components (as-built)

**Reused from the cloned repo** (`~/omnigent/deploy/docker/`, pinned `918c153`):
- `docker-compose.yaml` (services `omnigent` + `postgres`). The Caddy `https` overlay is
  **not** used in this design.

**Authored in this repo** (`/home/ethan/meta-harness/deploy/`):
- `docker-compose.gate.yaml` — override that (a) sets `OMNIGENT_AUTH_ENABLED=0`, (b) re-adds
  `127.0.0.1:8000:8000` for the loopback runner, (c) adds the `oauth2-proxy` service (GitHub
  gate), (d) adds the `cloudflared` service (tunnel connector).
- `.env` — secrets + `OMNIGENT_DOMAIN`, `OAUTH2_PROXY_*`, `CF_TUNNEL_TOKEN` (gitignored).
- `.gitignore` — `.env`.
- `DISCOVERED.md` — pinned values + auth-model notes.

**Host-level:** Node 22 (Volta); `omnigent` CLI (uv tool, web UI skipped) + `omnigent host`
daemon run as `ethan`, inheriting `~/.claude` + `~/.codex`.

**Run command:**
```
docker compose --env-file deploy/.env \
  -f ~/omnigent/deploy/docker/docker-compose.yaml \
  -f deploy/docker-compose.gate.yaml up -d
```

## Bring-up sequence (as-built)

1. Node 22 via Volta.
2. Pin the cloned repo (`918c153`).
3. GitHub OAuth App (callback `https://agents.ethanchung.dev/oauth2/callback`); Cloudflare
   Tunnel created → public hostname → `http://oauth2-proxy:4180`; token in `.env`.
4. Author `deploy/` gate files + `.env`.
5. `docker compose ... up -d`; verify localhost-only `8000`, cloudflared connections
   registered, gated public path serves the sign-in page.
6. UFW default-deny inbound; allow only 22.
7. Install `omnigent` CLI (uv, `OMNIGENT_SKIP_WEB_UI=true`); `omnigent host
   http://127.0.0.1:8000` (backgrounded).
8. Creds sanity: no `*_API_KEY`; Claude + Codex on subscriptions.
9. Smoke test from phone browser.

## Error handling / known failure modes

- cloudflared not connecting → check `CF_TUNNEL_TOKEN`, outbound egress, `logs cloudflared`.
- Public hostname 502 → CF tunnel public-hostname service must be `http://oauth2-proxy:4180`.
- App reachable without login → confirm cloudflared routes to oauth2-proxy (not omnigent),
  and omnigent has no public port.
- Runner can't reach server → must target `127.0.0.1:8000`; daemon needs Volta Node on PATH.
- Subscription billed as API → `*_API_KEY` unset in the daemon env; Codex `auth_mode=chatgpt`.

## Success criteria

1. `https://agents.ethanchung.dev` forces GitHub login; non-allowlisted accounts rejected. ✅ (gate page verified)
2. After login the UI loads and a session can start on this host. ⏳ (UI smoke test)
3. A trivial task (create `/tmp/hello.txt`) runs on this box via the **subscription** route. ⏳
4. No inbound ports reachable from the internet (only 22/SSH allowed; `8000` loopback). ✅
5. No secrets in git. ✅

## Open questions / follow-ups

- **Daemon persistence:** the `omnigent host` daemon is currently `nohup`-backgrounded —
  survives logout but not reboot. Promote to a systemd user service for durability.
- **Version pinning:** server image is `:latest` and the CLI is git-main — pin both to a
  release tag for reproducibility once alpha stabilizes.
