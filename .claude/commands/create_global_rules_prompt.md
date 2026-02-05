 when the user runs this command, the Coding Assistant (you) should run the prompt listed below, to help the user to develop there global rules:
---

**PROMPT BEGINS HERE:**

---

Help me create the global rules for my project. Analyze the project first to see if it is a brand new project or if it is an existing one, because if it's a brand new project, then we need to do research online to establish the tech stack and architecture and everything that goes into the global rules. If it's an existing code base, then we need to analyze the existing code base.

## Instructions for Creating Global Rules

Create a `CLAUDE.md` file (or similar global rules file) following this structure:

### Required Sections:

1. **Core Principles**
   - Non-negotiable development principles (naming conventions, logging requirements, type safety, documentation standards)
   - Keep these clear and actionable

2. **Tech Stack**
   - Backend technologies (framework, language, package manager, testing tools, linting/formatting)
   - Frontend technologies (framework, language, runtime, UI libraries, linting/formatting)
   - Include version numbers where relevant
   - Backend/frontend is just an example, this depends on the project of course

3. **Architecture**
   - Backend structure (folder organization, layer patterns like service layer, testing structure)
   - Frontend structure (component organization, state management, routing if applicable)
   - Key architectural patterns used throughout
   - Backend/frontend is just an example, this depends on the project of course

4. **Code Style**
   - Backend naming conventions (functions, classes, variables, model fields)
   - Frontend naming conventions (components, functions, types)
   - Include code examples showing the expected style
   - Docstring/comment formats required

5. **Logging**
   - Logging format and structure (structured logging preferred)
   - What to log (operations, errors, key events)
   - How to log (code examples for both backend and frontend)
   - Include examples with contextual fields

6. **Testing**
   - Testing framework and tools
   - Test file structure and naming conventions
   - Test patterns and examples
   - How to run tests

7. **API Contracts** (if applicable - full-stack projects)
   - How backend models and frontend types must match
   - Error handling patterns across the boundary
   - Include examples showing the contract

8. **Common Patterns**
   - 2-3 code examples of common patterns used throughout the codebase
   - Backend service pattern example
   - Frontend component/API pattern example
   - These should be general templates, not task-specific

9. **Development Commands**
   - Backend: install, dev server, test, lint/format commands
   - Frontend: install, dev server, build, lint/format commands
   - Any other essential workflow commands

10. **AI Coding Assistant Instructions**
    - 10 concise bullet points telling AI assistants how to work with this codebase
    - Include reminders about consulting these rules, following conventions, running linters, etc.

11. **Terminal Output Standards**
    - How AI assistants should format their responses when working on this project
    - Emphasize plain English explanations over code dumps
    - Use structured output (bullets, headers, tables) for scannability
    - Include guidelines like:
      - Explain what you're doing before showing code
      - Use brief summaries before detailed sections
      - Prefer bullet points over paragraphs
      - Only show code when actively implementing (not when explaining)
      - Use status indicators for progress (⚪🟡🟢🔴 or similar)

12. **Service Configuration** (for AI Agent Projects)
    - Include this section only if the project is an AI agent with external service integrations
    - Document which services the agent uses and how to configure them
    - Specify environment variables for API keys and credentials
    - Note any test accounts or sandbox endpoints for validation
    - Reference `.agents/services.yaml` for structured service configuration:
      ```yaml
      services:
        openai:
          auth_env: OPENAI_API_KEY
          health: /models
        postgres:
          auth_env: DATABASE_URL
          skip: false  # include in validation
      ```
    - This enables the validation system to test integrations automatically

13. **Planning Workflow** (for projects using PIV loop)
    - Document the plan-feature two-phase process if the project uses `/plan-feature`:
      1. **Scope Analysis**: Output recommendations with justifications to terminal
      2. **Plan Generation**: Create plan only after user validates approach
    - Explain that recommendations must include WHY:
      - Reference PRD requirements or user stories
      - Reference codebase patterns that inform the choice
      - Explain how the recommendation serves the implementation goal
    - Note the conversational validation checkpoint:
      - User reviews recommendations in terminal
      - Confirms or discusses changes
      - Plan generated with validated decisions baked in
    - This ensures plans are solid before execution begins

14. **Agent Teams Playbook** (for projects using Agent Teams)
    - Include this section if the project uses Claude Code Agent Teams for parallel execution
    - Document the teammate roles available in this project:

    **Teammate Roles:**

    ```markdown
    ### Implementer
    - **Purpose**: Execute implementation tasks from plans
    - **Tools**: Full codebase access, Bash, file operations
    - **Context**: Receives task description + technology profiles + codebase analysis
    - **Output**: Implemented code pushed to shared repo

    ### Validator
    - **Purpose**: Run scenario validation against PRD definitions
    - **Tools**: Test runners, Bash, file read access
    - **Context**: Receives PRD scenarios + plan acceptance criteria
    - **Output**: Validation report with pass/fail per scenario

    ### Researcher
    - **Purpose**: Deep-dive technology documentation and patterns
    - **Tools**: WebSearch, WebFetch, file write
    - **Context**: Receives technology name + PRD capability requirements
    - **Output**: Technology profile in `.agents/reference/`
    ```

    **Agent Teams Conventions:**
    - Each teammate gets a clear, single responsibility
    - Teammates coordinate through git push/pull on shared upstream
    - Direct messaging for integration questions
    - Lead coordinates but delegates implementation
    - One team per session only (experimental limitation)

    **When to Use Agent Teams:**
    - `/execute` with 3+ independent tasks → parallel implementers
    - `/research-stack` with 2+ technologies → parallel researchers
    - `/validate-implementation` with many scenarios → parallel validators
    **When NOT to Use Agent Teams:**
    - Tasks with tight sequential dependencies
    - Simple single-file changes
    - Quick bug fixes
    - When token budget is a concern (each teammate = full Claude instance)

## Process to Follow:

### For Existing Projects:
1. **Analyze the codebase thoroughly:**
   - Read package.json, pyproject.toml, or equivalent config files
   - Examine folder structure
   - Review 3-5 representative files from different areas (models, services, components, etc.)
   - Identify patterns, conventions, and architectural decisions already in place
2. **Extract and document the existing conventions** following the structure above
3. **Be specific and use actual examples from the codebase**

### For New Projects:
1. **Ask me clarifying questions:**
   - What type of project is this? (web app, API, CLI tool, mobile app, etc.)
   - What is the primary purpose/domain?
   - Any specific technology preferences or requirements?
   - What scale/complexity? (simple, medium, enterprise)
   - **Will this project use Agent Teams for parallel execution?**
2. **After I answer, research best practices:**
   - Use WebSearch for current best practices matching the tech stack
3. **Create global rules based on research and best practices**
4. **Include Agent Teams Playbook** if the project will use Agent Teams

## Critical Requirements:

- **Length: 100-500 lines MAXIMUM** - The document MUST be less than 500 lines. Keep it concise and practical.
- **Be specific, not generic** - Use actual code examples, not placeholders
- **Focus on what matters** - Include conventions that truly guide development, not obvious statements
- **Keep it actionable** - Every rule should be clear enough that a developer (or AI) can follow it immediately
- **Use examples liberally** - Show, don't just tell

## Output Format:

Create the CLAUDE.md with:
- Clear section headers (## 1. Section Name)
- Code blocks with proper syntax highlighting
- Concise explanations
- Real examples from the codebase (existing projects) or based on best practices (new projects)

Start by analyzing the project structure now. If this is a new project and you need more information, ask your clarifying questions first.
