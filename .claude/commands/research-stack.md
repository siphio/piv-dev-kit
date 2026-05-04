---
description: Deep-dive technology research producing structured profiles for all platforms in the PRD
argument-hint: [prd-file-path]
---

# Research Stack: Deep Technology Profiling

## Overview

Perform comprehensive research on every external technology, API, and platform referenced in the PRD. Produces structured technology profiles in `.agents/reference/` that are consumed by `/plan-feature`, `/execute`, and `/validate-implementation`.

**Run ONCE after PRD creation. Rerun only if new technologies are added.**

## When to Run

```
/create-prd → /research-stack → /plan-feature → /execute → /validate-implementation
                  ↑ Stage 2 — runs once after PRD; outputs profiles consumed by all later stages
```

## Input

Read the PRD from: `$ARGUMENTS` (default: `PRD.md`)

Extract technologies from:
- **Section 3: Technology Decisions** - Primary source of what to research
- **Section 7: Technology Stack** - External services table
- **Section 4: Agent Behavior Specification** - Tools and their platforms

## Output

For each technology, write a structured profile to:
`.agents/reference/{technology-name}-profile.md`

Examples: `instantly-api-profile.md`, `x-api-profile.md`, `elevenlabs-profile.md`

## Agent Teams Mode

> **When Agent Teams is available**, this command naturally parallelizes. Each technology gets its own research teammate working simultaneously.

**With Agent Teams:**
```
Team Lead reads PRD → identifies N technologies → spawns N research teammates
   ├── Teammate 1: Research Technology A → writes profile A
   ├── Teammate 2: Research Technology B → writes profile B
   └── Teammate 3: Research Technology C → writes profile C
All teammates work simultaneously, each producing their profile.
```

**Without Agent Teams:**
Research technologies sequentially, producing each profile before moving to the next.

## Reasoning Approach

**CoT Style:** Per-subtask (one per technology)

For each technology being researched:
1. Extract PRD context — why chosen, what agent needs, relevant scenarios
2. Read official documentation — auth, endpoints, rate limits, SDKs
3. **Verify against source** — read actual SDK/library source code, not just docs about it
4. **Maintenance health pass** — releases, issue trends, security advisories via `gh`
5. **Alt-comparison** (only if PRD records a Decision in Section 3) — confirm or flag
6. **Probe Tier 1 endpoints** — capture real responses as fixtures
7. Research community knowledge — gotchas, workarounds, real-world patterns
8. Structure findings into profile format with provenance tags
9. Classify all endpoints into testing tiers (1-4)

After completing each profile, reflect:
- Is the profile accurate and complete for this agent's needs?
- Are claims source-verified where possible, or only doc-derived?
- Are there missing gotchas or undocumented limitations?
- Does the testing tier classification cover all endpoints?
- Are maintenance signals current and the confidence score honest?

## Hook Toggle

Check CLAUDE.md for `## PIV Configuration` → `hooks_enabled` setting.
If arguments contain `--with-hooks`, enable hooks. If `--no-hooks`, disable.
Strip flags from arguments. Check for `--only [tech]` to research single technology.
Check for `--fresh` to bypass research cache (see Caching section).

## Tool Selection

Pick the most effective tool per research task. Don't default to one. Opus 4.7's 1M window means each parallel subagent can hold full docs + source + issue threads simultaneously — exploit that depth, but pick the cheapest tool that gets ground truth.

| Task | Preferred tool | Why |
|------|---------------|-----|
| Static doc pages, simple HTML | `WebFetch` | Cheapest; one round-trip |
| GitHub repo metadata, issues, releases, advisories | `gh` CLI (`gh release list`, `gh issue list`, `gh api`) | Structured output, no rate-limit dance |
| Reading source code | `git clone --depth 1` to a temp dir + `Read` | Full-fidelity, queryable, fits in context |
| Probing live API endpoints (Tier 1 only) | `curl` + `jq` (or language equivalent) | Confirms response shape, captures real fixtures |
| Package metadata / version history | `npm view`, `pip index`, `cargo search`, etc. | Authoritative, scriptable |
| OpenAPI/Swagger specs, JSON schemas | `WebFetch` or `curl` | Ground truth for §2/§3 — prefer over prose docs when published |
| JS-rendered SPAs, login-walled docs, dynamic dashboards | `claude-in-chrome` | Fallback only — when WebFetch returns empty/broken content |
| Cookbook / examples repos (if they exist) | `git clone --depth 1` + `Read` | Real working code beats doc snippets |

