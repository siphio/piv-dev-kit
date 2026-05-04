---
description: "Scenario-based agent validation that tests all user flows, capabilities, and error paths"
argument-hint: [plan-file-path] [--full]
---

# Validate Implementation: Scenario-Based Agent Validation

**This is Stage 5 of 5** in the PIV flow: PRD → Research → Plan → Build → **Validate**. This command runs in its own fresh context window — no implementation bias carried over from `/execute`. After this stage produces a clean report, the user runs `/commit` to ship.

## Overview

**Goes beyond code health checks.** Tests every user flow, tool capability, decision tree, and error recovery path defined in the PRD. Validates that the agent behaves correctly across all scenarios, not just that the code compiles.

**Three validation sources:**
1. **Plan file** → VALIDATION COMMANDS section (code-level checks)
2. **PRD Section 4.3** → Scenario definitions (agent behavior checks)
3. **Technology profiles** → Validation hooks from `.agents/reference/` (integration checks)

**Philosophy**: Static analysis tells you code *looks* right. Functional testing proves it *runs*. Scenario validation proves the agent *behaves correctly*.

## Arguments

- `$ARGUMENTS`: Plan file path (optional, defaults to most recent in `.agents/plans/`)
- `--full`: Run all levels including full pipeline end-to-end (slower, may cost API credits)

### What Runs in Each Mode

| Phase | What It Does | Default | --full |
|-------|-------------|---------|--------|
| Phase 1 | Static analysis (code review) | ✅ ALWAYS | ✅ ALWAYS |
| Phase 2 | Level 1 syntax + Level 2 unit tests | ✅ ALWAYS | ✅ ALWAYS |
| Phase 3 | **LIVE integration tests (Tiers 1-3) + scenario validation** | ✅ ALWAYS | ✅ ALWAYS |
| Phase 4 | Full end-to-end pipeline run | ❌ SKIPPED | ✅ RUNS |

**CRITICAL: Phase 3 ALWAYS runs live tests.** This means:
- **Tier 1** (auto-live health checks) — runs automatically, no approval needed
- **Tier 2** (auto-live with test data) — runs automatically, no approval needed
- **Tier 3** (approval-required) — presents each test to user for approval before executing
- **Tier 4** (mock-only) — runs automatically with fixtures

The `--full` flag ONLY adds Phase 4 (full pipeline). It does NOT control whether live tests run — **live tests (Tiers 1-3) are the default behavior, not an opt-in.**

## Reasoning Approach

**CoT Style:** Per-subtask (one per validation level/scenario category)

For each validation level:
1. Load the relevant source (plan commands, PRD scenarios, technology profiles)
2. Determine what to test and expected outcomes
3. Execute tests and capture results
4. Compare actual vs expected outcomes
5. Classify: PASS / FAIL / PARTIAL / SKIPPED

For scenario validation specifically:
1. Map scenario Given/When/Then to executable steps
2. Determine integration tier (live vs fixture vs mock)
3. Execute and verify each assertion
4. Document deviations with specific details

## Hook Toggle

Check CLAUDE.md for `## PIV Configuration` → `hooks_enabled` setting.
If arguments contain `--with-hooks`, enable hooks. If `--no-hooks`, disable.
Strip `--with-hooks`, `--no-hooks`, and `--full` from arguments before using remaining text as plan path.

## Validation Tooling

Pick the most effective tool per check. Don't default to one.

| Task | Preferred tool | Why |
|------|---------------|-----|
| API endpoint probing (Tier 1-2) | `curl` + `jq` (or language equivalent) | Direct, scriptable, captures actual responses |
| Comparing live response to captured fixture | `diff` / structural compare in script | Deterministic drift detection |
| Driving web UI scenarios (when relevant) | `claude-in-chrome` | Real browser, console + network capture, screenshots as evidence |
| Synthetic edge case generation | Reasoning over PRD §4.2/§4.4 + profile §3 schemas | 1M context holds all branches simultaneously; identifies uncovered edges |
| Reading source for failure attribution | `Read` + `Grep` | Trace from observed failure back to code path |
| Checking prior validation state | `Read` on most recent `.agents/validation/*.md` | Differential diff to flag regressions |

**1M context unlock:** load all PRD scenarios + all profiles + all decision trees + all prior run results into one context. Cross-reference instead of summarize-and-discard. This is what enables consistency checks and synthetic generation that targets actual gaps.

## Provenance-Aware Validation

Profiles tag claims with `[source-verified]`, `[doc-only]`, or `[community]` (set by `/research-stack`). Treat them differently:

| Tag | Validation behavior |
|-----|-------------------|
| `[source-verified]` | Trust as ground truth. Probe-confirm only if drift detection runs. |
| `[doc-only]` | **Probe live to confirm reality.** Doc-only claims are the most likely source of validation surprises. |
| `[community]` | Use as targeted edge-case input. If a community gotcha says "API returns 200 with empty body on duplicate POST," generate a synthetic test that triggers exactly that. |

When a `[doc-only]` claim is contradicted by a live probe, surface it in the report as a profile-drift finding so `/research-stack --fresh` can correct it.

