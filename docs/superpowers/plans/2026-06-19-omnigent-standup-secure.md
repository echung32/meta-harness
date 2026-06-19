# Omnigent Stand-up & Secure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the Omnigent control plane co-located on this Ubuntu host and lock its public web UI behind a self-hosted identity gate, for a single operator.

**Architecture:** One Docker Compose stack (Caddy + oauth2-proxy + omnigent-server + postgres) where only Caddy is published to the internet; Caddy uses `forward_auth` → oauth2-proxy (Google OIDC, single-email allowlist) before reaching the server. The Omnigent host daemon runs on the host as `ethan`, connects to the server over `127.0.0.1:6767` (token auth, bypassing the browser gate), and spawns claude/codex subprocesses using the host's subscription credentials.

**Tech Stack:** Docker Compose, Caddy, oauth2-proxy, Postgres, Omnigent (alpha), Volta/Node 22, Google OIDC, UFW.

**Spec:** [docs/superpowers/specs/2026-06-19-omnigent-standup-secure-design.md](../specs/2026-06-19-omnigent-standup-secure-design.md)

## Global Constraints

- **One operator only.** oauth2-proxy + Omnigent allow exactly one email; no self-signup.
- **Only Caddy is published to `0.0.0.0`** (ports 80/443). Every other service is on the internal Docker network. The Omnigent server's single host-published port is `127.0.0.1:6767` (localhost only).
- **No secrets in git.** `.env`, `*.credentials*`, and Caddy/oauth2-proxy data are gitignored.
- **Subscription route only.** `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` must be unset in the host daemon's environment.
- **Pin the alpha version.** Check out a fixed Omnigent commit/tag; do not track `main`.
- **Operator parameters** (set once, used throughout): export `DOMAIN=<your-domain>` and `EMAIL=ethan.chung8192@gmail.com` in your shell before running commands. Replace these in config files written below.
- **Working tree:** all stack files live under `deploy/` in this repo (`/home/ethan/meta-harness`).

---

### Task 1: Node 22 via Volta

**Files:** none (host toolchain).

**Interfaces:**
- Produces: `node` v22 on PATH for the omnigent CLI/daemon and claude/codex subprocesses.

- [ ] **Step 1: Verify the current (wrong) version**