**Escalation rule**: start with the cheapest tool. Escalate only when output is broken, empty, or insufficient. Don't reach for the browser when WebFetch worked.

---

## Research Process (Per Technology)

### Step 1: Read PRD Context

For this technology, extract from the PRD:
- **Why chosen**: The rationale from Technology Decisions (Section 3)
- **What the agent needs**: Specific capabilities required
- **Integration approach**: REST/SDK/MCP
- **Known constraints**: Rate limits, auth, pricing mentioned
- **Relevant scenarios**: Which scenarios from Section 4.3 involve this technology
- **Decision trees**: Which decisions from Section 4.2 depend on this technology

This context focuses the research. You're not documenting the entire API — only what this agent needs.

### Step 2: Read Official Documentation

Use the Tool Selection table above. Default path: `WebFetch` for static docs.

1. **Official API documentation**
   - Authentication guides (API keys, OAuth flows, token refresh)
   - Endpoint reference for capabilities the agent needs
   - Rate limit documentation
   - Error code reference
   - SDK/client library documentation

2. **Getting started / quickstart guides**
   - Setup requirements
   - First API call examples
   - Common patterns

3. **Changelog / what's new**
   - Recent breaking changes
   - Deprecated endpoints
   - New features relevant to agent's use case

4. **OpenAPI / Swagger spec** (if published)
   - Fetch and use as ground truth for §2 (data models) and §3 (endpoints)
   - Prefer spec over prose docs when they disagree — flag the disagreement as a gotcha

### Step 3: Verify Against Source

Don't stop at docs. Read the actual code that implements or wraps this API. Docs lie or lag; source is ground truth.

1. **Recommended SDK source** (highest yield)
   - `git clone --depth 1` the recommended SDK repo to a temp dir
   - Read its auth handler, request builder, retry logic, error types, and pagination helpers
   - Look for: undocumented default headers, retry-after handling, hidden timeouts, edge-case behaviors not in docs
   - These become §7 Integration Gotchas with `[source-verified]` provenance

2. **API server source** (if open source)
   - Read error response paths, validation logic, rate-limit enforcement
   - Confirms §5 Error Handling assumptions

3. **Cookbook / examples repo** (if it exists, e.g. `*-cookbook`, `*-samples`)
   - Real working code beats doc snippets
   - Surface idiomatic patterns into §6 SDK / §3 Endpoints

**Rule:** any claim about behavior should be traceable to source, OpenAPI spec, doc page, or community thread. Tag accordingly (see Provenance Tags).

### Step 4: Maintenance Health Pass

Use `gh` CLI to assess the technology's health. Drives the §6.5 Maintenance & Risk Signals section and the confidence score.

```
gh api /repos/{owner}/{repo}                       # stars, last update, archived flag
gh release list --repo {owner}/{repo} --limit 10   # release cadence
gh issue list --repo {owner}/{repo} --state open --limit 30 --json title,createdAt,labels
gh api /repos/{owner}/{repo}/security-advisories   # known advisories
```

Capture:
- **Last release date** and **cadence** (weekly / monthly / quarterly / stale > 6 months)
- **Open issue trends** — count over last 90 days, ratio of bug-labeled vs feature-labeled
- **Security advisories** — count, severity, latest patched version
- **License** — SPDX identifier; flag GPL / SSPL / non-OSI if relevant
- **Breaking-change signal** — major-version bumps in last 12 months

### Step 5: Alt-Comparison (Conditional)