---

## Architecture

```
Phase 0: Context Loading
    │
    ├── Read plan file → Extract VALIDATION COMMANDS
    ├── Read PRD → Extract Section 4.3 Scenario Definitions
    ├── Read PRD → Extract Section 4.4 Error Recovery Patterns
    ├── Read technology profiles → Extract Validation Hooks (Section 9)
    ├── Read provenance tags → Plan probe targets for [doc-only] claims
    └── Load most recent prior validation report (if any) → For differential diff

Phase 1: Static Analysis (Quick)
    │
    └── Brief code review for obvious issues

Phase 2: Code Validation
    │
    ├── Level 1: Syntax/lint/type checks
    └── Level 2: Unit + component tests

Phase 3: LIVE Integration + Scenario Validation ← THE KEY PHASE (ALWAYS RUNS)
    │
    ├── Step 1: Tier 1 — Auto-live health + fixture drift detection ← LIVE, NO APPROVAL
    ├── Step 2: Tier 2 — Auto-live with test data + cleanup ← LIVE, NO APPROVAL
    ├── Step 3: Tier 3 — Approval-required live tests ← LIVE, ASK USER FIRST
    ├── Step 4: Tier 4 — Mock-only tests with fixtures
    ├── Step 5: PRD scenario validation (Section 4.3) — UI scenarios via browser when relevant
    ├── Step 6: Decision tree verification (Section 4.2)
    ├── Step 7: Synthetic edge case generation (covers gaps from §4.2/§4.4)
    └── Step 8: Cross-scenario consistency check (uses 1M context)

Phase 4: Full Pipeline (--full only)
    │
    └── End-to-end agent run with real/mock inputs

Phase 5: Report
    │
    ├── Differential diff vs prior run (regressions flagged)
    ├── Coverage gap analysis (PRD vs synthetic vs untested)
    └── Failure attribution chains for all FAILs
```

---

## Phase 0: Context Loading

### Step 1: Locate Plan

```bash
# If $ARGUMENTS provided (excluding flags), use it
# Otherwise find most recent:
ls -t .agents/plans/*.md 2>/dev/null | head -1
```

### Step 2: Load Validation Sources

**From Plan File:**
- Read `## VALIDATION COMMANDS` section → Parse into Levels 1-5
- Read `## ACCEPTANCE CRITERIA` section → Extract criteria list

**From PRD (if exists):**
- Read `## 4. Agent Behavior Specification`
- Extract Section 4.3: Scenario Definitions (all scenarios with Given/When/Then/Error/Edge)
- Extract Section 4.2: Decision Trees (expected decision outcomes)
- Extract Section 4.4: Error Recovery Patterns (expected recovery behaviors)

**From Technology Profiles (if exist):**
- Check `.agents/reference/` for `*-profile.md` files
- From each relevant profile, read Section 9: Validation Hooks
- Read Section 6.5: Maintenance & Risk Signals (drives expected reliability)
- Extract provenance tags applied to claims — list all `[doc-only]` claims as live-probe targets
- Extract health checks, smoke tests, and mock strategies
- List captured Tier 1 fixtures from `.agents/fixtures/` for drift detection

### Step 3: Load Prior Validation Report (Differential Diff)

If a prior report exists at `.agents/validation/`, load the most recent one:

```bash
ls -t .agents/validation/*.md 2>/dev/null | head -1
```

Extract from the prior report:
- Pass/fail counts by category
- Specific scenarios that failed
- Failure attribution chains
- Captured fixtures referenced

Hold this in context for Phase 5's regression diff. **Don't fail-replicate**: a prior failure doesn't predetermine the current run — re-run the failed scenarios fresh and report the new outcome.

### Step 4: Build Validation Matrix

Combine all sources into a validation matrix:

```
## Validation Starting

**Feature**: [Name from plan]
**Plan**: [path]
**PRD Available**: [Yes/No]
**Technology Profiles**: [list or "none"]

### Validation Commands (from Plan)
- Level 1 (Syntax): [N] commands
- Level 2 (Components): [N] commands

### Scenario Validation (from PRD Section 4.3)
- Happy path scenarios: [N]
- Error recovery scenarios: [N]
- Edge case scenarios: [N]
- Integration failure scenarios: [N]

### Technology Validation (from Profiles)
- Health checks: [N]
- Smoke tests: [N]
- `[doc-only]` claims to probe: [N]
- Captured fixtures available for drift check: [N]

### Synthetic Coverage (planned)
- Decision branches in PRD §4.2: [N]
- Error recovery paths in PRD §4.4: [N]
- `[community]` gotchas to convert to test cases: [N]

### Differential Context
- Prior report: `[path]` ([date]) | none
- Prior failures to re-test: [N]

### Acceptance Criteria
- [ ] [Criterion 1]
- [ ] [Criterion 2]

---
Proceeding to validation...
```

---

## Phase 1: Static Analysis (Brief)

**Quick code review - max 5 minutes, max 10 tool calls**

