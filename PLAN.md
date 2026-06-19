# Multi-agent coding system — implementation handoff

## Objective

Stand up a multi-agent coding system where Claude Code and Codex run together under one orchestrator, review each other's work **cross-vendor**, run on **subscription credentials** (not API billing), use the **Superpowers** methodology, and execute inside an **Ubuntu 24 devcontainer** — all driven through **Omnigent's** web UI.

> Omnigent is alpha (open-sourced June 2026). Treat the commands/schema below as current-as-of-handoff and verify against the repo while building.
> Refs: `github.com/omnigent-ai/omnigent`, `omnigent.ai` docs, `docs/AGENT_YAML_SPEC.md`, `deploy/README.md`, `docs/POLICIES.md`, `SECURITY.md`.

## Requirements

- Writer runs on the ChatGPT plan (Codex), reviewer on the Claude plan (Claude Code) — via the official CLIs, not API keys.
- The model reviewing a diff is always a different vendor than the one that wrote it.
- Bounded review loop: writer → reviewer → revise → re-review, capped at ~3 rounds; the human merges.
- Superpowers active in each coding CLI.
- Execution happens inside an Ubuntu 24 devcontainer (host is the same Ubuntu 24 image).
- Orchestration/observation through the Omnigent web UI, reachable from other devices.

## Architecture (three planes)

1. **Clients** — browser / phone → web UI.
2. **Control plane** — the Omnigent server: web UI (`:6767`), session/transcript store (SQLite), auth, policies. Deployed via `docker compose up` on the Ubuntu host (or a VPS).
3. **Execution** — the devcontainer: runs the Omnigent **host daemon** (registered with `omnigent host <server-url>`), which opens an **outbound** tunnel to the server. Agents run as subprocesses here, against the project's real toolchain.

Key property: the **only inbound surface is the server**. Daemon→server and CLI→provider are both outbound, so the devcontainer needs no inbound port.

## The agent bundle (writer / reviewer loop)

A YAML bundle (`build-review/`) defines the orchestration. Condensed `config.yaml`:

```yaml
spec_version: 1
name: build-review-loop

executor:
  harness: claude-native          # orchestrator; writes no code

prompt: |
  You are the tech lead. You write no code yourself.
  1. Delegate the task to `writer` (its own git worktree).
  2. Send the returned diff to `reviewer`.
  3. Parse the reviewer's verdict block:
       APPROVED          -> stop, summarize the diff for the human to merge.
       CHANGES_REQUESTED -> send FINDINGS back to `writer`, then go to step 2.
  4. Cap at 3 review rounds; if still not APPROVED, stop and list open findings.
  Never merge. Never push. The human merges.

tools:
  writer:
    type: agent
    executor: { harness: codex-native }   # ChatGPT subscription
    instructions: AGENTS.md
    prompt: |
      Implement the task in your worktree. Follow the `implement` skill.
      Return ONLY the diff + a one-paragraph summary. Do not merge or push.

  reviewer:
    type: agent
    executor: { harness: claude-native }  # Claude subscription, different vendor
    instructions: AGENTS.md
    prompt: |
      Review a diff you did NOT write. Follow the `code-review` skill and end
      with its exact verdict block. Be adversarial: find problems.

policies:
  cap_calls:
    type: function
    handler: omnigent.policies.builtins.safety.max_tool_calls_per_session
    factory_params: { limit: 200 }
  # Add ask_on_os_tools + cost_budget here too (see Policies section).
```

Bundle contents:

- `skills/implement/SKILL.md` — writer's contract (tests first, smallest diff, self-review, return diff + summary). Defers the deep workflow to Superpowers when present.
- `skills/code-review/SKILL.md` — reviewer's checklist (correctness, tests, security, maintainability, scope) **plus a required structured verdict block** (`VERDICT: APPROVED` or `VERDICT: CHANGES_REQUESTED` + `FINDINGS:` lines). This verdict is the linchpin — the orchestrator branches on it deterministically.
- `AGENTS.md` — shared repo conventions: build/test commands, style rules, "nobody pushes or merges — the human does," and the definition of done. Reference it from `CLAUDE.md` via `@AGENTS.md` so both CLIs read it.

Loop control lives in the orchestrator prompt, not a skill. Standards live in the skills.

## Skills layering

- **Superpowers** installed per-CLI (Claude + Codex) → methodology: brainstorm → spec → plan → TDD → inline self-review. This is in-family by design (the writer self-reviews within its own vendor — cheap, catches obvious mistakes).
- **Bundle `skills/`** → the cross-vendor review contract + verdict format (the second pair of eyes the in-family pass structurally can't provide).
- The two layers stack: in-family self-review during the build, cross-vendor review after.

## Auth & credentials (subscription, all-Linux)

Host and container are both Ubuntu 24, so the credentials are files (no macOS Keychain problem): Claude → `~/.claude/.credentials.json` (mode 0600); Codex → `~/.codex/auth.json`.

Pick one:

- **A) Bind-mount** `~/.claude` and `~/.codex` from the host into the container, **read-write**, with a **matching UID**.
- **B) One-time in-container login** (`omnigent setup` / CLI login), credentials persisted on a **named volume** so they survive rebuilds. Preferred if you also run the CLIs on the host.