**Run only if PRD Section 3 records this technology as a Decision** (not just a list of options). Otherwise skip — the PRD already chose for non-Decision techs.

For each Decision:
1. Identify 1–2 named alternatives the PRD considered (or that are obvious peers)
2. Use existing research to compare on the dimensions the agent actually depends on (e.g. rate limits if the agent is throughput-sensitive, latency if real-time, pricing if high-volume)
3. **Confirm** the PRD's choice with evidence, **or** flag it if an alternative is materially better — surface in terminal Final Output, not the profile

Don't expand scope. Two-paragraph evaluation, terminal-only output. The PRD owner decides whether to revise.

### Step 6: Probe Tier 1 Endpoints & Capture Fixtures

Before classifying tiers, probe the read-only endpoints the agent will use to verify response shapes against docs.

1. Use `curl` (or language equivalent) with the agent's test API key from `.env`
2. Hit each candidate Tier 1 endpoint
3. Save the actual response to `.agents/fixtures/{technology}-{endpoint-name}.json` using the fixture format from §9.1
4. Compare actual response shape against documented shape — flag disagreements as `[source-verified]` gotchas
5. If credentials aren't available, skip and mark Tier 1 fixtures as `[doc-only]`

This kills hallucination risk in §9 — the fixtures are real, not invented.

### Step 7: Research Community Knowledge

Fill remaining gaps from real-world experience:

1. **GitHub issues** — known problems, common workarounds (read with `gh issue list` + `gh issue view`)
2. **Stack Overflow / Developer forums** — undocumented behaviors, performance tips
3. **Blog posts / tutorials** — production patterns, gotchas not in official docs

### Step 8: Compile Technology Profile

Write the profile to `.agents/reference/{technology-name}-profile.md` using the format below. Apply provenance tags throughout.

---

## Provenance Tags

Every non-trivial claim in the profile carries a provenance tag so `/plan-feature` knows what's load-bearing:

| Tag | Meaning | When to use |
|-----|---------|-------------|
| `[source-verified]` | Confirmed by reading SDK/API source, OpenAPI spec, or live probe | Auth flows, error shapes, undocumented behaviors found in code |
| `[doc-only]` | Stated in official docs, not independently verified | Default when source isn't available |
| `[community]` | From issues, Stack Overflow, blog posts | Gotchas, real-world workarounds |

Apply tags inline at the end of bullets, table notes, or section headers. Example:
```
- Auth header: `Authorization: Bearer {key}` `[source-verified]`
- Rate limit: 100 req/min on free tier `[doc-only]`
- Returns 200 OK with empty body on duplicate POST `[community]`
```

A profile with mostly `[doc-only]` tags signals lower confidence; `[source-verified]` claims are the ones `/plan-feature` and `/execute` should treat as guaranteed.

## Technology Profile Format

Every profile MUST follow this exact structure for consistency. The `/plan-feature` and `/execute` commands expect this format.

