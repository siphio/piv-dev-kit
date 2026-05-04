# PIV Dev Kit - Development Rules

This project contains the PIV loop framework commands. These rules govern how to develop and improve the framework itself.

## 1. Project Purpose

This is a **meta-project** - a collection of Claude Code slash commands that implement the PIV (Prime-Implement-Validate) loop methodology. The commands here are used in OTHER projects.

**We are building tools for AI-assisted development, not an application.**

## 2. Core Principles

1. **Plain English over code snippets** - Command outputs should be readable, not walls of code
2. **Context is King** - Every command should maximize useful context while minimizing noise
3. **One stage per context window** - Each of the 5 stages (PRD, Research, Plan, Build, Validate) runs in its own fresh context window. Do not chain stages within a single conversation.
4. **One comprehensive plan, no phases by default** - Phases are an opt-in pattern, not a default. Use phases only when work is genuinely incrementally shippable to users.
5. **Line discipline** - PRDs: 500-750 lines, Plans: 1500-2500 lines (single comprehensive). Profiles: 200-450 lines.
6. **Human checkpoints** - At stage boundaries (between context windows) and at natural milestones within a stage.

## 3. Terminal Output Standards

When writing or modifying commands, ensure outputs follow these rules:

**DO:**
- Use plain English to explain what's happening
- Use bullet points and headers for scannability
- Show status with emojis: ⚪🟡🟢🔴
- Provide brief summaries before detailed sections
- Use tables for structured comparisons

**DON'T:**
- Output large code blocks unless explicitly implementing
- Use technical jargon when plain words work
- Create walls of text without structure
- Include code snippets in PRD outputs (save for plan-feature)

**Example Good Output:**
```
## Phase 2 Analysis Complete

**What I found:**
- 3 API endpoints need implementation
- Authentication pattern exists in `auth/` folder
- Tests follow pytest conventions

**Recommended approach:**
Use the existing HTTP client wrapper and add new endpoints following the pattern in `api/users.py`.

**Ready for questions before planning.**
```

**Example Bad Output:**
```python
# Here's what the implementation might look like:
class APIClient:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.session = aiohttp.ClientSession()

    async def get(self, endpoint: str) -> dict:
        async with self.session.get(f"{self.base_url}/{endpoint}") as resp:
            return await resp.json()
# ... 50 more lines of code
```

## 4. Command File Structure

All commands live in `/commands/` with this structure:

```
commands/
├── prime.md                 # Context loading
├── create-prd.md           # PRD generation
├── plan-feature.md         # Implementation planning
├── execute.md              # Plan execution
├── commit.md               # Git commits
├── create_reference.md     # Reference guide creation
├── create_global_rules_prompt.md  # CLAUDE.md generation
└── orchestrate-analysis.md # Multi-agent analysis
```

## 5. Command Writing Conventions

**Frontmatter:**
```yaml
---
description: Brief description of what this command does
argument-hint: [optional-argument]
---
```

**Section Headers:** Use `##` for main sections, `###` for subsections

**Instructions to Claude:** Write as clear directives, not suggestions
- DO: "Create a summary with these sections..."
- DON'T: "You might want to consider creating..."

**Output Specifications:** Always define:
- Where output goes (file path or terminal)
- Expected format
- Length constraints if applicable

## 6. PIV Philosophy: 5-Stage Linear Flow

The framework implements this flow. Each stage runs in its own fresh context window.

```
Stage 1: PRD          /create-prd
            ↓ /clear
Stage 2: Research     /prime → /research-stack
            ↓ /clear
Stage 3: Plan         /prime → /plan-feature
            ↓ /clear
Stage 4: Build        /prime → /execute
            ↓ /clear
Stage 5: Validate     /prime → /validate-implementation
```

**Key insight:** Most AI coding failures are context failures, not capability failures. With Opus 4.7's 1M context window, each stage gets a full window dedicated to it — no cycling, no per-phase rebuilds, no carry-over noise.

