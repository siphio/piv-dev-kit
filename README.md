# PIV Dev Kit

**AI-Powered Agent Development Framework**

A systematic 5-stage flow for building AI agents with Claude Code. Designed for **Opus 4.7's 1M context window** — each stage gets a full window, no per-phase cycling, no carry-over noise.

---

## Philosophy

> "Most AI coding failures are context failures, not capability failures."

This framework solves the #1 problem with AI-assisted development: **context loss**. With Opus 4.7's 1M context, each of the 5 stages runs in its own fresh context window — the framework's job is to produce durable file artifacts at each stage so the next stage starts clean with full fidelity.

**Core Principles:**
- **One stage per context window** — PRD, Research, Plan, Build, Validate each get a fresh window
- **One comprehensive plan, no phases by default** — phases are opt-in only when work is genuinely incrementally shippable
- **File-based handoffs** — every stage produces durable artifacts (`PRD.md`, profiles, plan, code, validation report)
- **Plain English over code snippets** — documents are scannable by humans
- **Provenance throughout** — claims tagged `[source-verified]` / `[doc-only]` / `[community]` flow from research → plan → build → validate
- **Live + synthetic validation** — real API probes plus auto-generated edge cases for branches the PRD didn't think of
- **Human checkpoints** — between stages and at natural milestones inside `/execute`

---

## The 5-Stage Flow

```
Stage 1: PRD          /create-prd                    → PRD.md
            ↓ /clear  ↓
Stage 2: Research     /prime → /research-stack       → .agents/reference/*-profile.md
            ↓ /clear  ↓                              → .agents/fixtures/*.json
Stage 3: Plan         /prime → /plan-feature         → .agents/plans/plan.md
            ↓ /clear  ↓
Stage 4: Build        /prime → /execute              → source code
            ↓ /clear  ↓
Stage 5: Validate     /prime → /validate-impl        → .agents/validation/*.md
                                                     → /commit
```

| Stage | Purpose | Command | Output |
|-------|---------|---------|--------|
| **1. PRD** | Define agent-native requirements | `/create-prd` | `PRD.md` |
| **2. Research** | Deep-dive technology stack with source verification | `/research-stack` | Technology profiles + captured fixtures |
| **3. Plan** | One comprehensive plan covering full PRD | `/plan-feature` | `.agents/plans/plan.md` |
| **4. Build** | Implement plan with milestone checkpoints | `/execute` | Source code |
| **5. Validate** | Live + synthetic validation, fresh adversarial eye | `/validate-implementation` | `.agents/validation/*.md` |

**Why fresh context per stage:**
- PRD work is product thinking, research is technical, plan is architectural, build is implementation, validate is adversarial. Each benefits from a clean break.
- Validate especially benefits from zero implementation bias — fresh eye on the same artifacts catches blind spots `/execute` would miss.
- 1M per stage > 200k recycled across 4 cycles per phase. Less ritual, more depth.

---

## Quick Start

### 1. Install Commands

Copy the `.claude/commands/` folder to your project:

```bash
mkdir -p .claude/commands
cp -r /path/to/piv-dev-kit/.claude/commands/* .claude/commands/
```

### 2. Create Project Rules

```bash
/create_global_rules_prompt
```

Generates a `CLAUDE.md` file with tech stack docs, conventions, architecture patterns, AI assistant instructions, and PIV Configuration.

### 3. Run the 5 Stages

```bash
# Stage 1 — PRD (in current context)
/create-prd

# /clear, open new context
/prime              # Stage detection: ready for Research
/research-stack

# /clear, open new context
/prime              # Stage detection: ready for Plan
/plan-feature

# /clear, open new context
/prime              # Stage detection: ready for Build
/execute

# /clear, open new context
/prime              # Stage detection: ready for Validate
/validate-implementation

# Ship it
/commit
```

The user `/clear`s between stages. `/prime` detects the current stage from project artifacts and loads only the context that stage needs.

---

## Commands Reference

### Stage 1: PRD

#### `/create-prd`

Creates an agent-native Product Requirements Document.

```bash
/create-prd              # Creates PRD.md
/create-prd myproject    # Creates myproject.md
```