```markdown
# Technology Profile: [Technology Name]

**Generated**: [Date]
**PRD Reference**: Section 3 - [Technology Name]
**Agent Use Case**: [One sentence from PRD describing what agent needs this for]

---

## 1. Authentication & Setup

**Auth Type**: [API Key / OAuth 2.0 / Bearer Token / etc.]
**Auth Location**: [Header / Query param / Body]

**Setup Steps:**
1. [Create account at URL]
2. [Generate API key at URL]
3. [Set environment variable: ENV_VAR_NAME]

**Auth Code Pattern:**
```[language]
[Minimal working auth example - actual code, not pseudocode]
```

**Environment Variables:**
| Variable | Purpose | Required |
|----------|---------|----------|
| [VAR_NAME] | [What it's for] | Yes/No |

---

## 2. Core Data Models

> Only models relevant to this agent's use case.

**[Model Name]:**
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| [field] | [type] | [what it is] | Yes/No |

[Repeat for each relevant model]

---

## 3. Key Endpoints

> Only endpoints the agent needs based on PRD capabilities.

### [Capability from PRD]: [Endpoint Name]

**Method**: [GET/POST/PUT/DELETE]
**URL**: `[full endpoint URL with path params]`

**Request:**
```json
{
  "field": "example_value"
}
```

**Response:**
```json
{
  "field": "example_value"
}
```

**Notes**: [Pagination, filtering, important behaviors]

[Repeat for each needed endpoint]

---

## 4. Rate Limits & Throttling

| Endpoint/Scope | Limit | Window | Retry Strategy |
|----------------|-------|--------|----------------|
| [Global] | [N] req | [per second/minute] | [Exponential backoff / wait] |
| [Specific endpoint] | [N] req | [per period] | [Strategy] |

**Recommended throttle implementation:**
- [Specific guidance for this agent's use pattern]
- [How to detect rate limiting - headers, status codes]
- [Backoff strategy that fits the agent's workflow]

---

## 5. Error Handling

| Status Code | Meaning | Agent Should |
|-------------|---------|-------------|
| [400] | [Bad request] | [Validate input and retry with corrections] |
| [401] | [Unauthorized] | [Refresh token / alert user] |
| [429] | [Rate limited] | [Back off per Section 4 strategy] |
| [500] | [Server error] | [Retry with exponential backoff, max 3 attempts] |

**Error Response Format:**
```json
{
  "error": "example_error_response"
}
```

---

## 6. SDK / Library Recommendation

**Recommended**: [Library name] v[version]
**Install**: `[install command]`
**Why**: [Brief justification - maintained, typed, covers needed endpoints]

**Alternative**: [If primary is unavailable]

**MCP server**: [Yes — `[server-name]` (official/community) | No]
- If yes: brief description and when it's preferable to direct SDK use
- If no: skip

**If no suitable SDK**: Use raw HTTP with [requests/fetch/axios] - patterns below:
```[language]
[Minimal HTTP client example]
```

---

## 6.5 Maintenance & Risk Signals

> Health signals from `gh` and registry queries. Drives confidence score in hooks block.

**Repository**: `[owner/repo]`
**License**: [SPDX identifier — flag GPL/SSPL/non-OSI if relevant]

**Release cadence:**
| Signal | Value |
|--------|-------|
| Last release | [date] |
| Last 90-day release count | [N] |
| Cadence classification | [weekly / monthly / quarterly / stale > 6mo] |
| Major version bumps in last 12mo | [N] — [list versions if breaking] |

**Issue health (last 90 days):**
| Signal | Value |
|--------|-------|
| Open issues total | [N] |
| Bug-labeled open | [N] |
| Critical/P0 open | [N — link issue numbers if any] |

**Security advisories:**
- [None] OR [N advisories — list with severity and patched version]

**Confidence**: [high / medium / low]
- **High** = active releases, low critical-bug count, source-verified profile
- **Medium** = some staleness or doc-only sections, no critical issues
- **Low** = stale > 6mo, archived, unpatched advisories, or major undocumented divergence from docs

**Risks for this agent:**
- [Bullet 1 — concrete risk derived from signals above, e.g. "v2 breaking change planned Q3 — pin v1.x"]
- [Bullet 2]

---

## 7. Integration Gotchas

> Practical issues discovered from community research that official docs don't cover.

1. **[Gotcha title]**: [Description and workaround]
2. **[Gotcha title]**: [Description and workaround]
3. **[Gotcha title]**: [Description and workaround]

---

## 8. PRD Capability Mapping

> Maps PRD requirements to specific implementation paths.

| PRD Capability (from Section 3) | Endpoints/Methods | Notes |
|--------------------------------|-------------------|-------|
| [Capability 1] | [POST /endpoint] | [Implementation notes] |
| [Capability 2] | [GET /endpoint + POST /other] | [Sequence/workflow notes] |

---

## 9. Live Integration Testing Specification

> **This section drives `/validate-implementation` Phase 3.** It classifies every endpoint the agent uses into testing tiers and provides the exact data, commands, and approval requirements for live API validation.

### 9.1 Testing Tier Classification

Classify EVERY endpoint from Section 3 into exactly one tier:

#### Tier 1: Auto-Live (No Approval Needed)

> Read-only, zero cost, zero side effects. Runs automatically every validation.

| Endpoint | Method | Purpose | Expected Response Shape | Failure Means | Captured Fixture |
|----------|--------|---------|------------------------|---------------|------------------|
| [GET /endpoint] | GET | [Verify auth, check account] | `{ "field": "type" }` | [Auth broken / service down] | `.agents/fixtures/{tech}-{endpoint}.json` |

**Captured Fixtures** (populated by Step 6 probes during research):
- Each Tier 1 endpoint should have a real captured response saved at `.agents/fixtures/{tech}-{endpoint-name}.json`
- Fixtures are recorded once at research time and reused across validation runs
- If credentials weren't available during research, mark as `[doc-only]` and `/validate-implementation` will probe on first run

**Health Check Command:**
```[language]
[Actual executable code that calls a Tier 1 endpoint and verifies response against captured fixture]
```

**Schema Validation:**
```[language]
[Code that parses response and checks expected fields exist with correct types]
[Compares against the captured fixture as ground truth]
```

#### Tier 2: Auto-Live with Test Data (No Approval Needed)

> Controlled side effects with pre-defined test data. Includes automatic cleanup.

| Endpoint | Action | Test Data | Cleanup Action | Why Safe |
|----------|--------|-----------|----------------|----------|
| [POST /endpoint] | [Create test record] | [Test input with PIV_TEST_ prefix] | [DELETE /endpoint/{id}] | [Isolated, auto-deleted] |
| [POST /send] | [Send test message] | [To: user's personal email] | [None needed] | [Only affects user] |

**Test Data Configuration:**
```[language]
# Define test data used for Tier 2 validation
TEST_CONFIG = {
    "test_email": "[user's personal email - set in .env as PIV_TEST_EMAIL]",
    "test_prefix": "PIV_TEST_",
    "test_identifiers": []  # Populated during test, used for cleanup
}
```

**Cleanup Procedure:**
```[language]
[Actual executable code that cleans up all Tier 2 test artifacts]
[Must be idempotent - safe to run even if tests partially failed]
```

**Important:** Test data values that are user-specific (email addresses, account IDs) should reference environment variables, not hardcoded values. Document the required env vars.

#### Tier 3: Approval-Required Live (Human in the Loop)

> Costs real credits, has non-trivial side effects, or consumes metered resources. Validation MUST ask user before executing.

| Endpoint | Action | Estimated Cost | Side Effect | Fallback Fixture |
|----------|--------|---------------|-------------|-----------------|
| [POST /generate] | [AI inference call] | [~$0.01-0.05 per call] | [Consumes API credits] | `.agents/fixtures/{tech}-generate.json` |
| [POST /campaign/launch] | [Activates campaign] | [Free but visible] | [Campaign appears in dashboard] | `.agents/fixtures/{tech}-launch.json` |

**Approval Prompt Format** (used by `/validate-implementation`):
```
🔔 Tier 3 Approval Required: [Technology Name]