Read up to 3 key files mentioned in the plan and check for:
- Missing error handling (especially for error recovery paths in PRD 4.4)
- Obvious bugs or unimplemented functions (TODO/FIXME)
- Decision tree logic that doesn't match PRD Section 4.2

Output brief findings before running tests.

---

## Phase 2: Code Validation

### Level 1: Syntax Validation

**Always run.** Execute each command from plan's Level 1:

```
Running: `[command]`
✅ PASS: `[command]` (exit code 0)
```
or
```
❌ FAIL: `[command]` (exit code 1)
Error: [error details]
```

**Stop validation if Level 1 fails** - no point testing broken code.

### Level 2: Component Validation

**Always run.** Execute each command from plan's Level 2.

**Handle interactive commands:** Mark as "VERIFY MANUALLY"
**Handle timeouts:** 60 second timeout, mark as ⚠️ TIMEOUT

---

## Phase 3: Live Integration Testing & Scenario Validation

> **⚠️ THIS PHASE ALWAYS RUNS — it is NOT gated behind `--full`.**
> **THE CORE PHASE - Tests real API integrations AND agent behavior against PRD scenarios.**
> Uses the four-tier testing classification from technology profiles (Section 9).
>
> **You MUST execute Tiers 1-2 automatically (live, no approval), present Tier 3 for user approval,**
> **and run Tier 4 with fixtures. Skipping live tests or treating everything as mock-only is a validation failure.**

### Step 1: Tier 1 — Auto-Live Health Checks + Fixture Drift Detection (No Approval) — MANDATORY

**DO NOT SKIP. Execute these LIVE against real services — not mocked.**
Execute ALL Tier 1 tests from every relevant technology profile automatically.
These are read-only, zero-cost operations that verify connectivity and auth.

**Two-part check:**
1. **Health probe** — confirm endpoint is reachable and auth works
2. **Fixture drift detection** — diff live response against captured fixture from `.agents/fixtures/{tech}-{endpoint}.json` (written by `/research-stack` Step 6)

```
### Tier 1: Live Integration Health Checks

[Technology Name]:
  Endpoint: GET /[endpoint]
  Running: [health check command from profile Section 9.1]
  Live response: [actual response summary]
  Captured fixture: .agents/fixtures/[tech]-[endpoint].json (recorded [date])
  Drift check:
    - Schema match: ✅ identical | ⚠️ added/removed fields | ❌ shape changed
    - Field types: ✅ match | ❌ [field] changed [old-type] → [new-type]
    - Required fields present: ✅ / ❌ missing [field]
  Status: ✅ HEALTHY | ⚠️ DRIFT DETECTED | ❌ UNREACHABLE | ⚠️ AUTH FAILED
  [if drift] → Recommend: /research-stack --only [tech] --fresh
```

**If Tier 1 fails for a technology:**
- Mark ALL scenarios depending on that technology as ⚠️ DEGRADED
- Continue with other technologies
- Attempt mock fallback for dependent scenarios if fixtures exist

**If drift detected:**
- Continue validation but flag profile as stale in the report
- Drift is a warning, not a failure — but it means downstream `[doc-only]` claims may be wrong

**If no captured fixture exists:**
- Capture one now from the live response and write to `.agents/fixtures/{tech}-{endpoint}.json`
- Mark as `[probe-captured-during-validation]` so future drift checks have a baseline

### Step 2: Tier 2 — Auto-Live with Test Data (No Approval) — MANDATORY

**DO NOT SKIP. Execute these LIVE with real test data — not mocked.**
Execute ALL Tier 2 tests automatically using pre-defined test data from profiles.
These have controlled side effects with automatic cleanup.

```
### Tier 2: Live Tests with Test Data

[Technology Name] - [Operation]:
  Endpoint: POST /[endpoint]
  Test data: [summary of test input - e.g., "PIV_TEST_ prefixed campaign"]
  Response: [actual response]
  Schema valid: ✅ / ❌

  Cleanup: [DELETE /endpoint/{id}]
  Cleanup result: ✅ CLEANED | ❌ CLEANUP FAILED (manual cleanup needed)
```

**Test data sourcing:**
- Read test configuration from profile Section 9.1 (Tier 2 table)
- Environment variables (PIV_TEST_EMAIL, etc.) must be set in .env
- If env vars missing: WARN and skip Tier 2 for that technology

**Cleanup is mandatory:** Always run cleanup procedures after Tier 2 tests, even if the test itself failed. Cleanup must be idempotent.

### Step 3: Tier 3 — Approval-Required Live Tests (Human in the Loop) — MANDATORY

**DO NOT SKIP. Present each test to the user for approval — do not silently mark as "Not applicable".**
For each Tier 3 endpoint in the technology profiles, present the user with an approval prompt BEFORE executing. **Never auto-execute Tier 3 tests.**