**Output:** 500-750 line PRD with Agent Identity, Technology Decisions (with rationale), Agent Behavior Specification (decision trees + 8-15 scenario definitions + error recovery patterns), user stories.

**Phases are opt-in.** By default the PRD describes the feature as one coherent system. Section 9 (Implementation Phases) only appears when work is genuinely incrementally shippable to users (e.g. "Phase 1 ships to prod, Phase 2 builds on it").

---

### Stage 2: Research

#### `/research-stack`

Deep technology research with source verification, maintenance health, and live fixture capture.

```bash
/research-stack                    # Research all technologies from PRD.md
/research-stack PRD.md             # Specify PRD file
/research-stack --only instantly   # Single technology
/research-stack --fresh            # Bypass research cache
```

**What it does (per technology, in parallel via Agent Teams):**

1. **Read official documentation** (WebFetch for static docs, browser as fallback for JS-heavy SPAs, `gh` CLI for GitHub)
2. **Verify against source** — `git clone --depth 1` the recommended SDK; read auth handlers, retry logic, error types. Surface undocumented behaviors as gotchas. Use OpenAPI specs as ground truth when published.
3. **Maintenance health pass** via `gh` — release cadence, 90-day issue trends, security advisories, license, breaking-change history → drives confidence score
4. **Alt-comparison** (only when PRD records a Decision in Section 3) — confirm or flag the choice with evidence
5. **Probe Tier 1 endpoints** with `curl` — capture real responses to `.agents/fixtures/{tech}-{endpoint}.json` for downstream drift detection
6. **Provenance tags** on every claim: `[source-verified]` / `[doc-only]` / `[community]`

**Output:** `.agents/reference/{technology}-profile.md` per technology (200-450 lines each), plus captured fixtures in `.agents/fixtures/`.

**Profile sections include:**
- Auth & setup, data models, key endpoints (with tier classification)
- SDK/MCP server availability
- **§6.5 Maintenance & Risk Signals** (release cadence, advisories, confidence)
- Integration gotchas, capability mapping
- §9 Live Integration Testing Specification (Tiers 1-4 with real fixtures)

**Caching:** `.agents/cache/research/{tech}-{date}.{md,json}` — 14-day reuse window unless `--fresh`.

**Run once.** Profiles persist across stages and are consumed by Plan, Build, and Validate.

---

### Stage 3: Plan

#### `/plan-feature`

ONE comprehensive plan covering the entire PRD. No per-phase invocation, no per-section calls.

```bash
/plan-feature              # Plan the full PRD
/plan-feature --reflect    # Extended reflection pass
```

**Process:**
1. **Scope Analysis** — Reviews full PRD, all profiles, captured fixtures. Outputs decision recommendations + proposed logical sections + proposed natural milestones to terminal.
2. **User Validation** — Discuss approach before generating plan.
3. **Plan Generation** — 1500-2500 line plan organized by **logical sections** (Data Models, External Integrations, Core Logic, Error Handling, Configuration, Tests) with **natural milestones** for `/execute` checkpoints.

**Output:** `.agents/plans/plan.md` with:
- Cross-cutting decisions and architecture
- Logical sections (not phases) with concrete file changes
- Natural Milestones (M1, M2, M3...) — observable, testable checkpoint moments for `/execute`
- Validation commands per milestone
- Provenance-aware: `[doc-only]` claims flagged for verification during build

---

### Stage 4: Build

#### `/execute`

Builds the full plan in one session, pausing at natural milestones for user review.

```bash
/execute                              # Default: .agents/plans/plan.md
/execute .agents/plans/plan.md        # Explicit plan
/execute --no-checkpoints             # Auto-build all milestones (small features only)
```

**Milestone checkpoint loop:**
```
🛑 Milestone M2: Auth wired up complete

  Sections built: External Integrations (auth path)
  Files modified: 8
  Validation: pnpm test src/auth
  Validation result: ✅ PASS

  Continue to M3? [y/n/details]
```

**Process:**
1. Reads full plan (sections + milestones + provenance) + all profiles + fixtures
2. For each milestone: implements its sections, runs its validation command, pauses for approval
3. **Agent Teams**: Independent tasks within a milestone parallelize across teammates
4. **Sequential mode**: One task at a time (fallback)
5. Tracks progress in `.agents/progress/`

