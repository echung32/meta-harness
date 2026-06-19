# Omnigent Stand-up & Secure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans (inline) to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Rev 2** — rebased on the real Omnigent deploy after Task 2 discovery. Port is **8000**; we build on the **shipped** compose + a gate override; Omnigent runs in **local mode** (`AUTH_ENABLED=0`) behind oauth2-proxy. See [DISCOVERED.md](../../../deploy/DISCOVERED.md).

> ⚠️ **AS-BUILT (rev 3): the edge changed from Caddy to a Cloudflare Tunnel.** The campus host has Cloudflare-proxied DNS, so we use `cloudflared` (outbound-only, zero inbound ports) → oauth2-proxy → omnigent, and dropped Caddy + the HTTPS overlay. Tasks 4–6 below still describe the original Caddy path; the **authoritative as-built design is the [spec rev 3](../specs/2026-06-19-omnigent-standup-secure-design.md)** and the real files in `deploy/`. Auth gate is **GitHub** (not Google).

**Goal:** Stand up the Omnigent control plane co-located on this Ubuntu host and lock its public web UI behind a single-email oauth2-proxy gate, for a single operator.

**Architecture:** Reuse the official `docker-compose.yaml` + `docker-compose.https.yaml` (Caddy auto-TLS); add `docker-compose.gate.yaml` that inserts oauth2-proxy as a full reverse proxy (`caddy → oauth2-proxy → omnigent:8000`), sets `OMNIGENT_AUTH_ENABLED=0`, and re-adds a `127.0.0.1:8000` publish for the local runner. The runner connects on loopback as the `local` user; the browser path is gated by GitHub login restricted to one account.

**Tech Stack (as-built):** Docker Compose, **cloudflared (Cloudflare Tunnel)**, oauth2-proxy, Postgres, Omnigent (alpha, pinned), Volta/Node 22, GitHub OAuth, UFW.

**Spec:** [docs/superpowers/specs/2026-06-19-omnigent-standup-secure-design.md](../specs/2026-06-19-omnigent-standup-secure-design.md)

## Global Constraints

- **One operator only.** oauth2-proxy allowlist = exactly one email; Omnigent itself runs auth-off (`local` user).
- **No inbound ports** (as-built): cloudflared is outbound-only; UFW denies all inbound except 22. `omnigent`'s only host port is `127.0.0.1:8000` (loopback runner); `postgres`/`oauth2-proxy`/`cloudflared` publish nothing.
- **No secrets in git.** `deploy/.env` gitignored.
- **Subscription route only.** `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` unset in the runner env.
- **Pin the alpha version.** Detach the clone at a fixed SHA; pin `OMNIGENT_IMAGE_TAG`.
- **Operator parameters:** `DOMAIN` (your domain) and `EMAIL=ethan.chung8192@gmail.com`. Substitute in files below.
- **Paths:** repo = `/home/ethan/meta-harness`; clone = `/home/ethan/omnigent`.

---

### Task 1: Node 22 via Volta

**Files:** none (host toolchain).

- [ ] **Step 1: Verify current (wrong) version** — Run: `node -v` → Expected `v18.19.1`.
- [ ] **Step 2: Install** — `volta install node@22`
- [ ] **Step 3: Verify** — Run: `node -v && which node` → Expected `v22.x` under `~/.volta/`.

---

### Task 2: Pin the cloned Omnigent ref  *(discovery already done → DISCOVERED.md committed)*

**Files:** Modify `deploy/DISCOVERED.md` (record the detached SHA).

- [ ] **Step 1: Record + detach at a fixed SHA**

```bash
git -C ~/omnigent log --oneline -1     # record this SHA
git -C ~/omnigent checkout <SHA>       # detached HEAD at the pin
```
Expected: `HEAD detached at <SHA>`.

- [ ] **Step 2: Update DISCOVERED.md** with the pinned SHA in the `OMNIGENT_REF` line.
- [ ] **Step 3: Commit**

```bash
cd /home/ethan/meta-harness && git add deploy/DISCOVERED.md
git commit -m "chore: pin Omnigent ref for deploy"
```

---

### Task 3: GitHub OAuth App + DNS A record

**Files:** none (external consoles).

**Interfaces:** Produces `OAUTH2_PROXY_CLIENT_ID`, `OAUTH2_PROXY_CLIENT_SECRET` (Task 4); DNS so Caddy can issue a cert (Task 5).

- [ ] **Step 1: Create the OAuth App** — github.com/settings/developers → OAuth Apps → New OAuth App. Homepage `https://YOUR-DOMAIN`; **Authorization callback URL** `https://YOUR-DOMAIN/oauth2/callback`. Record the Client ID + generate a Client secret. Also note your GitHub **username** (for the `--github-user` allowlist).
- [ ] **Step 2: DNS** — `A` record `YOUR-DOMAIN` → this host's public IP.
- [ ] **Step 3: Verify** — Run: `dig +short $DOMAIN` → Expected: the host's public IP.