Run: `node -v`
Expected: `v18.19.1` (this is what we're fixing).

- [ ] **Step 2: Install Node 22 with Volta**

```bash
volta install node@22
```

- [ ] **Step 3: Verify Node 22 is active**

Run: `node -v && which node`
Expected: `v22.x.x` and a path under `~/.volta/`.

- [ ] **Step 4: Commit** (nothing to commit — host change only; record in notes and move on).

---

### Task 2: Clone & pin Omnigent, discover compose base + env names

**Files:**
- Create: `~/omnigent/` (clone target, outside the repo)
- Create: `deploy/DISCOVERED.md` (record of pinned values)

**Interfaces:**
- Produces: `OMNIGENT_REF` (pinned commit), `SERVER_SERVICE` (compose service name for the server), and the exact env var names used by the server's compose — all recorded in `deploy/DISCOVERED.md` and consumed by Tasks 4–5.

- [ ] **Step 1: Clone and pin a fixed ref**

```bash
git clone https://github.com/omnigent-ai/omnigent.git ~/omnigent
cd ~/omnigent
git log --oneline -1          # record this SHA as OMNIGENT_REF
git checkout <OMNIGENT_REF>   # detach at the pinned SHA
```
Expected: HEAD detached at the recorded SHA.

- [ ] **Step 2: Read the deploy compose + bootstrap**

Run: `sed -n '1,200p' ~/omnigent/deploy/docker/docker-compose.yml; ls ~/omnigent/deploy/docker/`
Expected: you can see the server service block, the postgres service, and `bootstrap.sh`.

- [ ] **Step 3: Record discovered values**

Write `deploy/DISCOVERED.md` with the exact, observed values:
```markdown
# Discovered (pinned <OMNIGENT_REF>)
- OMNIGENT_REF: <sha>
- SERVER_SERVICE: <service name of the omnigent server in compose, e.g. "omnigent">
- SERVER_INTERNAL_PORT: <container port, expected 6767>
- DB env var: <e.g. DATABASE_URL>
- Auth env var: <e.g. OMNIGENT_AUTH_ENABLED=1>
- OIDC cookie secret env var: <e.g. OMNIGENT_OIDC_COOKIE_SECRET>
- Admin init env var: <e.g. OMNIGENT_ACCOUNTS_INIT_ADMIN_PASSWORD>
- CLI verbs: <e.g. `omnigent login` / `omnigent host`>  (from `omnigent --help` once installed)
```

- [ ] **Step 4: Commit**

```bash
cd /home/ethan/meta-harness
git add deploy/DISCOVERED.md
git commit -m "chore: pin Omnigent ref and record discovered deploy values"
```

---

### Task 3: Google OAuth client + DNS A record

**Files:** none (external consoles).

**Interfaces:**
- Produces: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` (used by oauth2-proxy in Task 5); a DNS A record so Caddy can issue a cert in Task 5.

- [ ] **Step 1: Create the OAuth client**

In Google Cloud Console → APIs & Services → Credentials → Create OAuth client ID → Web application.
- Authorized redirect URI: `https://YOUR-DOMAIN/oauth2/callback` (substitute `$DOMAIN`).
Record the client ID and secret.

- [ ] **Step 2: Point DNS at the host**

Create an `A` record: `YOUR-DOMAIN` → this host's public IP.

- [ ] **Step 3: Verify DNS resolves to the public IP**

Run: `dig +short $DOMAIN`
Expected: the host's public IP (allow for propagation delay).

- [ ] **Step 4: Commit** (nothing to commit — external setup; note client ID location).

---

### Task 4: Compose core — postgres + omnigent-server (localhost-only), secrets

**Files:**
- Create: `deploy/docker-compose.yml`
- Create: `deploy/.env` (gitignored)
- Create: `deploy/.gitignore`
- Modify: `.gitignore` (repo root)

**Interfaces:**
- Consumes: `SERVER_SERVICE`, env var names from `deploy/DISCOVERED.md` (Task 2).
- Produces: an omnigent-server reachable at `127.0.0.1:6767`; an internal Docker network `omninet` that Task 5 attaches to.

- [ ] **Step 1: Write the gitignore (test that secrets stay untracked)**

`deploy/.gitignore`:
```
.env
*.credentials*
caddy_data/
caddy_config/
emails.txt
```
Also append to repo-root `/home/ethan/meta-harness/.gitignore`:
```
deploy/.env
deploy/emails.txt
deploy/caddy_data/
deploy/caddy_config/
```

- [ ] **Step 2: Generate secrets into `deploy/.env`**

```bash
cd /home/ethan/meta-harness/deploy
{
  echo "POSTGRES_PASSWORD=$(python3 -c 'import secrets;print(secrets.token_urlsafe(32))')"
  echo "OMNIGENT_OIDC_COOKIE_SECRET=$(python3 -c 'import secrets;print(secrets.token_urlsafe(32))')"
  echo "OMNIGENT_ACCOUNTS_INIT_ADMIN_PASSWORD=$(python3 -c 'import secrets;print(secrets.token_urlsafe(24))')"
} >> .env
chmod 600 .env
```
(Use the exact env var names recorded in `DISCOVERED.md`; adjust if they differ.)

- [ ] **Step 3: Write `deploy/docker-compose.yml` core services**

```yaml
name: omnigent-stack
networks:
  omninet:
    driver: bridge
volumes:
  pgdata:
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: omnigent
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: omnigent
    volumes: [pgdata:/var/lib/postgresql/data]
    networks: [omninet]
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U omnigent"]
      interval: 5s
      timeout: 3s
      retries: 10
  omnigent-server:
    image: ghcr.io/omnigent-ai/omnigent:<OMNIGENT_REF>   # or build: from ~/omnigent per DISCOVERED.md
    depends_on:
      postgres: { condition: service_healthy }
    environment:
      DATABASE_URL: postgresql://omnigent:${POSTGRES_PASSWORD}@postgres:5432/omnigent
      OMNIGENT_AUTH_ENABLED: "1"
      OMNIGENT_OIDC_COOKIE_SECRET: ${OMNIGENT_OIDC_COOKIE_SECRET}
      OMNIGENT_ACCOUNTS_INIT_ADMIN_PASSWORD: ${OMNIGENT_ACCOUNTS_INIT_ADMIN_PASSWORD}
    ports:
      - "127.0.0.1:6767:6767"   # localhost-only; the machine path
    networks: [omninet]
    restart: unless-stopped
```
(Reconcile image/build, service name, and env keys against `DISCOVERED.md`.)

- [ ] **Step 4: Bring up the core and verify it's localhost-only**

```bash
cd /home/ethan/meta-harness/deploy
docker compose --env-file .env up -d postgres omnigent-server
docker compose ps
ss -ltnp | grep 6767
```
Expected: `ss` shows `127.0.0.1:6767` bound (NOT `0.0.0.0:6767`).

- [ ] **Step 5: Verify the server answers on localhost**

Run: `curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:6767/`
Expected: a `200`/`302`/`needs_setup` response (not connection-refused).

- [ ] **Step 6: Commit**

```bash
cd /home/ethan/meta-harness
git add deploy/docker-compose.yml deploy/.gitignore .gitignore
git status --short   # confirm deploy/.env is NOT listed
git commit -m "feat: omnigent core compose (postgres + localhost-only server)"
```

---

### Task 5: Edge — Caddy (TLS) + oauth2-proxy (Google OIDC, single-email gate)

**Files:**
- Create: `deploy/Caddyfile`
- Create: `deploy/emails.txt` (gitignored — single email)
- Modify: `deploy/docker-compose.yml` (add `caddy` + `oauth2-proxy` services)
- Modify: `deploy/.env` (add Google + oauth2-proxy secrets)

**Interfaces:**
- Consumes: `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET` (Task 3), `omnigent-server` on `omninet` (Task 4).
- Produces: public HTTPS endpoint at `https://$DOMAIN` gated by Google login.

- [ ] **Step 1: Add edge secrets to `deploy/.env`**

```bash
cd /home/ethan/meta-harness/deploy
{
  echo "GOOGLE_CLIENT_ID=<from Task 3>"
  echo "GOOGLE_CLIENT_SECRET=<from Task 3>"
  echo "OAUTH2_PROXY_COOKIE_SECRET=$(python3 -c 'import secrets,base64;print(base64.urlsafe_b64encode(secrets.token_bytes(32)).decode())')"
  echo "DOMAIN=<your-domain>"
} >> .env
echo "ethan.chung8192@gmail.com" > emails.txt
chmod 600 .env emails.txt
```

- [ ] **Step 2: Write `deploy/Caddyfile` (forward_auth pattern)**

```
{$DOMAIN} {
    handle /oauth2/* {
        reverse_proxy oauth2-proxy:4180
    }
    handle {
        forward_auth oauth2-proxy:4180 {
            uri /oauth2/auth
            copy_headers X-Auth-Request-User X-Auth-Request-Email
            @error status 401
            handle_response @error {
                redir * /oauth2/sign_in?rd={scheme}://{host}{uri}
            }
        }
        reverse_proxy omnigent-server:6767
    }
}
```

- [ ] **Step 3: Add `caddy` + `oauth2-proxy` services to `deploy/docker-compose.yml`**

```yaml
  oauth2-proxy:
    image: quay.io/oauth2-proxy/oauth2-proxy:v7.6.0
    command:
      - --provider=google
      - --http-address=0.0.0.0:4180
      - --reverse-proxy=true
      - --set-xauthrequest=true
      - --upstream=static://200
      - --email-domain=*
      - --authenticated-emails-file=/etc/oauth2-proxy/emails.txt
      - --redirect-url=https://${DOMAIN}/oauth2/callback
      - --cookie-secure=true
      - --cookie-domain=${DOMAIN}
      - --whitelist-domain=${DOMAIN}
    environment:
      OAUTH2_PROXY_CLIENT_ID: ${GOOGLE_CLIENT_ID}
      OAUTH2_PROXY_CLIENT_SECRET: ${GOOGLE_CLIENT_SECRET}
      OAUTH2_PROXY_COOKIE_SECRET: ${OAUTH2_PROXY_COOKIE_SECRET}
    volumes:
      - ./emails.txt:/etc/oauth2-proxy/emails.txt:ro
    networks: [omninet]
    restart: unless-stopped
  caddy:
    image: caddy:2
    depends_on: [oauth2-proxy, omnigent-server]
    ports:
      - "80:80"
      - "443:443"
    environment:
      DOMAIN: ${DOMAIN}
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./caddy_data:/data
      - ./caddy_config:/config
    networks: [omninet]
    restart: unless-stopped
```

- [ ] **Step 4: Bring up the full stack**

```bash
cd /home/ethan/meta-harness/deploy
docker compose --env-file .env up -d
docker compose ps
docker compose logs caddy | grep -i certificate
```
Expected: all services `running`/`healthy`; Caddy logs show a successfully obtained certificate for `$DOMAIN`.

- [ ] **Step 5: Verify unauthenticated requests are redirected to Google login**

Run: `curl -sS -o /dev/null -w '%{http_code} %{redirect_url}\n' https://$DOMAIN/`
Expected: `302` with a redirect to `/oauth2/sign_in` (or onward to `accounts.google.com`) — NOT a `200` Omnigent page.

- [ ] **Step 6: Commit**

```bash
cd /home/ethan/meta-harness
git add deploy/Caddyfile deploy/docker-compose.yml
git status --short   # confirm deploy/.env and deploy/emails.txt are NOT listed
git commit -m "feat: Caddy TLS + oauth2-proxy Google gate (single-email allowlist)"
```

---

### Task 6: Host firewall (UFW)

**Files:** none (host firewall state).

**Interfaces:**
- Consumes: published ports from Tasks 4–5.
- Produces: only 22/80/443 reachable externally; 6767 unreachable from outside.

- [ ] **Step 1: Confirm current exposure before locking down**

Run: `sudo ufw status`
Expected: `inactive` (or a permissive state) — the thing we're fixing.

- [ ] **Step 2: Apply default-deny inbound + allow 22/80/443**

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
```

- [ ] **Step 3: Verify the ruleset**

Run: `sudo ufw status verbose`
Expected: default deny incoming; only 22/80/443 allowed; 6767 absent.

- [ ] **Step 4: Verify 6767 is not externally reachable**

From a DIFFERENT machine/network: `nc -vz -w3 $DOMAIN 6767`
Expected: timeout/refused. (And `nc -vz $DOMAIN 443` succeeds.)

- [ ] **Step 5: Commit** (nothing to commit — host firewall; record in notes).

---

### Task 7: Install omnigent CLI on host + register this host as executor

**Files:** none (host CLI + daemon).

**Interfaces:**
- Consumes: omnigent-server on `127.0.0.1:6767` (Task 4); CLI verbs from `DISCOVERED.md`.
- Produces: this host registered and selectable as an execution target in the UI.

- [ ] **Step 1: Install the CLI on the host**

```bash
curl -fsSL https://raw.githubusercontent.com/omnigent-ai/omnigent/main/scripts/install_oss.sh | sh
omnigent --help   # confirm verbs; reconcile with DISCOVERED.md (omnigent vs omni)
```
Expected: `omnigent` on PATH; help lists `login` and `host`.

- [ ] **Step 2: Log in and register this host against the localhost server**

```bash
omnigent login http://127.0.0.1:6767     # use the admin password from deploy/.env
omnigent host  http://127.0.0.1:6767     # registers THIS machine
```
Expected: login stores a token; host command reports the host registered/online.

- [ ] **Step 3: Verify the host appears in the server**

Run (after logging into the UI): the host shows as online in the UI host list. CLI check: `omnigent host --status` (or equivalent per `--help`).
Expected: this host listed as online.

- [ ] **Step 4: Commit** (nothing to commit — host registration; record in notes).

---

### Task 8: Credentials sanity (subscription route, not API)

**Files:** none.

**Interfaces:**
- Consumes: the running host daemon (Task 7).
- Produces: confirmation that agents bill the subscription, not API keys.

- [ ] **Step 1: Assert no API-key env vars in the daemon's environment**

Run: `env | grep -E 'ANTHROPIC_API_KEY|OPENAI_API_KEY' || echo CLEAN`
Expected: `CLEAN`.

- [ ] **Step 2: Confirm Claude is on the subscription route**

Run: `claude /status`
Expected: subscription/plan indicated, not "API key".

- [ ] **Step 3: Confirm Codex is logged in**

Run: `codex` (check auth/status) — confirm ChatGPT-plan login, then exit.
Expected: logged in via ChatGPT plan.

- [ ] **Step 4: Commit** (nothing to commit — verification only; record in notes).

---

### Task 9: End-to-end smoke test (definition of done)

**Files:** none (produces `/tmp/hello.txt` on this host as proof).

**Interfaces:**
- Consumes: everything from Tasks 1–8.
- Produces: signed-off success criteria from the spec.

- [ ] **Step 1: From your PHONE browser, hit the UI**

Navigate to `https://$DOMAIN`.
Expected: forced Google login.

- [ ] **Step 2: Verify the allowlist rejects other accounts**

Attempt login with a NON-allowlisted Google account.
Expected: access denied by oauth2-proxy.

- [ ] **Step 3: Log in with the allowlisted account; UI loads**

Expected: Omnigent UI renders; this host is selectable.

- [ ] **Step 4: Run a trivial task on this host**

Start a session on this host; instruct the agent: "create the file /tmp/hello.txt containing the word omnigent".

- [ ] **Step 5: Verify it executed on THIS box via the subscription route**

On the host: `cat /tmp/hello.txt`
Expected: `omnigent`. (Cross-check the session used the subscription route, not an API key.)

- [ ] **Step 6: Final external port check**

From a different network: confirm only 443/80/22 reachable, 6767 closed (re-run Task 6 Step 4).

- [ ] **Step 7: Rotate the admin password & finalize**

Log into the UI as admin once, rotate the password, store it in your password manager. Confirm `git status` shows no secrets tracked.

- [ ] **Step 8: Commit the runbook notes**

```bash
cd /home/ethan/meta-harness
git add deploy/DISCOVERED.md
git commit -m "docs: record final pinned values and smoke-test results"
```

---

## Done means

All five spec success criteria pass: (1) public URL forces Google login and rejects non-allowlisted accounts; (2) UI loads and a session starts on this host; (3) a trivial task runs on this box via the subscription route; (4) only 443/80/22 reachable externally, 6767 closed; (5) no secrets in git, admin password rotated.

**Deferred to sub-project B:** cross-vendor writer↔reviewer loop, Polly/bundle, Superpowers-in-subagent wiring, cost/ask-on-shell policies.