Either way:

- **Unset** `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` in the container — if set, they override the subscription and silently bill API usage.
- Don't run the host CLI and the container CLI against the **same** credential file at the same time — OAuth refresh rotation can log one of them out. Prefer container-owned credentials (option B).
- Optionally mount `~/.agents/skills` to share skills into the container.
- Once the in-container CLIs are logged in, Omnigent picks them up automatically (it keys off "a claude/codex CLI you're logged into").

## Devcontainer bootstrap (self-registration on launch)

Omnigent does **not** launch a local devcontainer itself (its only auto-provisioning is managed Modal/Daytona sandboxes). Make *launching the container* perform the configuration by putting the bootstrap in the container lifecycle — `devcontainer.json` `postCreateCommand`, a compose `entrypoint`, or baked into the image:

```bash
curl -fsSL https://raw.githubusercontent.com/omnigent-ai/omnigent/main/scripts/install_oss.sh | sh
omnigent login https://your-server   # token, run once; reused after
omnigent host  https://your-server   # registers THIS container as a host
# plus: ensure claude + codex are logged in (mounted creds or one-time login)
```

Result: bringing up the container registers a configured, logged-in host with the server. The `<url>` passed here is always the **server** URL; the container becomes a host because the command runs inside it.

## Policies (governance backstop)

Enable as defense-in-depth regardless of network posture:

- `ask_on_os_tools` — approve before shell / file writes (scope to orchestrator/writer as needed).
- `cost_budget` — hard spend cap + soft warning threshold.
- `max_tool_calls_per_session` — bound runaway loops.
- A contextual policy: require approval to `git push` after a fresh package install.

Policies stack server-wide / per-agent / per-session, stricter session rules checked first.

## Security posture

- The server can drive code execution with your subscription tokens, so it is the asset to protect.
- **Prefer not exposing it on a raw public IP.** Use Tailscale / WireGuard / VPN, or an identity-aware proxy.
- If it must be public: terminate TLS (Render/Fly/Railway certs, or Caddy/nginx); keep `OMNIGENT_AUTH_ENABLED=1` (default in the Docker deploy); use OIDC with a domain allowlist or the proxy `header` auth mode rather than the shared admin password; treat session **Share** links as sensitive (a co-driver's messages execute on your machine).
- Read `SECURITY.md` before exposing anything.

## Constraints / non-goals

- **Don't double-sandbox.** The devcontainer is the sandbox — register it as a local host; do **not** use Modal/Daytona per-session (those lack your toolchain). If you later want auto-provisioned fresh environments, build a custom image and point a Daytona/Modal sandbox host at it — cloud, optional, later.
- **Don't give agents the Docker socket / DinD** to manage containers — it's effectively root on the host and defeats the isolation. Keep container lifecycle in your tooling; agents work inside the container. (Only exception: a deliberate, scoped Docker grant if the project's own tests need it.)
- **ToS:** subscription auth is for your individual use. If the server is shared and teammates drive agents through your seat, switch those sessions to API keys.

## Suggested build order

1. **Server up** — `docker compose up` on the Ubuntu host; confirm auth is on; set admin login; add OIDC/VPN if it needs to be remote.
2. **Devcontainer image** — Ubuntu 24 + toolchain + Node 22 + `tmux` + `omnigent`; add the bootstrap hook for login + host registration.
3. **Credentials** — mount or in-container login for Claude + Codex; verify `/status` shows the subscription route, not an API key.
4. **Single-agent smoke test** — `omnigent host`, then New session in the web UI, pick the container, run a trivial task; confirm it executes in-container.
5. **Superpowers** — install in both CLIs; confirm a skill triggers inside a sub-agent.
6. **Agent bundle** — add `config.yaml` + `skills/` + `AGENTS.md`; run the writer→reviewer loop on a throwaway repo; confirm the verdict block parses and the loop bounds at 3 rounds.
7. **Policies** — enable ask-on-shell + cost cap; confirm the gates fire.
8. **Harden** — TLS / auth / allowlist; write the runbook.

## Open questions to resolve during implementation

- Do Superpowers **and** the bundle `skills/` actually trigger inside an Omnigent-driven, sandboxed sub-agent runtime? (Smoke-test before trusting the loop.)
- Does `omnigent host` register a container as cleanly as a laptop? (Should — outbound tunnel — but verify.)
- Confirm the current Omnigent agent YAML schema and harness names (`claude-native` / `codex-native`) against `docs/AGENT_YAML_SPEC.md`.
- Pin an Omnigent version; expect breaking changes during alpha.