To validate [scenario name], I need to:
→ Call: [METHOD /endpoint]
→ With: [brief description of test input]
→ Cost: [estimated cost or "free but creates visible record"]
→ Effect: [what happens - credits consumed, record created, etc.]
→ Cleanup: [auto-cleanup available / manual cleanup needed / none needed]

Options:
  [1] Approve - make live call
  [2] Use recorded fixture (last recorded: [date or "none"])
  [3] Skip this test
```

**Response Recording:**
When user approves a Tier 3 call:
1. Execute the API call
2. Save full response to `.agents/fixtures/{technology}-{endpoint-name}.json`
3. Log timestamp and input used
4. On next validation run, offer choice: use recorded response or make fresh call

**Fixture Format:**
```json
{
  "recorded_at": "2026-02-05T14:30:00Z",
  "endpoint": "POST /generate",
  "request": { "input": "test prompt" },
  "response": { "actual response body" },
  "status_code": 200,
  "notes": "Recorded during Phase 1 validation"
}
```

#### Tier 4: Mock Only (Never Live)

> Irreversible, affects real users/customers, or high financial risk. Always use fixtures.

| Endpoint | Why Mock Only | Fixture File | Mock Strategy |
|----------|-------------- |-------------|---------------|
| [POST /bulk-send] | [Sends to real customers] | `.agents/fixtures/{tech}-bulk-send.json` | [Return pre-recorded success response] |
| [DELETE /account] | [Irreversible deletion] | `.agents/fixtures/{tech}-delete-account.json` | [Return pre-recorded confirmation] |

**Mock Implementation:**
```[language]
[Code showing how to mock this endpoint - interceptor, fixture loader, etc.]
[Must return realistic response matching actual API format from Section 3]
```

**Fixture Data:**
```json
[Actual realistic response that matches the API's real response format]
[Include edge cases: empty results, pagination, partial failures]
```

### 9.2 Test Environment Configuration

**Environment Variables Required for Testing:**
| Variable | Purpose | Tier | Example |
|----------|---------|------|---------|
| [API_KEY] | Authentication | All tiers | `sk-test-...` |
| [PIV_TEST_EMAIL] | Tier 2 test recipient | Tier 2 | `marley@personal.com` |
| [SANDBOX_MODE] | Enable sandbox if available | Tier 1-2 | `true` |

**Sandbox Availability:**
- [ ] This API has a sandbox/test mode: [Yes/No]
- If yes: [How to enable - test API key, sandbox URL, env flag]
- If yes: [Differences between sandbox and production responses]
- If no: [Tier 1-2 tests use production API with safe operations]

### 9.3 Testing Sequence

The recommended order for `/validate-implementation` to execute tests:

```
1. Tier 1 (auto-live) → Verify connectivity and auth
   ├── If FAIL → Stop. Integration is broken.
   └── If PASS → Continue

2. Tier 2 (auto-live with test data) → Verify write operations
   ├── Execute test operations
   ├── Verify responses match expected schemas
   ├── Run cleanup procedure
   └── If cleanup FAILS → WARN user, continue

3. Tier 3 (approval-required) → Verify costly operations
   ├── Present approval prompt for each
   ├── If approved → Execute, record response, validate
   ├── If "use fixture" → Load recorded fixture, validate against it
   └── If "skip" → Log as SKIPPED in report

4. Tier 4 (mock only) → Verify agent handles responses correctly
   ├── Load fixtures
   ├── Feed to agent's processing logic
   └── Verify agent behavior matches PRD decision trees
```

---

## Profile Quality Criteria

Each profile must:
- [ ] Follow the exact structure above (commands expect this format)
- [ ] Include ONLY capabilities relevant to this agent's PRD
- [ ] Have working code examples (not pseudocode)
- [ ] Include actual request/response examples from docs
- [ ] Document rate limits with retry strategies
- [ ] Map every PRD capability to specific endpoints
- [ ] Include community-sourced gotchas (not just official docs)
- [ ] **Apply provenance tags to all non-trivial claims**
- [ ] **Populate §6.5 Maintenance & Risk Signals from `gh` queries**
- [ ] **Note MCP server availability in §6**
- [ ] **Classify every endpoint into a testing tier (1-4)**
- [ ] **Capture real fixtures from Tier 1 probes when credentials available**
- [ ] **Provide test data and cleanup procedures for Tier 2**
- [ ] **Document cost estimates and approval prompts for Tier 3**
- [ ] **Include realistic fixture data for Tier 3 fallbacks and Tier 4**
- [ ] **Specify environment variables needed for testing**

**Length guideline**: 200-450 lines per profile. Dense and actionable. The testing specification (§9) typically accounts for 30-40%; maintenance signals (§6.5) ~5-10%.

### PIV-Automator-Hooks Per Profile (If Enabled)

If hooks are enabled, append to each profile file:

```
## PIV-Automator-Hooks
tech_name: [technology name]
research_status: complete
endpoints_documented: [N]
tier_1_count: [N]
tier_2_count: [N]
tier_3_count: [N]
tier_4_count: [N]
gotchas_count: [N]
fixtures_captured: [N]
source_verified_claims: [N]
release_cadence: [weekly|monthly|quarterly|stale]
last_release: [YYYY-MM-DD]
advisories_count: [N]
mcp_server_available: [yes|no]
confidence: [high|medium|low]
```

---

## Final Output

After generating all profiles:

### Summary Report (Terminal Output)

```
## Research Stack Complete

**PRD**: [path]
**Technologies Researched**: [N]

### Profiles Generated

| Technology | Profile | Endpoints | T1 Auto | T2 Test | T3 Approval | T4 Mock |
|-----------|---------|-----------|---------|---------|-------------|---------|
| [Name] | `.agents/reference/[name]-profile.md` | [N] | [N] | [N] | [N] | [N] |

### Key Findings
- [Any critical discovery that affects the PRD or architecture]
- [Any technology limitation that may require PRD revision]
- [Any recommended additional technology not in PRD]

### Next Step
→ Stage 2 (Research) complete. Next: `/clear`, then open a fresh context window and run `/plan-feature` (Stage 3).
```

### Reasoning

Output 4-8 bullets summarizing all research:

```
### Reasoning
- Researched [N] technologies from PRD Section 3
- Generated [N] profiles totaling [N] endpoints documented
- Key finding: [most important discovery]
- Potential issue: [any technology limitation affecting PRD]
```

### Reflection

Self-critique the research (terminal only):
- Are all PRD technologies covered?
- Are profiles accurate based on docs + source + community sources?
- What share of claims are `[source-verified]` vs `[doc-only]`? Is that share honest about confidence?
- Do testing tier classifications make sense for each endpoint?
- Were Tier 1 fixtures captured from real probes, or skipped due to missing credentials?
- Are §6.5 maintenance signals current and the confidence score honest?
- Did alt-comparison run for every PRD Decision? Any flagged for revision?
- Any profile gaps that could cause issues during planning?

---

## Error Handling

### Technology docs not accessible
- Try alternative documentation sources (GitHub README, dev portal)
- Note gaps in profile and flag to user

### Technology appears unsuitable
- If research reveals the technology can't support a PRD capability:
  - Flag immediately in terminal output
  - Suggest alternatives with brief rationale
  - Do NOT silently proceed — this needs user decision

---

## Caching

Source-code analysis, `gh` queries, and live probes are expensive. Cache the raw research artifacts so reruns within a fresh window reuse them.

**Cache layout:**
```
.agents/cache/research/
├── {tech}-{YYYY-MM-DD}.md         # Raw research notes (docs excerpts, source observations)
├── {tech}-gh-{YYYY-MM-DD}.json    # Maintenance signal snapshot from gh CLI
└── {tech}-probes-{YYYY-MM-DD}.json # Tier 1 probe responses
```

**Reuse window:** 14 days. Within this window, reuse cached artifacts and only re-run the **Step 8 Compile** phase (which is cheap) plus a brief delta check (any new releases? new advisories?).

**Bypass cache:** pass `--fresh` to force a full re-research. Use when:
- Profile flagged a serious doc/source disagreement and you want to re-verify
- A technology released a major version since the last cache write
- `/validate-implementation` reports profile drift

Captured fixtures (`.agents/fixtures/`) and final profiles (`.agents/reference/`) are NOT in the cache — they're durable artifacts.

---

## Rerun Policy

**Run once** after PRD creation. Rerun only when:
- New technology added to PRD
- Existing technology swapped for alternative
- Major API version change discovered during implementation
- `/validate-implementation` reveals profile gaps

When rerunning for a single technology:
```bash
/research-stack --only [technology-name]
```
Parse `--only` from arguments to research just that technology and update its profile. Cache (above) is reused unless `--fresh` is also passed.
