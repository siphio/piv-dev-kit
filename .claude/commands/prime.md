---
description: Prime agent with codebase understanding, technology profiles, and development context
---

# Prime: Load Project Context

## Arguments: $ARGUMENTS

## Objective

Build comprehensive understanding of the codebase by analyzing structure, documentation, technology research profiles, and key files. Produces a context summary that enables all other PIV commands.

## Reference Files Policy

**Default behavior**: Do NOT read reference files (`.agents/reference/`, `ai_docs/`, `ai-wiki/`, or similar reference directories) in full.

**Exception**: Only read reference files if the arguments explicitly request it (e.g., `--with-refs`, `include references`, `read reference files`).

**Technology Profiles**: Always LIST available profiles in `.agents/reference/` but only read their summaries (first 10 lines). Full profiles are consumed by `/plan-feature` and `/execute`.

Check arguments for keywords: `ref`, `reference`, `--with-refs`, `include ref`

## Process

### 1. Analyze Project Structure

List all tracked files:
!`git ls-files`

Show directory structure:
On Linux, run: `tree -L 3 -I 'node_modules|__pycache__|.git|dist|build'`

### 2. Read Core Documentation

- Read CLAUDE.md or similar global rules file
- Read README files at project root and major directories
- Read `.agents/PRD.md` or `PRD.md` (if exists)
- Read any architecture documentation in root or docs/

**Skip these unless arguments explicitly request references:**
- `.agents/reference/` directory (full content)
- `ai_docs/` directory
- `ai-wiki/` directory

### 3. Discover Technology Profiles

Check for research profiles produced by `/research-stack`:

```bash
ls .agents/reference/*-profile.md 2>/dev/null
```

If profiles exist:
- List each profile name and the technology it covers
- Read the first section (Agent Use Case line) from each to understand what's available
- Report profile count and technologies covered
- Note: These are consumed in full by `/plan-feature` and `/execute`

If no profiles exist:
- Note: "No technology profiles found. Run `/research-stack` after PRD creation."

### 4. Identify Key Files

Based on the structure, identify and read:
- Main entry points (main.py, index.ts, app.py, etc.)
- Core configuration files (pyproject.toml, package.json, tsconfig.json)
- Key model/schema definitions
- Important service or controller files

### 5. Understand Current State

Check recent activity:
!`git log -10 --oneline`

Check current branch and status:
!`git status`

### 6. Check Development Progress

If `.agents/plans/` exists:
```bash
ls -t .agents/plans/*.md 2>/dev/null
```
Report which plans exist and their phases.

If `.agents/validation/` exists:
```bash
ls -t .agents/validation/*.md 2>/dev/null
```
Report most recent validation results.

If PRD exists, check the "Current Focus" section for active phase and status.

## Output Report

Provide a concise summary covering:

### Project Overview
- Purpose and type of application
- Primary technologies and frameworks
- Current version/state

### Architecture
- Overall structure and organization
- Key architectural patterns identified
- Important directories and their purposes

### Tech Stack
- Languages and versions
- Frameworks and major libraries
- Build tools and package managers
- Testing frameworks

### Technology Research Status
- Profiles available in `.agents/reference/`: [list or "none"]
- Technologies covered: [list]
- `/research-stack` status: [Complete / Not Run]

### Development Progress
- Active PRD phase: [Phase N or "No PRD"]
- Plans created: [list or "none"]
- Latest validation: [date and result or "none"]
- Progress tracking: [`.agents/progress/` status or "no active execution"]

### Core Principles
- Code style and conventions observed
- Documentation standards
- Testing approach

### Current State
- Active branch
- Recent changes or development focus
- Any immediate observations or concerns

### Recommended Next Step
Based on the project state, suggest the next PIV command:
- No PRD? → "Run `/create-prd` to define requirements"
- PRD exists, no profiles? → "Run `/research-stack` to research technologies"
- Profiles exist, no plans? → "Run `/plan-feature \"Phase 1\"` to start planning"
- Plan exists, not executed? → "Run `/execute .agents/plans/[plan].md`"
- Executed, not validated? → "Run `/validate-implementation`"
- Validated? → "Run `/commit` to ship"

**Make this summary easy to scan - use bullet points and clear headers.**
