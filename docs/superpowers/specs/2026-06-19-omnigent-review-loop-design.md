# Design: Cross-vendor writer↔reviewer loop (sub-project B)

- **Date:** 2026-06-19
- **Status:** Approved — implementing on branch `feat/omnigent-review-loop`
- **Source of truth for goals:** [PLAN.md](../../../PLAN.md) ("The agent bundle", "Skills layering", "Policies")
- **Builds on:** sub-project A (control plane stood up + secured; merged as PR #1)

## Scope

Stand up the cross-vendor build→review loop on top of the control plane from sub-project A.
**Adopt** Omnigent's stock `polly` orchestrator (which already implements the loop), **validate**
it end-to-end on a throwaway repo, and **close the four gaps** between stock Polly and PLAN.md's
explicit guarantees. We are NOT rebuilding the loop from scratch.

## Key finding that reframed this sub-project

PLAN.md proposed authoring a custom `build-review/` bundle. Inspecting the pinned clone
(`~/omnigent/examples/polly`) showed the stock `polly` agent already provides the intended loop,
more maturely than the PLAN sketch:

| PLAN.md sub-project B wanted | Stock Polly today |
|---|---|
| Writer on Codex, reviewer on Claude, via CLIs not API | `codex` (codex-native) + `claude_code` (claude-native) sub-agents |
| Reviewer always a different vendor than writer | Hard rule in prompt + `cross-review` skill |
| Writer → review → revise loop; human merges | `cross-review` skill loop; implementer opens PR, "polly does NOT merge" |
| Policies (runaway/loop bounds) | `blast_radius`, `spawn_bounds` (5/turn), `headless_purpose_guard`, `ask_timeout` |
| Superpowers active in each coding CLI | Installed for both — `~/.claude/skills/superpowers` (v6.0.3) and Codex's plugin set |

## Decisions (approved)

| Decision | Choice | Rationale |
|---|---|---|
| Approach | **Adopt + validate + close gaps** | Polly already implements the loop; rebuilding duplicates it |
| Workers | **2: `claude_code` ↔ `codex`** | `pi` is uninstalled and no gateway is configured; cross-vendor still holds (each PR reviewed by the other vendor) |
| Customization vehicle | **Thin fork in `agents/review-loop/`** | Version-controlled in this repo (matches sub-project A); cap + budget baked into config, not per-session |
| Validation target | **Dedicated throwaway scratch repo** | Zero risk to real code; clean PR target; matches PLAN.md step 6 |
| Orchestrator brain | **claude-sdk (Opus), subscription** | Inherited from Polly; confirmed subscription in sub-project A |

## The four gaps to close

1. **Superpowers proven to fire inside an Omnigent sub-agent** — PLAN.md's #1 open risk ("the
   linchpin"). Installed ≠ firing. Closed by *empirical validation*, not config: the smoke test
   must show a worker actually following a Superpowers skill (e.g. TDD / writing-plans) inside its
   sub-agent run.
2. **Hard 3-round review cap** — stock Polly says "a few loops" (soft). Encode an explicit cap of
   3 review rounds in the orchestrator prompt + `cross-review` skill: after 3 rounds without an
   APPROVED state, stop and hand the human the open findings.
3. **`cost_budget` policy** — add `omnigent.policies.builtins.cost.cost_budget` to the
   orchestrator's `guardrails.policies`. **Semantics (from source):** the hard `max_cost_usd` is a
   *downgrade gate* (DENYs while the session is on an expensive model — `fable`/`opus`/`gpt-5` —
   prompting `/model` to a cheaper one), not a hard kill; soft `ask_thresholds_usd` checkpoints ASK
   for approval. Paired with the existing `spawn_bounds` (5 dispatches/turn) + `blast_radius` for
   runaway protection.
4. **`AGENTS.md` per-project contract** — build/test commands, "nobody merges but the human",
   definition of done; referenced from `CLAUDE.md` via `@AGENTS.md` so both CLIs read it. Lives as
   a template in the bundle; dropped into each target repo.

## Components (authored in `agents/review-loop/`)

A thin fork of `~/omnigent/examples/polly`, with these deltas (tracked against upstream in
`agents/review-loop/FORK.md`):

- **`config.yaml`** — orchestrator. Deltas: drop `pi` from `tools.agents` and scrub `pi` dispatch
  mentions; "THREE sub-agents" → "TWO"; roster preflight `command -v claude codex`; add the hard
  3-round cap to the cross-review guidance; add `cost_budget` to `guardrails.policies`.
- **`agents/claude_code/config.yaml`, `agents/codex/config.yaml`** — copied verbatim (already
  correct: own worktree, opens own PR, cross-vendor review, blast_radius).
- **`skills/cross-review/SKILL.md`** — edited: explicit 3-round cap at step 7; `pi` mentions
  removed.
- **`skills/investigate/SKILL.md`, `skills/fanout/SKILL.md`** — kept (compose with the loop);
  `pi` mentions removed from dispatch lists.
- **`AGENTS.md.template`, `CLAUDE.md.snippet`** — gap #4 templates for target repos.
- **`FORK.md`** — records the exact diff from upstream Polly + the pinned ref, for re-syncing.

## Validation (the smoke test)

On a dedicated throwaway scratch repo (created outside this repo, e.g. `/tmp/review-loop-smoke`):

1. `git init`; add one trivial failing test (e.g. `add(a, b)` returning the wrong value) + the
   `AGENTS.md` template + `CLAUDE.md` include. Make it a GitHub repo so `gh pr create` works.
2. From the web UI, start a `review-loop` session: "make the failing test pass."
3. Observe and confirm:
   - **Linchpin (gap #1):** a worker visibly follows a Superpowers skill inside its sub-agent run.
   - Implementer (one vendor) makes the change in its own worktree, drives tests green, opens a PR.
   - Cross-review runs on the **opposite** vendor; blocking issues loop back to the same implementer.
   - The loop **bounds at 3 rounds** (gap #2) if it can't converge.
   - The orchestrator never merges; the PR is left for the human.
4. Throw the scratch repo away.

## Error handling / known risks

- **Custom-bundle discovery:** verify how the runner/server surfaces a forked bundle in the web UI
  agent picker (built-ins appeared automatically in A). Resolve during the plan; fall back to
  `omnigent run agents/review-loop/config.yaml` if needed.
- **Superpowers may not fire in a sub-agent** (the linchpin): if validation shows it doesn't,
  that's a finding to fix (skill path / env in the worker) before trusting the loop — do not paper
  over it.
- **`pi` scrub regressions:** an incompletely scrubbed `pi` mention could make the brain dispatch a
  non-existent worker. Mitigated by removing the `pi` agent dir + `tools.agents` entry so a stray
  dispatch fails loud rather than silently.
- **cost_budget on headless workers:** the downgrade-gate semantics assume an interactive `/model`
  switch; a headless worker can't. The budget is per-session and routed to the root conversation,
  so the orchestrator surfaces it; document this so a budget hit is understood, not mysterious.

## Success criteria

1. A `review-loop` session runs on the scratch repo and produces a mergeable PR via the
   **subscription** route.
2. The diff is reviewed by the **opposite vendor**; blocking issues loop back; the loop bounds at
   **3 rounds**.
3. A Superpowers skill is observed **firing inside a sub-agent** (gap #1 closed empirically).
4. `cost_budget` is active on the orchestrator (gap #3); `AGENTS.md` contract is read by the
   workers (gap #4).
5. The orchestrator never merges; no secrets in git; the fork's delta from upstream is recorded.

## Out of scope

- Devcontainer / Modal / Daytona sandboxes (PLAN.md non-goal: "don't double-sandbox").
- Multi-user / team seats (subscription ToS is individual use).
- Re-running on real projects — that comes after validation passes.