```
### Tier 3: Approval-Required Tests

🔔 [Technology Name] - [Operation]

  To validate: [which PRD scenario this tests]
  Action: [METHOD /endpoint] with [test input description]
  Cost: [estimated cost from profile]
  Effect: [what happens - credits consumed, record created, etc.]
  Cleanup: [auto / manual / none]
  Last recorded: [date of existing fixture, or "no fixture"]

  → User chose: APPROVE / USE FIXTURE / SKIP

  [If APPROVED]:
    Response: [actual response]
    Schema valid: ✅ / ❌
    Fixture saved: .agents/fixtures/[tech]-[endpoint].json
    ✅ PASS | ❌ FAIL

  [If FIXTURE]:
    Loaded: .agents/fixtures/[tech]-[endpoint].json (recorded [date])
    Agent processed fixture: [result]
    ✅ PASS (fixture) | ❌ FAIL (fixture)

  [If SKIPPED]:
    ⏭️ SKIPPED by user
```

**Approval interaction:**
- Present ALL Tier 3 tests at once so user can batch approve/skip
- For each, show: endpoint, cost, effect, and which scenario it validates
- Accept: approve all, skip all, or individual decisions
- Record user decisions in validation report

**Response recording:**
When user approves a live Tier 3 call:
1. Execute the call and capture full response
2. Save to `.agents/fixtures/{technology}-{endpoint-name}.json`
3. Include timestamp, request, and response
4. On future runs, offer recorded fixture as alternative to fresh call

### Step 4: Tier 4 — Mock-Only Tests (Automatic)

Load fixtures for Tier 4 endpoints and feed responses into agent logic.
Tests that the agent correctly processes responses, not that the API works.

```
### Tier 4: Mock-Based Validation

[Technology Name] - [Operation]:
  Fixture: .agents/fixtures/[tech]-[endpoint].json
  Agent behavior: [what agent did with the fixture data]
  Decision tree: [which PRD 4.2 decision was triggered]
  Expected outcome: [from PRD]
  Actual outcome: [what happened]
  ✅ PASS | ❌ FAIL
```

**If fixture doesn't exist:**
- WARN: "No fixture for [endpoint]. Create one by running Tier 3 approval test first, or manually add fixture."
- Mark as ⚠️ NO FIXTURE

### Step 5: PRD Scenario Validation

Now test full agent scenarios from PRD Section 4.3 using the integration results from Steps 1-4.

For each scenario, the integration tier determines how it's tested:

```
### Scenario: [Name] (PRD 4.3)

Given: [Initial state from PRD]
APIs involved: [Technology A (Tier 1), Technology B (Tier 3)]
Integration status: [All APIs healthy / degraded / mocked]

When: [Trigger action]
Execute: [Command to trigger the agent workflow]

Then: [Expected outcome from PRD]
Verify: [Check output, state changes, API calls made]

Result: ✅ PASS | ❌ FAIL | ⚠️ PARTIAL (some APIs mocked)
Details: [What happened, which tiers were live vs mocked]
```

**Scenario categories:**

**Happy paths** — Test with maximum live integration (Tiers 1-3 where approved):
- Agent receives real API responses and processes them correctly
- Verify full decision tree execution with real data shapes

**Error recovery** — Simulate errors using mocks even for live APIs:
- Override Tier 1/2 responses with error fixtures to test recovery
- Verify agent handles timeouts, rate limits, auth failures per PRD Section 4.4

**Edge cases** — Test with unusual inputs and boundary conditions:
- Feed edge case data to agent logic
- Verify graceful handling per PRD scenarios

**Browser-driven scenarios (when relevant)** — For agents that involve a web UI or trigger browser actions, drive the scenario through `claude-in-chrome` rather than just calling underlying functions:
- Detect: scenario mentions a UI surface (login, form submit, dashboard interaction, OAuth redirect, hosted-page completion)
- Drive each Given/When/Then step in a real browser
- Capture as evidence: screenshots at each Then step, console errors during the run, network requests fired
- Save evidence to `.agents/validation/evidence/{scenario-id}/` (screenshots, har files)
- Compare actual UI state to PRD's Then clause — fail if visible state diverges from expected

```
### Browser Scenario: [Name] (PRD 4.3)

Browser flow:
  Step 1 (Given): Navigate to [URL] — ✅ loaded | ❌ failed
  Step 2 (When): [click X / fill form / submit] — ✅ acted | ❌ no element found
  Step 3 (Then): Verify [expected UI state] — ✅ matches | ❌ diverged

Console errors during run: [N] — [list if any]
Network failures: [N] — [list 4xx/5xx]
Evidence: .agents/validation/evidence/[scenario-id]/

Result: ✅ PASS | ❌ FAIL | ⚠️ PARTIAL
```

For CLI/backend agents with no UI surface, skip browser path entirely — existing function-call validation applies.

### Step 6: Decision Tree Verification

For each decision tree in PRD Section 4.2, verify with real data where possible:

```
### Decision Tree: [Name] (PRD 4.2)

Data source: [Live Tier 1-3 response / Fixture / Mock]

| Condition | Expected Action | Actual Action | Data Source | Status |
|-----------|----------------|---------------|-------------|--------|
| [Condition A] | [Action A] | [What happened] | [Live/Fixture] | ✅/❌ |
| [Condition B] | [Action B] | [What happened] | [Live/Fixture] | ✅/❌ |
| [Failure] | [Recovery] | [What happened] | [Mock error] | ✅/❌ |
```