---

### Task 4: Author the gate stack (`deploy/`)

**Files:**
- Create: `deploy/docker-compose.gate.yaml`, `deploy/Caddyfile`, `deploy/.env`
- Modify: `.gitignore` (repo root)

**Interfaces:** Consumes Task 3 creds + DISCOVERED service names. Produces a runnable stack definition for Task 5.

- [ ] **Step 1: gitignore (verify secrets stay untracked)**

Append to `/home/ethan/meta-harness/.gitignore`:
```
deploy/.env
deploy/caddy_data/
deploy/caddy_config/
```

- [ ] **Step 2: Generate `deploy/.env`**

```bash
cd /home/ethan/meta-harness/deploy
cat > .env <<EOF
POSTGRES_PASSWORD=$(python3 -c 'import secrets;print(secrets.token_urlsafe(32))')
OMNIGENT_DOMAIN=YOUR-DOMAIN
OMNIGENT_AUTH_ENABLED=0
OMNIGENT_IMAGE_TAG=latest
OAUTH2_PROXY_CLIENT_ID=<from Task 3>
OAUTH2_PROXY_CLIENT_SECRET=<from Task 3>
OAUTH2_PROXY_GITHUB_USER=<your-github-username>
OAUTH2_PROXY_COOKIE_SECRET=$(python3 -c 'import secrets,base64;print(base64.urlsafe_b64encode(secrets.token_bytes(32)).decode())')
EOF
chmod 600 .env
```
(Then edit `.env` to replace `YOUR-DOMAIN`, the two GitHub OAuth values, and your GitHub username.)

- [ ] **Step 3: `deploy/Caddyfile`**

```
{$OMNIGENT_DOMAIN} {
	encode zstd gzip
	reverse_proxy oauth2-proxy:4180
}
```

- [ ] **Step 4: `deploy/docker-compose.gate.yaml`**

```yaml
services:
  omnigent:
    environment:
      OMNIGENT_AUTH_ENABLED: "0"
    ports: !override
      - "127.0.0.1:8000:8000"   # loopback runner path only
  oauth2-proxy:
    image: quay.io/oauth2-proxy/oauth2-proxy:v7.6.0
    restart: unless-stopped
    depends_on: [omnigent]
    command:
      - --provider=github
      - --http-address=0.0.0.0:4180
      - --reverse-proxy=true
      - --upstream=http://omnigent:8000
      - --email-domain=*
      - --redirect-url=https://${OMNIGENT_DOMAIN}/oauth2/callback
      - --cookie-secure=true
      - --cookie-domain=${OMNIGENT_DOMAIN}
      - --whitelist-domain=${OMNIGENT_DOMAIN}
    environment:
      OAUTH2_PROXY_CLIENT_ID: ${OAUTH2_PROXY_CLIENT_ID}
      OAUTH2_PROXY_CLIENT_SECRET: ${OAUTH2_PROXY_CLIENT_SECRET}
      OAUTH2_PROXY_COOKIE_SECRET: ${OAUTH2_PROXY_COOKIE_SECRET}
      OAUTH2_PROXY_GITHUB_USER: ${OAUTH2_PROXY_GITHUB_USER}
  caddy:
    depends_on: [oauth2-proxy]
    volumes:
      - /home/ethan/meta-harness/deploy/Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy-data:/data
      - caddy-config:/config
```

- [ ] **Step 5: Commit (no secrets)**

```bash
cd /home/ethan/meta-harness
git add deploy/docker-compose.gate.yaml deploy/Caddyfile .gitignore
git status --short   # confirm deploy/.env is NOT listed
git commit -m "feat: oauth2-proxy gate override on the official Omnigent stack"
```

---

### Task 5: Bring up the stack & verify exposure + TLS

**Files:** none (runtime).

**Interfaces:** Consumes Task 4 files. Produces a running, gated server.

- [ ] **Step 1: (If GHCR private) authenticate the pull**

Run: `docker pull ghcr.io/omnigent-ai/omnigent-server:latest`
If denied: `echo $GHCR_TOKEN | docker login ghcr.io -u <user> --password-stdin` (token: `read:packages`), or add `--build` in Step 2.

- [ ] **Step 2: Up**