If a milestone's validation fails, halts even if user said "continue all" — needs explicit direction.

---

### Stage 5: Validate

#### `/validate-implementation`

Scenario-based validation with live integration tests, synthetic edge case generation, fixture drift detection, and cross-scenario consistency checks.

```bash
/validate-implementation              # Levels 1-3 (always live)
/validate-implementation --full       # Levels 1-4 (adds end-to-end pipeline)
```

**Always-on capabilities:**

| Capability | What it does |
|-----------|-------------|
| **Tier 1 health + drift** | Re-probes endpoints, diffs against captured fixtures from research-stack — flags API drift |
| **Tier 2 with cleanup** | Live mutating tests with mandatory idempotent cleanup |
| **Tier 3 with approval** | Costly tests presented to user; recorded fixtures for replay |
| **Tier 4 mock-only** | Irreversible operations always use fixtures |
| **Provenance-aware** | `[doc-only]` claims probed live; `[community]` gotchas converted to directed tests |
| **Browser-driven scenarios** | When PRD scenarios involve a UI, drives via `claude-in-chrome` with screenshots + console + network capture |
| **Synthetic edge cases** | Generates inputs/calls for uncovered decision branches, error recovery paths, boundary values |
| **Cross-scenario consistency** | 1M context holds all scenarios + profiles + results — catches contradictions |
| **Failure attribution** | Every FAIL gets a chain: auth / schema / logic / external / data / profile-drift |
| **Differential vs prior run** | Loads previous validation report — flags regressions explicitly |
| **Coverage gap analysis** | Distinguishes human-defined PRD scenarios vs synthetic auto-probes vs untested |

**Output:** `.agents/validation/{feature}-{date}.md` with structured report.

#### `/commit`

Creates git commits following conventional commit format.

```bash
/commit
/commit "feat: agent reasoning loop"
```

---

## Provenance System

Every non-trivial claim in technology profiles is tagged. Tags flow through every downstream stage:

| Tag | Meaning | How each stage uses it |
|-----|---------|----------------------|
| `[source-verified]` | Confirmed by reading SDK/API source, OpenAPI spec, or live probe | Plan trusts; Build trusts; Validate trusts |
| `[doc-only]` | Stated in docs, not independently verified | Plan flags for verification; Build verifies via live probe; Validate probes live as default |
| `[community]` | From issues, Stack Overflow, blog posts | Used as targeted edge-case input by Validate's synthetic generation |

When a `[doc-only]` claim is contradicted by a live probe during Validate, the report flags it as profile drift — re-run `/research-stack --fresh` to correct.

---

## Why Opus 4.7 + 1M Context Changes the Game

The old PIV loop was designed for 200k context — it forced per-phase cycles to fit work in the window. With 1M:

| Before (200k) | Now (1M) |
|---------------|----------|
| N×4 cycles per N-phase PRD | 5 stages, period |
| Per-phase plans (500-750 lines each) | One comprehensive plan (1500-2500 lines) |
| Plans must be self-contained per phase | Plans cross-reference freely |
| Research separate per technology, summarized | Each parallel research subagent holds full docs + source + issues |
| Validation works on summaries | Validation holds all scenarios + profiles + run results simultaneously |
| Drift between phases as context rebuilds | Cross-cutting consistency from full-fidelity reads |

The framework also exploits Opus 4.7's better agentic capabilities: deeper source-code reasoning, multi-step browser automation, parallel subagent coordination. The 5-stage flow keeps each cognitive activity isolated so each stage gets full attention without carryover noise.

---

## Agent Teams Integration

The framework supports Claude Code Agent Teams for parallel execution:

```json
// settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

**Where Agent Teams helps:**

| Command | Without Teams | With Teams |
|---------|--------------|------------|
| `/research-stack` | Sequential per technology | Parallel — one researcher per technology |
| `/execute` | One task at a time | Independent tasks within a milestone run simultaneously |
| `/validate-implementation` | Sequential scenario testing | Parallel — one validator per scenario category |
| `/orchestrate-analysis` | Sequential agent phases | Parallel where dependencies allow |

Each teammate gets full context — with 1M, that's a lot of fidelity per worker.

---

## Chain-of-Thought & Structured Reasoning

Every PIV command uses structured reasoning. No configuration needed.

**Three layers:**

1. **Chain-of-Thought (CoT)** — internal step-by-step reasoning. Styles vary per command:

| Style | Commands | Description |
|-------|----------|-------------|
| Zero-shot | `/prime`, `/commit` | Simple step-by-step |
| Few-shot | `/create-prd`, `/create_global_rules_prompt` | Includes examples of good vs bad output |
| Tree-of-Thought | `/plan-feature`, `/orchestrate-analysis` | Explores 2-3 approaches, evaluates each |
| Per-subtask | `/execute`, `/research-stack`, `/validate-implementation` | Each task/technology/scenario gets own reasoning chain |

2. **Reasoning Summary** — 4-8 bullet summary visible in terminal:
```
### Reasoning
- Researched 4 technologies from PRD §3
- 87% claims [source-verified], 11% [doc-only], 2% [community]
- Key finding: Anthropic SDK v0.30 deprecates messages.create — use messages.stream
- Maintenance: 3/4 profiles high-confidence; ElevenLabs flagged (stale 8mo)
```

3. **Self-Reflection** — terminal-only self-critique catching gaps before human review.

---

## PIV-Automator-Hooks

Optional machine-readable metadata for future autonomous SDK orchestration.

```
## PIV-Automator-Hooks
validation_status: partial
fixture_drift_count: 1
profile_drift_flagged: 1
synthetic_tests_generated: 12
synthetic_tests_passed: 11
regressions_vs_prior: 0
suggested_action: refresh-research
suggested_command: research-stack
suggested_arg: "--only elevenlabs --fresh"
confidence: medium
```

5-15 lines, regex-parseable (`^([a-z_]+): (.+)$`). Disabled by default.

**Enable per-project** in CLAUDE.md:
```markdown
## PIV Configuration
- hooks_enabled: true
```

**Per-command override:** `--with-hooks` / `--no-hooks` flags.

---

## Project Structure

After running the 5 stages:

```
your-project/
├── .claude/
│   └── commands/                    # PIV commands
├── .agents/
│   ├── plans/
│   │   └── plan.md                  # Single comprehensive plan
│   ├── progress/
│   │   └── plan-progress.md         # Milestone progress tracking
│   ├── validation/
│   │   ├── feature-2026-05-04.md    # Validation reports
│   │   └── evidence/                # Browser scenario screenshots/HARs
│   ├── reference/
│   │   ├── instantly-api-profile.md
│   │   ├── x-api-profile.md
│   │   └── elevenlabs-profile.md    # Technology profiles
│   ├── fixtures/
│   │   ├── instantly-list.json      # Captured Tier 1 fixtures
│   │   └── x-tweet.json
│   └── cache/
│       └── research/                # 14-day research cache
├── CLAUDE.md
├── PRD.md
└── src/
```

---

## Best Practices

### PRD Writing

**Do:**
- Keep to 500-750 lines
- Make Agent Behavior Specification the most detailed section
- Include 8-15 scenario definitions with error paths
- Capture technology decisions with rationale from conversation
- Skip Section 9 (Phases) by default — only opt in for genuinely incrementally shippable work

**Don't:**
- Include code snippets (save for plans)
- Add phases out of habit — they create artificial seams
- Skip error recovery patterns
- Leave technology choices unexplained

### Technology Research

**Do:**
- Run `/research-stack` once after PRD
- Let it verify docs against source (the biggest accuracy win)
- Review provenance distribution — high `[source-verified]` ratio = high confidence
- Capture fixtures with real test credentials when available
- Run `--fresh` if a major version dropped or validate flagged drift

**Don't:**
- Skip research and guess at API patterns
- Run before the PRD exists
- Treat profiles as immutable — re-run on flagged drift

### Planning

**Do:**
- Plan the full PRD in one shot
- Organize by logical sections, not phases
- Identify 3-7 natural milestones — observable, testable, independently meaningful
- Bake validated decisions into NOTES section
- Reference provenance — flag `[doc-only]` claims for live verification during build

**Don't:**
- Re-invoke `/plan-feature` per phase (the old way)
- Plan without reading all profiles + fixtures
- Set milestones at arbitrary points — they should map to real testable units

### Validation

**Do:**
- Run `/validate-implementation` in a fresh context window — implementation bias is real
- Always run live tier tests (Tiers 1-3) — that's the default
- Review synthetic edge case results — they catch what PRD authors missed
- Investigate failure attribution chains, don't stop at "test failed"
- Compare against prior run — regressions matter

**Don't:**
- Skip live tier tests by treating everything as mock
- Ignore fixture drift warnings
- Move to ship with cross-scenario consistency findings unresolved
- Trust static analysis alone

---

## Command Cheat Sheet

| Command | Stage | Output |
|---------|-------|--------|
| `/prime` | Any | Stage detection + context summary |
| `/create-prd` | 1 | `PRD.md` |
| `/create_global_rules_prompt` | Setup | `CLAUDE.md` |
| `/research-stack` | 2 | `.agents/reference/*.md` + `.agents/fixtures/*.json` |
| `/research-stack --fresh` | 2 | Bypass cache, re-research |
| `/plan-feature` | 3 | `.agents/plans/plan.md` |
| `/plan-feature --reflect` | 3 | Plan with extended reflection pass |
| `/execute` | 4 | Implemented code, milestone checkpoints |
| `/execute --no-checkpoints` | 4 | Auto-build all milestones (small features) |
| `/validate-implementation` | 5 | `.agents/validation/*.md` |
| `/validate-implementation --full` | 5 | Adds end-to-end pipeline test |
| `/commit` | Ship | Git commit |

**Global flags:**

| Flag | Effect |
|------|--------|
| `--with-hooks` | Enable PIV-Automator-Hooks for this run |
| `--no-hooks` | Disable hooks for this run |

---

## Troubleshooting

### "Technology profiles not found"
```bash
ls .agents/reference/   # Check what exists
/research-stack         # Stage 2 — generate profiles
```

### "Fixture drift detected" during validation
The API changed since `/research-stack` ran. Update profiles:
```bash
/research-stack --only [tech] --fresh
```

### "Profile drift flagged" during validation
A `[doc-only]` claim was contradicted by a live probe. Re-run research with verification:
```bash
/research-stack --only [tech] --fresh
```

### Plan not found
```bash
ls .agents/plans/       # Check what exists
/plan-feature           # Stage 3 — plan the PRD
```

### Synthetic edge case failed
The plan didn't account for that branch. Two options:
1. Update the implementation to handle the case
2. Update PRD §4.2 (decision trees) or §4.4 (error recovery) to clarify intended behavior, then re-plan

### Context lost after /clear
```bash
/prime                  # Detects current stage, loads stage-appropriate context
```

### Agent Teams not working
- Verify `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings
- Check you're using Claude Code (not API directly)
- One team per session only
- Teammates can't spawn nested teams

---

## Contributing

This framework evolves based on real-world usage. Key files:
- `.claude/commands/*.md` - Command definitions
- `CLAUDE.md` - Framework development rules

When modifying commands:
1. Read the full command first
2. Preserve the 5-stage flow
3. Ensure Agent Teams compatibility (parallel + sequential paths)
4. Include Reasoning Approach, Hook Toggle, Reasoning/Reflection sections
5. Test the workflow end-to-end across all 5 stages
6. Keep cross-references consistent (PRD sections, profile structure, provenance tags)

---

## License

MIT - Use freely for your agent development projects.

---

## Summary

The PIV Dev Kit transforms AI agent development from chaotic to systematic, exploiting Opus 4.7's 1M context for cleaner, faster, deeper iteration:

1. **PRD** — agent-native requirements with behavior specs and scenarios
2. **Research** — source-verified technology profiles with maintenance signals and captured fixtures
3. **Plan** — one comprehensive plan covering the full PRD with natural milestones
4. **Build** — milestone-checkpointed implementation in one session
5. **Validate** — live + synthetic + browser-driven validation with fresh adversarial context

Provenance tags flow end-to-end. Drift detection keeps profiles honest. Synthetic generation catches what PRD authors missed. Cross-scenario consistency checks catch contradictions invisible in pure pass/fail.

Every artifact survives context resets. Every stage starts fresh.

**Start here:**
```bash
/create-prd
# /clear
/prime
/research-stack
# /clear
/prime
/plan-feature
# /clear
/prime
/execute
# /clear
/prime
/validate-implementation
/commit
```