### Step 7: Synthetic Edge Case Generation

PRD scenarios cover the cases the author thought of. Synthetic generation covers the cases they didn't. Use the 1M context to hold all PRD scenarios + decision branches + error recovery paths + profile schemas + community gotchas simultaneously, identify uncovered edges, and generate test cases that exercise them.

**Generation sources (highest signal first):**

1. **Decision branches not exercised by §4.3 scenarios** — for each branch in PRD §4.2 not covered by Step 5, synthesize input that should trigger it
2. **Error recovery paths in PRD §4.4** — synthesize the error condition (mock 429, mock 500, mock timeout, malformed response) and verify the documented recovery actually happens
3. **`[community]` gotchas from profiles** — convert each into a directed test (e.g. "API returns 200 with empty body on duplicate POST" → submit a duplicate, assert agent handles empty body)
4. **Boundary values from profile §2 data models** — empty/null, max-length, unicode, special chars, negative numbers — for each field the agent processes
5. **Concurrency edges (only if relevant)** — race conditions, parallel calls — only for agents whose architecture handles concurrency

**Tier classification for synthetic calls — MUST respect:**

| Synthetic call type | Tier | Approval | Cleanup |
|---------------------|------|----------|---------|
| Read-only probe with synthetic params | Tier 1 | Auto-live | None |
| Mutating call with synthetic test data | Tier 2 | Auto-live | **Mandatory cleanup, idempotent** |
| Costly synthetic call (paid inference, etc.) | Tier 3 | **Present to user with "synthetic edge case" label** | Per profile |
| Irreversible synthetic call | **NEVER GENERATED** | N/A — use mock with crafted fixture | N/A |

Output:
```
### Synthetic Edge Cases

Coverage gaps identified: [N]
Test cases generated: [N]

[Gap 1: Decision branch §4.2.3 "rate limit on retry"]
  Generated: Mock 429 response with Retry-After: 60 header
  Tier: Mock (no live call)
  Expected agent action (per PRD): wait 60s then retry
  Actual agent action: [observed behavior]
  Result: ✅ PASS | ❌ FAIL — agent retried immediately

[Gap 2: Boundary "empty string in name field"]
  Generated: POST /[endpoint] with `{"name": ""}`
  Tier: Tier 2 (mutating, with cleanup)
  Live response: [actual]
  Cleanup: ✅ CLEANED
  Expected: 400 with validation error
  Actual: [observed]
  Result: ✅ PASS | ❌ FAIL

[Gap 3: Community gotcha "duplicate POST returns 200 empty"]
  Generated: Two identical POST /[endpoint] calls in sequence
  Tier: Tier 2 (mutating, with cleanup)
  Result: [observed agent handling]
```

**Auditability rule:** every synthetic call must log the gap it addresses (decision branch ID, error path, gotcha source). No untraceable synthetic calls — they pollute the report.

### Step 8: Cross-Scenario Consistency Check

With all PRD scenarios + all results + all profiles loaded in context, look for contradictions and behavioral conflicts. This catches a class of bug pure pass/fail can't see.

Check for:
- **Contradictory expectations** — e.g. scenario A expects retry-on-429, scenario B expects fail-fast on the same condition
- **Behavior divergence** — same input produces different agent decisions across scenarios without explaining why
- **Missing fallthroughs** — decision tree exits one branch but no scenario tests where it goes after
- **Implicit dependencies** — scenario X assumes state from scenario Y but they run independently

```
### Cross-Scenario Consistency

Findings: [N]

[Finding 1: Contradiction]
  Scenarios: SC-003, SC-007
  Conflict: Both trigger HTTP 429 from [tech]; SC-003 expects retry-after, SC-007 expects immediate fail
  Resolution needed: [PRD owner clarifies which is correct, or both branches need explicit conditions]

[Finding 2: Coverage gap with implicit assumption]
  Scenario SC-012 assumes [tech] response includes field X
  No scenario tests behavior when field X is absent
  Risk: agent crashes on real-world responses missing X
  Recommendation: add error recovery scenario or synthetic edge case
```

If no findings: report "No cross-scenario contradictions detected. [N] scenarios, [M] decision branches checked."

### Agent Teams Mode (When Available)

> Parallel validation across tiers and scenario categories.

```
Team Lead coordinates validation:
├── Teammate 1: Tier 1-2 integration tests (Steps 1-2, auto)
├── Teammate 2: Happy path scenarios (Step 5, after Tier 1-2 complete)
├── Teammate 3: Error recovery + edge cases (Step 5)
├── Lead: Handles Tier 3 approvals (Step 3 - requires user interaction)
└── Lead: Decision tree verification + report (Step 6)
```

Tier 3 stays with the Lead because it requires human interaction. All other tiers parallelize across teammates.

---

## Phase 4: Full Pipeline (--full flag only)

**Only run if `--full` flag provided.** This may:
- Make real API calls (cost money)
- Take several minutes
- Create actual output files