```bash
cd /home/ethan/meta-harness
docker compose --env-file deploy/.env \
  -f /home/ethan/omnigent/deploy/docker/docker-compose.yaml \
  -f /home/ethan/omnigent/deploy/docker/docker-compose.https.yaml \
  -f deploy/docker-compose.gate.yaml up -d
docker compose --env-file deploy/.env \
  -f /home/ethan/omnigent/deploy/docker/docker-compose.yaml \
  -f /home/ethan/omnigent/deploy/docker/docker-compose.https.yaml \
  -f deploy/docker-compose.gate.yaml ps
```
Expected: `postgres`, `omnigent`, `oauth2-proxy`, `caddy` all up; postgres healthy.

- [ ] **Step 3: Verify omnigent is localhost-only**

Run: `ss -ltnp | grep ':8000'`
Expected: `127.0.0.1:8000` bound (NOT `0.0.0.0:8000`).

- [ ] **Step 4: Verify the loopback app answers**

Run: `curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8000/`
Expected: `200`/`302` (server reachable on loopback).

- [ ] **Step 5: Verify Caddy got a cert**

Run: `docker compose ... logs caddy | grep -i "certificate obtained\|serving"` (use the same `-f` set)
Expected: a Let's Encrypt cert obtained for `$DOMAIN`.

- [ ] **Step 6: Verify the public path forces GitHub login**

Run: `curl -sS -o /dev/null -w '%{http_code} %{redirect_url}\n' -L --max-redirs 0 https://$DOMAIN/`
Expected: `302` toward oauth2-proxy `/oauth2/sign_in` or `github.com/login` — NOT a 200 app page.

---

### Task 6: Host firewall (UFW)

**Files:** none.

- [ ] **Step 1: Pre-state** — Run: `sudo ufw status` → Expected `inactive` (the thing we fix).
- [ ] **Step 2: Apply**

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
```

- [ ] **Step 3: Verify ruleset** — Run: `sudo ufw status verbose` → Expected: default deny in; only 22/80/443; no 8000.
- [ ] **Step 4: Verify 8000 closed externally** — From another network: `nc -vz -w3 $DOMAIN 8000` → Expected timeout/refused; `nc -vz $DOMAIN 443` → succeeds.

---

### Task 7: Install omnigent CLI + launch the local runner

**Files:** none (host CLI).

**Interfaces:** Consumes the running server (`127.0.0.1:8000`). Produces this host selectable in the UI.

- [ ] **Step 1: Install CLI**

```bash
curl -fsSL https://omnigent.ai/install.sh | sh   # or: uv tool install omnigent
omnigent --help                                  # confirm runner/host verbs
```
Expected: `omnigent` on PATH.

- [ ] **Step 2: Launch the local runner against the loopback server**

Use the exact command the web UI prints. Documented form:
```bash
omnigent run <agent.yaml> --server http://localhost:8000
```
(or `omnigent host http://localhost:8000` if `--help` shows a persistent host verb).
Expected: runner connects; UI shows this host online.

- [ ] **Step 3: Verify** — In the UI (after GitHub login), this host appears as an available runner/host.

---

### Task 8: Credentials sanity (subscription, not API)

**Files:** none.

- [ ] **Step 1:** `env | grep -E 'ANTHROPIC_API_KEY|OPENAI_API_KEY' || echo CLEAN` → Expected `CLEAN`.
- [ ] **Step 2:** `claude /status` → Expected subscription/plan (not API key).
- [ ] **Step 3:** confirm `codex` is logged in to the ChatGPT plan, then exit.

---

### Task 9: End-to-end smoke test (definition of done)

**Files:** produces `/tmp/hello.txt` on this host as proof.

- [ ] **Step 1:** From your PHONE browser, open `https://$DOMAIN` → Expected: forced GitHub login.
- [ ] **Step 2:** Try a NON-allowlisted GitHub account → Expected: access denied by oauth2-proxy.
- [ ] **Step 3:** Log in with the allowlisted account → Expected: UI loads; this host selectable.
- [ ] **Step 4:** Start a session on this host; instruct: "create the file /tmp/hello.txt containing the word omnigent".
- [ ] **Step 5:** On the host: `cat /tmp/hello.txt` → Expected `omnigent`; confirm the session used the subscription route.
- [ ] **Step 6:** Re-run Task 6 Step 4 from outside → Expected: only 443/80/22 reachable, 8000 closed.
- [ ] **Step 7:** Confirm `git status` shows no tracked secrets.

---

## Done means

All five spec success criteria pass: (1) public URL forces GitHub login + rejects non-allowlisted accounts; (2) UI loads, session starts on this host; (3) trivial task runs here via the subscription route; (4) only 443/80/22 reachable externally, 8000 closed; (5) no secrets in git.

**Deferred to sub-project B:** cross-vendor writer↔reviewer loop, Polly/bundle, Superpowers-in-subagent wiring, cost/ask-on-shell policies.