**Why fresh context per stage:**
- PRD work is product thinking; research is technical thinking; plan is architectural thinking; build is implementation thinking; validate is adversarial thinking. Different mindsets benefit from clean breaks.
- Validate especially benefits from zero implementation bias — fresh eye on the same artifacts.
- Cost-efficient: each stage's context is sized for that stage's needs.

**Stage handoffs are file-based:**
- PRD → `PRD.md`
- Research → `.agents/reference/*-profile.md` + `.agents/fixtures/*.json`
- Plan → `.agents/plans/plan.md`
- Build → source code in repo
- Validate → `.agents/validation/*.md`

Every command does one of:
1. **Load context** (`/prime`)
2. **Create artifact** (`/create-prd`, `/research-stack`, `/plan-feature`, `/validate-implementation`)
3. **Apply artifacts to repo** (`/execute`)
4. **Ship** (`/commit`)

**Human checkpoints exist at two levels:**
- **Between stages** (between context windows) — review the artifact before opening the next stage
- **Within a stage at natural milestones** — `/execute` pauses at meaningful breakpoints defined in the plan, not at artificial phase boundaries

## 7. Editing Commands

When modifying existing commands:

1. **Read the full command first** - Understand current behavior
2. **Preserve the philosophy** - Don't break the PIV loop flow
3. **Test the workflow** - Ensure changes work in the full cycle
4. **Update related commands** - If PRD format changes, check plan-feature compatibility

## 8. Length Constraints

| Document | Min Lines | Max Lines | Reason |
|----------|-----------|-----------|--------|
| PRD | 500 | 750 | Human readable, full feature scope |
| Plan (single comprehensive) | 1500 | 2500 | Covers entire PRD in one detailed document; 1M window can hold this plus profiles plus source |
| Technology profiles | 200 | 450 | Dense + actionable; testing tier + maintenance signals included |
| CLAUDE.md | 100 | 600 | Quick reference, not a manual |
| Reference guides (other) | 50 | 200 | Scannable, actionable |
| Validation reports | — | — | No cap; structured report with evidence |

**Why plans grew (1500-2500 vs old 500-750):** the old cap reflected per-phase plans designed to fit a 200k context. With single comprehensive plans covering the full PRD, more density is appropriate. Trim ruthlessly within that range — don't pad.

## 9. Development Commands

```bash
# No build process - these are markdown files

# Test a command manually:
# 1. Open a test project
# 2. Copy command to .claude/commands/
# 3. Run with /command-name
# 4. Verify output meets standards
```

## 10. AI Assistant Instructions

When working on this project:

1. Read this CLAUDE.md first for context
2. Prioritize plain English readability in all outputs
3. Respect line limits — trim ruthlessly if needed
4. Keep the 5-stage flow intact (PRD → Research → Plan → Build → Validate)
5. Test changes mentally through all 5 stages — does the artifact handoff still work?
6. Don't add code snippets to PRD-related outputs
7. Use status emojis consistently (⚪🟡🟢🔴)
8. **No phases by default.** Single comprehensive plan, organized by logical sections (data models, API client, agent loop, error handling). Phases are opt-in only when the work is genuinely incrementally shippable to users.
9. Ensure each stage works standalone after `/clear` + `/prime`
10. When in doubt, optimize for human scannability

## 11. Plan-Feature Workflow

`/plan-feature` produces ONE comprehensive plan covering the entire PRD. No per-phase invocation. No phase boundaries unless the PRD explicitly opts into phases.

**Step 1: Scope Analysis (Terminal Output)**
1. Read the full PRD (all sections)
2. Read all technology profiles in `.agents/reference/`
3. Read all captured fixtures in `.agents/fixtures/`
4. Extract user stories, prerequisites, scope boundaries from the entire PRD
5. Identify decision points from "Discussion Points" sections across the whole PRD
6. Output recommendations with justifications to terminal:
   ```
   ### Recommendations

   1. **[Decision Point]**
      → [Recommendation]
      Why: [Justification - how this serves the goal]
   ```
7. Wait for user validation (confirm or discuss changes)