```
### Full Pipeline

Running: `[end-to-end command]`
⏳ Running... (this may take several minutes)
✅ PASS: Agent completed successfully

Verifying outputs...
Running: `[output verification command]`
✅ FOUND: [output file] ([size])
✅ VALID: [format/quality details]
```

### Output Verification

After running commands, verify expected outputs:
- Check output files exist and are valid
- For media files: verify format with ffprobe or similar
- For data files: verify schema and content
- For API responses: verify response structure

---

## Phase 5: Report

### Write Report File

Location: `.agents/validation/{feature-name}-{YYYY-MM-DD}.md`

```markdown
# Validation Report: [Feature Name]

**Date**: [YYYY-MM-DD]
**Mode**: Standard | Full
**Duration**: [X] minutes
**PRD Scenarios Tested**: [N] of [Total]

---

## Code Validation Results

### Level 1: Syntax
| Command | Status | Details |
|---------|--------|---------|
| `[command]` | ✅ PASS | No errors |

### Level 2: Components
| Command | Status | Details |
|---------|--------|---------|
| `[command]` | ✅ PASS | [N] tests |

---

## Scenario Validation Results

### Happy Paths
| Scenario (PRD Ref) | Status | Details |
|---------------------|--------|---------|
| [Name] (SC-001) | ✅ PASS | [Brief] |

### Error Recovery
| Scenario (PRD Ref) | Status | Details |
|---------------------|--------|---------|
| [Name] (SC-005) | ✅ PASS | [Brief] |

### Edge Cases
| Scenario (PRD Ref) | Status | Details |
|---------------------|--------|---------|
| [Name] (SC-010) | ✅ PASS | [Brief] |

### Decision Trees
| Decision (PRD 4.2) | Branches Tested | Pass | Fail |
|---------------------|-----------------|------|------|
| [Name] | [N] | [N] | [N] |

---

## Technology Integration (Four-Tier Results)

### Tier 1: Auto-Live Health Checks
| Technology | Endpoint | Status | Details |
|-----------|----------|--------|---------|
| [Name] | GET /[endpoint] | ✅ HEALTHY | [Response summary] |

### Tier 2: Auto-Live with Test Data
| Technology | Operation | Status | Cleanup | Details |
|-----------|-----------|--------|---------|---------|
| [Name] | POST /[endpoint] | ✅ PASS | ✅ CLEANED | [Brief] |

### Tier 3: Approval-Required
| Technology | Operation | User Decision | Status | Fixture Saved |
|-----------|-----------|---------------|--------|---------------|
| [Name] | POST /[endpoint] | APPROVED | ✅ PASS | `.agents/fixtures/[file]` |
| [Name] | POST /[endpoint] | SKIPPED | ⏭️ SKIP | N/A |

### Tier 4: Mock-Only
| Technology | Operation | Fixture Used | Agent Behavior | Status |
|-----------|-----------|-------------|----------------|--------|
| [Name] | POST /[endpoint] | `.agents/fixtures/[file]` | [Decision triggered] | ✅ PASS |

### Fixture Drift (Tier 1)
| Technology | Endpoint | Drift | Recommendation |
|-----------|----------|-------|----------------|
| [Name] | GET /[endpoint] | ✅ none | — |
| [Name] | GET /[endpoint] | ⚠️ field added | Run `/research-stack --only [tech] --fresh` |

### Profile Provenance (where doc-only claims were probed)
| Technology | Claim | Provenance | Live probe | Result |
|-----------|-------|-----------|-----------|--------|
| [Name] | Auth header format | `[doc-only]` | Probed | ✅ confirmed |
| [Name] | Returns 200 on success | `[doc-only]` | Probed | ❌ actually returns 201 — profile drift |

---

## Synthetic Edge Case Validation

| Gap source | Generated test | Tier | Result | Details |
|-----------|---------------|------|--------|---------|
| Decision branch §4.2.3 | Mock 429 retry | Mock | ✅ PASS | Agent retried after 60s |
| Boundary: empty name | POST with `{"name":""}` | Tier 2 | ❌ FAIL | Returned 200 instead of 400 |
| Community gotcha (SDK) | Duplicate POST | Tier 2 | ✅ PASS | Agent handled empty body |

**Coverage delta from synthetic generation:**
- Decision branches now exercised: [N]/[Total]
- Error recovery paths now exercised: [N]/[Total]
- Community gotchas now tested: [N]/[Total]

---

## Coverage Gap Analysis

> Tracks what was tested by humans (PRD scenarios), what was tested by synthetic generation, and what remains untested.

| Category | PRD-defined | Synthetic | Untested | Total |
|----------|-------------|-----------|----------|-------|
| Decision branches (§4.2) | [N] | [N] | [N] | [N] |
| Error recovery paths (§4.4) | [N] | [N] | [N] | [N] |
| Endpoint capabilities | [N] | [N] | [N] | [N] |

**Untested items requiring attention:**
- [Decision branch / scenario / endpoint] — Why untested: [reason — e.g. "irreversible action, no fixture available"]
- [...] — Action: [add scenario / capture fixture / accept as known limitation]

---

## Cross-Scenario Consistency

[N] findings:
- [Finding 1: contradiction or implicit dependency]
- [Finding 2: ...]

If none: "No contradictions detected across [N] scenarios and [M] decision branches."

---

## Failure Attribution

> For every FAIL above, the chain that produced it. Don't accept "test failed" as a final answer.

| Scenario/Test | Observed failure | Attribution chain |
|---------------|------------------|-------------------|
| SC-007 | Agent didn't retry on 429 | External: API returned 429 → Logic: agent's retry handler skipped Retry-After header → Code: `src/api/retry.ts:42` ignores header — ROOT CAUSE |
| Synthetic empty-name | API returned 200 instead of 400 | External: API doesn't validate → Profile drift: §3 claimed 400 but `[doc-only]` — Recommend: re-run `/research-stack --fresh`, fix expected behavior |

**Attribution categories:**
- `Auth` — credentials, OAuth, token expiry
- `Schema drift` — API response shape changed since profile was captured
- `Logic` — agent code path doesn't implement PRD behavior correctly
- `External` — service outage, rate limit, transient failure
- `Data` — invalid input, missing required field, malformed payload
- `Profile` — research-stack profile was wrong; doc-only claim contradicted by reality

---

## Differential vs Prior Run

> Comparison with most recent prior validation report.

**Prior report**: `[path]` ([date])
**Regressions** (passed before, fails now): [N]
- [Scenario X] — was PASS, now FAIL — [brief attribution]

**Improvements** (failed before, passes now): [N]
- [Scenario Y] — was FAIL, now PASS

**New tests** (not in prior run): [N]
**Removed tests** (in prior, not now): [N]

If no prior run: "First validation run — no diff available."

---

## Acceptance Criteria

- [x] [Criterion] - **VERIFIED** (Level/Scenario)
- [ ] [Criterion] - **MANUAL CHECK NEEDED**

---

## Summary

**Overall**: 🟢 READY | 🟡 ISSUES | 🔴 BROKEN

| Category | Pass | Fail | Skip |
|----------|------|------|------|
| Syntax | [N] | [N] | [N] |
| Components | [N] | [N] | [N] |
| Happy Paths | [N] | [N] | [N] |
| Error Recovery | [N] | [N] | [N] |
| Edge Cases | [N] | [N] | [N] |
| Browser scenarios (if any) | [N] | [N] | [N] |
| Decision Trees | [N] | [N] | [N] |
| Synthetic edge cases | [N] | [N] | [N] |
| Tier 1 (Auto-Live) | [N] | [N] | [N] |
| Tier 1 fixture drift | [N] none | [N] drifted | — |
| Tier 2 (Test Data) | [N] | [N] | [N] |
| Tier 3 (Approval) | [N] | [N] | [N] |
| Tier 4 (Mock) | [N] | [N] | [N] |
| Cross-scenario consistency | [N] clean | [N] findings | — |
| Regressions vs prior run | — | [N] | — |
| Pipeline (if --full) | [N] | [N] | [N] |

---

## Issues Found

[List any failures with details and suggested fixes]

## Next Steps

[Based on results - ready for /commit or needs fixes]
```

