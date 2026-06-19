# review-loop — fork notes

`review-loop` is a thin fork of Omnigent's stock **`polly`** orchestrator,
customized for sub-project B (cross-vendor writer↔reviewer loop). Keeping the
delta small and recorded here lets us re-sync against upstream as Omnigent
(alpha) evolves.

- **Upstream source:** `~/omnigent/examples/polly`
- **Pinned ref:** `918c153` (same clone pinned by sub-project A — see
  `deploy/DISCOVERED.md`)
- **Design:** `docs/superpowers/specs/2026-06-19-omnigent-review-loop-design.md`

## Deltas from upstream `polly`

### Dropped `pi` (→ 2-worker, claude_code ↔ codex)
`pi` is not installed on this host and no gateway model is configured, so the
loop runs on two vendors only. Cross-vendor review still holds: each PR is
reviewed by the *other* vendor.
- Removed `agents/pi/` entirely.
- Removed `- pi` from `tools.agents` in `config.yaml`.
- Scrubbed `pi` dispatch mentions from `config.yaml`, `skills/cross-review`,
  `skills/investigate` ("THREE sub-agents" → "TWO"; roster preflight
  `command -v claude codex`; review/explore routing to "the other vendor").
  (Incidental, non-dispatch mentions — e.g. `pi` in the "bad titles" example
  list — are left as-is.)

### Gap #2 — hard 3-round review cap
Upstream loops "a few" times (soft). Added an explicit cap of **3 review rounds
per PR** in both:
- `config.yaml` (the cross-review bullet in the skills list), and
- `skills/cross-review/SKILL.md` (new step 7).
After round 3 without a clean PR, STOP and escalate to the human with the PR URL
+ remaining blocking findings.

### Gap #3 — `cost_budget` policy
Added `omnigent.policies.builtins.cost.cost_budget` to `guardrails.policies` in
`config.yaml` (`max_cost_usd: 10.0`, `ask_thresholds_usd: [3.0, 6.0]`). NOTE the
builtin's semantics: the hard cap is a **downgrade gate** (DENYs while on an
expensive model — fable/opus/gpt-5 — prompting `/model`), not a hard kill;
`spawn_bounds` + `blast_radius` remain the runaway backstop. Tune the dollar
values to taste.

### Identity / naming
- `name: polly` → `name: review-loop`; description rewritten.
- Self-references in the prompt updated ("You are review-loop"), and the
  skill-authoring path corrected from `examples/polly/skills/` →
  `agents/review-loop/skills/`.
- Sub-agent prompts: "dispatched by the polly orchestrator" → "review-loop".
- Internal working-file name `.polly/registry.json` is **left unchanged** (an
  orchestrator-created dotfile in the target repo; renaming adds risk for no
  functional gain).

## Added files (not in upstream)
- `AGENTS.md.template` + `CLAUDE.md.snippet` — gap #4 per-project contract,
  dropped into each target repo.
- `FORK.md` — this file.

## Re-syncing against upstream
1. `cd ~/omnigent && git fetch && git log --oneline 918c153..origin/main -- examples/polly`
2. Review upstream changes to `examples/polly`; re-apply the deltas above on top.
3. Bump the pinned ref here and in `deploy/DISCOVERED.md`.
