# PIV Dev Kit

**AI-Powered Agent Development Framework**

A systematic approach to building AI agents with Claude Code. The PIV (Prime-Implement-Validate) loop ensures every feature is properly planned, implemented, and verified before moving forward. Designed for Opus 4.6 and Agent Teams.

---

## Philosophy

> "Most AI coding failures are context failures, not capability failures."

This framework solves the #1 problem with AI-assisted development: **context loss**. By creating structured documents at each phase, you maintain continuity across sessions, context resets, and even different AI assistants.

**Core Principles:**
- **Plain English over code snippets** - Documents should be readable by humans
- **Context is King** - Every command maximizes useful context
- **Self-contained phases** - Each phase works standalone after `/clear` + `/prime`
- **Human checkpoints** - Discussion before implementation, validation before shipping
- **Scenario-based validation** - Prove the agent behaves correctly, not just that code compiles
- **Agent Teams ready** - Commands parallelize automatically when Agent Teams is available

---

## The PIV Loop

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              PIV LOOP                                    │
│                                                                          │
│   PRIME ────► DEFINE ────► RESEARCH ────► PLAN ────► BUILD ────► VERIFY  │
│     │           │            │              │          │           │      │
│  /prime    /create-prd  /research-stack  /plan     /execute  /validate   │
│                                         -feature              -impl     │
│                                                                          │
│     ◄──────────────────── Feedback Loop ◄────────────────────            │
└──────────────────────────────────────────────────────────────────────────┘
```

| Phase | Purpose | Commands |
|-------|---------|----------|
| **Prime** | Load context, understand project state | `/prime` |
| **Define** | Create agent-native requirements | `/create-prd`, `/create_global_rules_prompt` |
| **Research** | Deep-dive technology stack (run once) | `/research-stack` |
| **Plan** | Create implementation plans per phase | `/plan-feature` |
| **Build** | Execute plans with parallel task support | `/execute` |
| **Verify** | Scenario-based agent validation | `/validate-implementation`, `/commit` |

---

## Quick Start

### 1. Install Commands

Copy the `.claude/commands/` folder to your project:

```bash
# From your project root
mkdir -p .claude/commands
cp -r /path/to/piv-dev-kit/.claude/commands/* .claude/commands/
```

### 2. Create Project Rules

```bash
/create_global_rules_prompt
```

This generates a `CLAUDE.md` file with:
- Tech stack documentation
- Code conventions
- Architecture patterns
- AI assistant instructions
- Agent Teams playbook (if using teams)

### 3. Start the PIV Loop

```bash
# Prime: Understand the project
/prime

# Create agent-native requirements document
/create-prd

# Research all technologies in the PRD (run ONCE)
/research-stack

# Plan first phase
/plan-feature "Phase 1 from PRD"

# Execute the plan (parallelizes with Agent Teams)
/execute .agents/plans/phase-1.md

# Validate against PRD scenarios
/validate-implementation .agents/plans/phase-1.md --full

# Ship it
/commit
```

---

## Commands Reference

### Prime Phase

#### `/prime`
Builds comprehensive understanding of your codebase, discovers technology profiles, and reports development progress.

```bash
/prime                    # Standard analysis
/prime --with-refs        # Include reference documentation
```

**Output:** Project overview, architecture, tech stack, technology research status, recommended next step

---

### Define Phase

#### `/create-prd`
Creates an agent-native Product Requirements Document from conversation context.

```bash
/create-prd              # Creates PRD.md
/create-prd myproject    # Creates myproject.md
```

**Output:** 500-750 line PRD with:
- Agent Identity (personality, decision philosophy, autonomy level)
- Technology Decisions with rationale (feeds `/research-stack`)
- Agent Behavior Specification with decision trees and scenario definitions
- User stories linked to scenarios
- Implementation phases referencing technologies and scenarios

#### `/create_global_rules_prompt`
Generates project-specific CLAUDE.md rules including Agent Teams playbook.

---

### Research Phase

#### `/research-stack`
**NEW** - Deep-dives every technology in the PRD. Run once after PRD creation.

```bash
/research-stack              # Research all technologies from PRD.md
/research-stack PRD.md       # Specify PRD file
/research-stack --only instantly  # Research single technology
```

**Process:**
1. Reads PRD Section 3 (Technology Decisions) to identify technologies
2. For each technology: researches official docs, community knowledge via WebSearch
3. Produces structured profile with auth, endpoints, rate limits, gotchas, validation hooks
4. **Agent Teams**: Spawns parallel researchers (one per technology)

**Output:** `.agents/reference/{technology}-profile.md` per technology

**Run once.** Profiles persist across sessions and are consumed by all downstream commands.

---

### Plan Phase

#### `/plan-feature`
Creates comprehensive implementation plan consuming PRD and technology profiles.

```bash
/plan-feature "Phase 2: Agent Intelligence"
```

**Process:**
1. **Scope Analysis** - Reviews PRD, checks technology profiles exist, outputs recommendations
2. **User Validation** - Discuss approach before planning
3. **Technology Integration** - Reads `.agents/reference/` profiles, maps endpoints to tasks
4. **Plan Generation** - Creates 500-750 line plan with agent behavior specs

**Output:** `.agents/plans/phase-N-feature-name.md` with:
- Technology profiles consumed and key constraints
- Agent behavior implementation (decision trees, scenario mappings)
- Step-by-step tasks referencing profile endpoints
- Validation strategy mapped to PRD scenarios

---

### Build Phase

#### `/execute`
Executes a development plan with intelligent task parallelization.

```bash
/execute .agents/plans/phase-1.md
```

**Process:**
1. Parses plan, loads technology profiles
2. Analyzes task dependencies, builds dependency graph
3. **Agent Teams Mode**: Groups independent tasks into parallel batches, spawns teammates
4. **Sequential Mode**: Executes tasks one at a time (fallback)
5. Tracks progress in `.agents/progress/`
6. Runs validation phase with unit tests

**Agent Teams parallelization:**
- Independent tasks run simultaneously across teammates
- Teammates share code through git push/pull
- Direct messaging for integration questions
- Lead coordinates overall flow

---

### Verify Phase

#### `/validate-implementation`
Scenario-based validation testing agent behavior against PRD definitions.

```bash
# Standard: Syntax + Components + Scenarios
/validate-implementation

# Full: Includes end-to-end pipeline
/validate-implementation --full

# Specific plan
/validate-implementation .agents/plans/phase-2.md --full
```

**Validation Levels:**

| Level | What It Tests | Source |
|-------|---------------|--------|
| Level 1 | Syntax, types, lint | Plan validation commands |
| Level 2 | Unit + component tests | Plan validation commands |
| Level 3 | PRD scenario validation | PRD Section 4.3 scenarios |
| Level 4 | Full pipeline end-to-end | Plan + PRD (--full only) |

**Scenario validation tests:**
- Happy path scenarios from PRD
- Error recovery paths from PRD
- Edge cases from PRD
- Decision tree branch verification
- Technology integration health checks

**Agent Teams**: Parallel validation — one teammate per scenario category.

#### `/commit`
Creates git commits following project conventions.

```bash
/commit                  # Commits staged changes
/commit "feat: message"  # With custom message
```

---

## Developing AI Agents

The PIV framework is optimized for AI agent development. Here's the recommended workflow:

### Phase 1: Foundation

**Goal:** Core infrastructure, tools, basic pipeline

```bash
# 1. Create project structure and CLAUDE.md
/create_global_rules_prompt

# 2. Define requirements (agent-native PRD)
/create-prd
# → Includes Agent Behavior Specification
# → Includes Technology Decisions with rationale
# → Includes Scenario Definitions

# 3. Research all technologies (run ONCE)
/research-stack
# → Produces profiles in .agents/reference/
# → Auth patterns, endpoints, rate limits, gotchas

# 4. Plan foundation
/plan-feature "Phase 1: Foundation Pipeline"
# → Consumes technology profiles
# → Maps PRD scenarios to implementation

# 5. Execute (parallelizes with Agent Teams)
/execute .agents/plans/phase-1.md

# 6. Validate - tests PRD scenarios
/validate-implementation --full
# → Tests happy paths, error recovery, edge cases
# → Verifies decision tree behavior
# → Checks technology integration
```

### Phase 2: Agent Intelligence

**Goal:** Agent reasoning loop, decision trees, tool orchestration

```bash
# Plan agent layer (profiles already available)
/plan-feature "Phase 2: Agent Intelligence"

# Execute
/execute .agents/plans/phase-2.md

# Validate - scenario-based
/validate-implementation --full
```

### Phase 3+: Iteration

```bash
# Each phase follows the same loop:
/plan-feature "Phase N: [Feature]"
/execute .agents/plans/phase-N.md
/validate-implementation --full
/commit
```

---

## Agent Teams Integration

The framework supports Claude Code Agent Teams for parallel execution. Agent Teams is experimental and must be enabled:

```json
// settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### Where Agent Teams Helps

| Command | Without Teams | With Teams |
|---------|--------------|------------|
| `/research-stack` | Sequential per technology | Parallel — one researcher per technology |
| `/execute` | One task at a time | Independent tasks run simultaneously |
| `/validate-implementation` | Sequential scenario testing | Parallel — one validator per scenario category |
| `/orchestrate-analysis` | Sequential agent phases | Parallel phases where dependencies allow |

### How It Works

- **Team Lead** reads the plan and identifies parallelizable work
- **Teammates** are spawned with specific tasks and full context windows
- Teammates share code through **git push/pull** on shared upstream
- **Direct messaging** enables real-time coordination between teammates
- Lead **waits for batch completion** before starting dependent tasks

### Token Considerations

Agent Teams uses significantly more tokens (each teammate = full Claude instance). Best for:
- Plans with 3+ independent tasks
- Research across multiple technologies
- Validation with many scenarios

Not recommended for simple single-file changes or tightly sequential work.

---

## Project Structure

After using PIV commands, your project will have:

```
your-project/
├── .claude/
│   └── commands/           # PIV commands (copy from piv-dev-kit)
├── .agents/
│   ├── plans/              # Implementation plans
│   │   ├── phase-1-foundation.md
│   │   └── phase-2-agent.md
│   ├── validation/         # Validation reports
│   │   ├── phase-1-2026-02-05.md
│   │   └── phase-2-2026-02-05.md
│   └── reference/          # Technology profiles (from /research-stack)
│       ├── instantly-api-profile.md
│       ├── x-api-profile.md
│       └── elevenlabs-profile.md
├── CLAUDE.md               # Project rules (with Agent Teams playbook)
├── PRD.md                  # Agent-native requirements
└── src/                    # Your code
```

---

## Best Practices

### PRD Writing

**Do:**
- Keep to 500-750 lines
- Make Agent Behavior Specification the most detailed section
- Include 8-15 scenario definitions with error paths
- Capture technology decisions with rationale from conversation
- Reference scenarios in user stories

**Don't:**
- Include code snippets (save for plans)
- Make Agent Behavior Specification optional
- Skip error recovery patterns
- Leave technology choices unexplained

### Technology Research

**Do:**
- Run `/research-stack` once after PRD
- Let it research official docs AND community knowledge
- Review profiles for accuracy before planning
- Update profiles if implementation reveals gaps

**Don't:**
- Skip research and guess at API patterns
- Run before the PRD exists
- Rerun for every phase (profiles persist)

### Validation

**Do:**
- Run `--full` before shipping
- Test all PRD scenarios (happy, error, edge)
- Verify decision tree branches
- Check technology integration health

**Don't:**
- Skip scenario validation
- Trust static analysis alone
- Move to next phase with scenario failures
- Ignore error recovery path testing

---

## Command Cheat Sheet

| Command | When to Use | Output |
|---------|-------------|--------|
| `/prime` | Start of session | Terminal summary + next step |
| `/create-prd` | New project/feature | `PRD.md` (agent-native) |
| `/create_global_rules_prompt` | New project setup | `CLAUDE.md` |
| `/research-stack` | After PRD, before planning (once) | `.agents/reference/*.md` |
| `/plan-feature` | Before implementing each phase | `.agents/plans/*.md` |
| `/execute` | After plan approved | Implemented code |
| `/validate-implementation` | After execution | `.agents/validation/*.md` |
| `/validate-implementation --full` | Before shipping | Full scenario results |
| `/commit` | After validation | Git commit |
| `/create_reference` | Need documentation | `.agents/reference/*.md` |
| `/orchestrate-analysis` | Complex codebase analysis | Analysis report |

---

## Troubleshooting

### "Technology profiles not found"
```bash
ls .agents/reference/  # Check what exists
/research-stack        # Generate profiles from PRD
```

### "Plan not found"
```bash
ls .agents/plans/  # Check what exists
/plan-feature "Phase 1"  # Create if missing
```

### Scenario validation failing
- Check PRD Section 4.3 scenarios match implementation
- Verify technology profiles are accurate (rate limits, endpoints)
- Check if external services are reachable
- Review mock strategy in technology profile Section 9

### Context lost after /clear
```bash
/prime  # Reloads context, discovers profiles, shows next step
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
2. Preserve PIV loop philosophy
3. Ensure Agent Teams compatibility (parallel + sequential paths)
4. Test the workflow end-to-end
5. Keep cross-references consistent (PRD sections, profile structure)

---

## License

MIT - Use freely for your agent development projects.

---

## Summary

The PIV Dev Kit transforms AI agent development from chaotic to systematic:

1. **Prime** - Build context that persists
2. **Define** - Agent-native PRD with behavior specs and scenarios
3. **Research** - Deep technology profiles (run once)
4. **Plan** - Technology-informed, scenario-mapped implementation plans
5. **Build** - Parallel execution with Agent Teams
6. **Verify** - Scenario-based validation proving the agent behaves correctly

Every phase produces artifacts that survive context resets, enabling true iterative development with AI assistance.

**Start here:**
```bash
/prime
/create-prd
/research-stack
/plan-feature "Phase 1"
```