**Step 2: Plan Generation (File Output)**
- Only proceed after user validates approach
- Single file: `.agents/plans/plan.md`
- Organize by **logical sections** (data models, API client, agent loop, error handling, config), not phases
- Identify **natural milestones** for `/execute` to checkpoint at — meaningful, testable units of work (e.g. "auth wired up", "first end-to-end loop running", "error recovery in place")
- Bake validated decisions into the plan
- Document decisions in NOTES section so executor understands constraints

**Key Principles:**
- Recommendations must include WHY — justification based on PRD requirements, user stories, or codebase patterns
- Natural milestones are NOT phases — they're checkpoint moments inside one continuous build
- The plan should treat the feature as one coherent system, not a sequence of independently shippable chunks (unless the PRD explicitly opts into phases)

## 12. PIV Configuration

Settings that control PIV command behavior across all commands.

| Setting | Default | Description |
|---------|---------|-------------|
| hooks_enabled | false | Append `## PIV-Automator-Hooks` to file artifacts |

**Current Settings:**
- hooks_enabled: false

**Override per command:** Add `--with-hooks` or `--no-hooks` to any command invocation to override the project default for that run.

**How commands check this:** Read this section from CLAUDE.md. If `hooks_enabled: true`, append hooks. If argument contains `--with-hooks`, enable regardless. If `--no-hooks`, disable regardless.

## 13. Prompting & Reasoning Guidelines

All PIV commands use structured reasoning internally. These are the shared patterns.

### CoT Styles

| Style | When Used | Commands |
|-------|-----------|----------|
| Zero-shot | Lightweight/focused tasks | /prime, /commit |
| Few-shot | Complex generation with examples | /create-prd, /create_global_rules_prompt |
| Tree-of-Thought | Decision exploration with multiple approaches | /plan-feature, /orchestrate-analysis |
| Per-subtask | Parallel teammate tasks | /execute, /research-stack, /validate-implementation |

### Terminal Reasoning Summary

Every command outputs a brief `### Reasoning` section to terminal showing the key steps taken:

```
### Reasoning
- Scanned 14 tracked files, identified 3 config patterns
- Cross-referenced PRD Phase 2 with 2 technology profiles
- Gap found: no rate limit handling for X API
- Recommending: add retry logic before planning
```

Rules:
- 4-8 bullet points maximum
- Shows *what was found*, not the full thinking process
- Appears before the main output section

### Reflection Pattern

After main generation, each command performs a brief self-critique:
- Is output aligned with PRD/scenarios/profiles?
- Is it complete — any missing sections or gaps?
- Is it consistent with existing artifacts?

Reflection output goes to **terminal only** — never into file artifacts. Format:

```
### Reflection
- ✅ All PRD scenarios accounted for
- ⚠️ Technology profile for Redis not found — flagged in recommendations
- ✅ Line count within budget (623 lines)
```

### Hook Block Format

When hooks are enabled, append to the **end** of file artifacts:

```
## PIV-Automator-Hooks
key: value
key: value
```

Rules:
- 5-15 lines maximum
- Simple key-value pairs (no nesting, no arrays)
- Parseable with regex: `^([a-z_]+): (.+)$`
- Each command defines its own keys (documented per command)
- **Placement rule**: Hooks are appended to the primary file artifact when the command produces one (e.g. PRD.md, plan.md, profile.md). For commands that output only to terminal (e.g. /prime, /commit, /create_global_rules_prompt), the hooks block appears in terminal output.

### Argument Parsing

Commands that accept flags parse them from `$ARGUMENTS`:
- Strip `--with-hooks` and `--no-hooks` from arguments before processing
- Strip `--reflect` where applicable — currently supported only by `/plan-feature`; other commands ignore it
- Remaining text is the actual argument (filename, phase name, etc.)

### Manual Mode Preservation

When hooks are disabled (the default), commands behave exactly as they did before this enhancement — no visible change in artifacts or terminal output except for the added Reasoning and Reflection sections in terminal.