### Terminal Summary

```
## Validation Complete

**Report**: `.agents/validation/[file].md`

### Code Validation
| Level | Results |
|-------|---------|
| Syntax | ✅ [N]/[N] passed |
| Components | ✅ [N]/[N] passed |

### Scenario Validation
| Category | Results |
|----------|---------|
| Happy Paths | ✅ [N]/[N] |
| Error Recovery | ✅ [N]/[N] |
| Edge Cases | ✅ [N]/[N] |
| Decision Trees | ✅ [N]/[N] branches |

### Technology Integration (Four Tiers)
| Tier | Results |
|------|---------|
| Tier 1 (Auto-Live) | ✅ [N]/[N] healthy, ⚠️ [N] drift |
| Tier 2 (Test Data) | ✅ [N]/[N] passed, [N]/[N] cleaned |
| Tier 3 (Approval) | ✅ [N] approved, [N] skipped, [N] fixture |
| Tier 4 (Mock) | ✅ [N]/[N] passed |

### Synthetic Coverage
| Source | Tests | Pass | Fail |
|--------|-------|------|------|
| Decision branch gaps | [N] | [N] | [N] |
| Error recovery synth | [N] | [N] | [N] |
| Community gotchas | [N] | [N] | [N] |
| Boundary values | [N] | [N] | [N] |

### Cross-Scenario Consistency
[N] findings | None detected

### Differential vs Prior Run
[N] regressions | [N] improvements | First run

### Coverage
- PRD scenarios: [N]/[Total]
- Decision branches: [N]/[Total] (PRD + synthetic)
- Error recovery paths: [N]/[Total]
- Untested items: [N] — see report

### Acceptance Criteria
✅ [N]/[N] verified

### Next Steps
→ Ready for `/commit` | Fix [N] issues first | Re-run `/research-stack --fresh` for [N] drift
```

### Reasoning

Output 4-8 bullets summarizing validation:

```
### Reasoning
- Tested [N] code validation commands (Level 1-2)
- Validated [N] PRD scenarios ([N] happy, [N] error, [N] edge, [N] browser-driven)
- Verified [N] decision tree branches
- Technology integration: [N] Tier 1, [N] Tier 2, [N] Tier 3, [N] Tier 4
- Generated [N] synthetic edge cases — [N] uncovered branches now exercised
- Fixture drift: [N] APIs drifted from captured profile
- Cross-scenario findings: [N] (contradictions/gaps)
- Differential: [N] regressions vs prior run
- Key finding: [most important result]
```

### Reflection

Self-critique the validation (terminal only):
- Did we achieve full scenario coverage from PRD Section 4.3?
- Are any decision tree branches untested even after synthetic generation?
- Were `[doc-only]` claims probed live? Any profile drift surfaced?
- Did synthetic edge cases respect tier classification (no irreversible synth-calls, all Tier 2 cleaned up)?
- Were failure attributions traced to root cause, or did we stop at symptom?
- Were regressions vs prior run flagged honestly?
- Is the recommended next step accurate given results?

### PIV-Automator-Hooks (If Enabled)

If hooks are enabled, append to the validation report file:

```
## PIV-Automator-Hooks
validation_status: [pass|partial|fail]
scenarios_passed: [N]/[Total]
scenarios_failed: [N]
decision_branches_tested: [N]/[Total]
synthetic_tests_generated: [N]
synthetic_tests_passed: [N]
fixture_drift_count: [N]
profile_drift_flagged: [N]
cross_scenario_findings: [N]
regressions_vs_prior: [N]
coverage_pct: [N]
failure_categories: [comma-separated: e.g. edge-cases,rate-limits,profile-drift]
suggested_action: [commit|re-execute|fix-and-revalidate|refresh-research]
suggested_command: [commit|execute|validate-implementation|research-stack]
suggested_arg: "[appropriate argument]"
retry_remaining: [N]
requires_clear: [true|false]
confidence: [high|medium|low]
```

---

## Handling Failures

### If Level 1 fails:
Stop validation. Syntax errors must be fixed first.

### If Level 2 fails:
Continue to scenario validation — component failures don't always block scenario testing.

### If scenarios fail:
Document which scenarios failed and what the agent actually did vs. expected.
Provide specific guidance: "PRD says agent should retry 3 times, but agent only retries once."

### If technology integration fails:
Check if service is reachable. If not, attempt mock mode.
Document whether failure is agent code issue or external service issue.

---

## Handling Different Project Types

### Agent Projects (Claude SDK / Custom)
```bash
# Level 1: Syntax
pnpm exec tsc --noEmit
# Level 2: Components
pnpm test
# Level 3: Scenarios
pnpm exec tsx -e "import('./src/agent').then(a => a.handleScenario(...))"
# Level 4: Full pipeline
pnpm dev agent -u "..." -a "..."
```

### Python Agent Projects
```bash
# Level 1: Syntax
uv run ruff check .
# Level 2: Components
uv run pytest
# Level 3: Scenarios
uv run python -m agent.test_scenarios
# Level 4: Full pipeline
uv run python -m agent.main --input "..."
```

---

## Usage

```bash
# Standard: Syntax + Components + Scenarios (Levels 1-3)
/validate-implementation

# Standard with specific plan
/validate-implementation .agents/plans/phase-2.md

# Full: Includes end-to-end pipeline (Levels 1-4)
/validate-implementation --full

# Full with specific plan
/validate-implementation .agents/plans/phase-2.md --full
```

---

## Completion Criteria

- [ ] Plan read and validation commands extracted
- [ ] PRD scenarios extracted (if PRD exists)
- [ ] Technology profiles read with provenance tags (if they exist)
- [ ] Prior validation report loaded for differential diff (if exists)
- [ ] Level 1 commands executed (all must pass to continue)
- [ ] Level 2 commands executed (failures noted)
- [ ] Tier 1 auto-live health checks executed + fixture drift checked
- [ ] `[doc-only]` claims probed live; profile drift surfaced
- [ ] Tier 2 auto-live tests with test data executed + cleanup run
- [ ] Tier 3 approval-required tests presented to user (approved/skipped/fixture)
- [ ] Tier 4 mock-only tests executed with fixtures
- [ ] PRD scenario validation complete (happy paths, error recovery, edge cases, browser-driven if applicable)
- [ ] Decision tree verification complete
- [ ] Synthetic edge cases generated and executed (respecting tier classification)
- [ ] Cross-scenario consistency check complete
- [ ] Differential diff vs prior run computed (if prior exists)
- [ ] Failure attribution chains documented for every FAIL
- [ ] Coverage gap analysis section in report
- [ ] Level 4 pipeline tested (if --full flag)
- [ ] Report written with actual pass/fail results per tier
- [ ] Acceptance criteria verified against scenarios
- [ ] All PRD scenarios accounted for (tested or documented as untestable)
- [ ] All synthetic Tier 2 calls have completed cleanup
- [ ] Browser scenario evidence saved to `.agents/validation/evidence/` (if applicable)
