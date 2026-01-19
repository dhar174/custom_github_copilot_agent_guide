Guide to Writing GitHub Copilot Custom Agent Configurations1. Context & PurposeThis guide is strictly for AI agents tasked with generating or modifying GitHub Copilot Custom Agent configuration files. These files enable the "Agent Mode" in GitHub Copilot to adopt specific personas, utilize defined toolsets, and adhere to repository-specific strictures.The primary goal of these configurations is to move beyond generic AI assistance into specialized, deterministic workflows. By defining an agent, you effectively "freeze" a specific set of instructions, constraints, and capabilities into a reusable persona. This ensures consistency across team members and prevents "context drift" (where the model forgets project rules during long conversations) by re-injecting the system prompt at the start of every agent session.Location: All agent files must be placed in /.github/agents/. The Copilot extension specifically monitors this directory.Naming Convention: Lowercase, kebab-case, ending in .agent.md (e.g., lnf-lead.agent.md, db-migrator.agent.md).2. File Structure: The .agent.md FormatEvery agent file consists of two distinct parts:YAML Frontmatter: Configuration metadata, tool permissions, and routing logic.System Prompt (Markdown): The agent's cognitive architecture, constraints, and workflow.Part A: YAML FrontmatterThe frontmatter defines how the Copilot infrastructure treats the agent. It is the control layer that sits above the LLM.---
name: <agent-name>          # MUST match filename (e.g., lnf-lead for lnf-lead.agent.md)
description: <string>       # Concise summary of capabilities (shown in the agent selection UI)
target: github-copilot      # Standard target for this environment
infer: false                # CRITICAL: Set to false to force explicit invocation/orchestration
tools: [<tool_list>]        # Array of allowed tools (see below)
metadata:                   # Custom tags for organization and RAG filtering
  project: "LNF"
  role: "lead"              # or "specialist"
  scope: "all"              # or specific domain like "notebook", "rag"
---

Critical Fields Explainedname: Must be unique, URL-safe, and match the filename prefix. This is the handle users will type (e.g., @lnf-lead) to invoke the agent.description: This text appears in the GitHub Copilot UI when a user is selecting an agent. It should be punchy and explain what the agent does, not how it does it.infer: false: This is a specific pattern to ensure agents are deterministic.If true (or omitted), Copilot might automatically select this agent based on vague user intent.If false, the agent is only activated when explicitly called by the user (@agent) or by an orchestrator agent using the agent tool. This is preferred for complex, multi-agent systems to prevent 'agent confusion' and ensure the orchestrator remains the single source of truth for planning.tools: This array defines the action space of the agent.Orchestrators: Must include "agent" (to delegate tasks). Common additions: "read", "search", "github/*". They typically do not need edit if their job is purely management.Specialists: Specific capabilities only."edit": Allows modifying files (essential for coding agents)."execute": Allows running terminal commands (essential for testing/linting)."web": Allows browsing the internet (essential for research).Wildcards: github/* allows all GitHub-related MCP tools; playwright/* allows browser automation. Warning: Granting * (all tools) is discouraged as it dilutes the agent's focus.Part B: System Prompt (Markdown Body)The body is the "brain" of the agent. It must be structured to prevent hallucination and enforce strict adherence to the project's implementation plan. Unlike a standard chat, this prompt persists throughout the session.Standard Sections & Best PracticesPersona Definition:Start with "You are $$Role Name$$..."Define the exact scope (e.g., "You are the technical lead for Project X," or "You are the QA Specialist implementation Phase 4...").Why: This "primes" the model to adopt a specific tone and knowledge base.Primary Goals & Responsibilities:Bulleted list of high-level objectives.Differentiate between Operational Goals (e.g., "Plan the work") and Technical Goals (e.g., "Implement the RAG pipeline").Example: "Keep work aligned to the implementation plan," "Break work into small PR-sized chunks."Scope & Constraints (Crucial):Define what the agent cannot do. Negative constraints are harder for LLMs to follow, so be explicit and explain why or what to do instead.Weak Constraint: "Don't break the build."Strong Constraint: "Do not commit broken code. Always run npm test before finalizing your output."Example: "Avoid network calls at runtime," "Do not attempt to write vector store indexing yourself; delegate to lnf-rag."Delegation Strategy (Orchestrators Only):Mandatory for Lead Agents: Explicitly list available sub-agents and their triggers.The "Tool Handoff" Rule: "You cannot 'become' another agent. You must invoke the agent tool."Map: Provide a markdown table mapping Task Type → Agent Name → Usage Prompt Example. Providing a specific prompt example ensures the orchestrator speaks the specialist's language.Example: "If user asks for database changes -> call @db-agent."Repo Conventions:Hard rules for code generation. Instead of repeating generic PEP-8 rules, focus on project-specific patterns.Examples: "All source code goes in src/", "Use Pydantic for typing", "Always add tests in tests/ mirroring the source structure."Definition of Done / Quality Gates:Criteria the agent must verify before reporting success.Self-Correction: Instruct the agent to review its own work.Example: "Notebook validates with nbformat", "Tests added and passing," "Linter returns exit code 0."3. Agent Archetypes & PatternsType 1: The Orchestrator (e.g., lnf-lead)Function: Parses user intent, plans the workflow, and delegates to specialists. It rarely writes complex implementation code itself. It acts as a router and a manager.Tools: ["agent", "read", "search", "execute"] (Note: agent tool is mandatory).Key Prompt Feature: A "How to Delegate" section.Pitfall to Avoid: The Orchestrator trying to "hallucinate" the output of a sub-agent. It must wait for the tool result.<!-- end list -->## How to Delegate (CRITICAL)
- You cannot "become" another agent.
- Map user requests to these specialists:
  - RAG Tasks -> `lnf-rag`
  - Notebooks -> `lnf-notebook`

Type 2: The Specialist (e.g., lnf-notebook)Function: Executes a specific phase or technical domain. Deeply knowledgeable about specific libraries (e.g., nbformat, langgraph).Tools: ["edit", "execute", "web", "read"] (Usually lacks "agent" tool to prevent circular delegation loops).Key Prompt Feature: "Constraints" and "Scope" specific to the tech stack.<!-- end list -->Constraints:
- Generated notebooks must be runnable in Google Colab.
- Use `nbformat` v4.

Type 3: The Quality Assurance / Reviewer (e.g., lnf-qa)Function: Critiques code, runs complex test suites, and enforces security standards. It does not generate features, only verifies them.Tools: ["read", "execute", "github/issues"].Key Prompt Feature: "Strictness" and "Verification Protocols".Goal: "Reject code that does not meet coverage metrics."4. Writing Checklist for AI AgentsWhen generating a new .agent.md file, verify the following 10 points:$$ $$ Filename matches name: my-agent.agent.md → name: my-agent.$$ $$ infer: false is set: Ensure the agent is deterministic.$$ $$ Tools are sufficient but minimal: Don't give edit to an agent that only needs to read and delegate.$$ $$ Delegation is explicit: If it's a lead agent, are the sub-agents listed by exact name in the prompt?$$ $$ Repo Standards included: Did you copy the standard src/ layout and Pydantic rules?$$ $$ Environment Awareness: Did you include the "Environment Note" (headless container, screenshot usage) if the agent uses browser tools?$$ $$ Negative Constraints: Are "Don't do X" rules accompanied by "Do Y instead"?$$ $$ Project Metadata: Are role and scope defined in the YAML?$$ $$ Definition of Done: Is there a clear stop condition for the agent?$$ $$ Self-Correction: Does the prompt instruct the agent to fix its own errors before asking the user?5. Example TemplatesTemplate: Orchestrator Agent---
name: project-lead
description: Orchestrates project implementation; delegates tasks and enforces standards.
target: github-copilot
infer: false
tools: ["agent", "read", "search", "edit", "execute"]
metadata:
  role: "lead"
  project: "core-platform"
---

You are the **Lead Architect** for [Project Name].

## Goals
- Analyze implementation plans and break them down into steps.
- Delegate specific coding tasks to specialists.
- Verify integration tests pass after changes.

## Delegation (CRITICAL)
Use the `agent` tool to assign work. **Do not write implementation code yourself.**
- DB Schema -> `db-specialist`
- Frontend -> `ui-specialist`
- Security -> `sec-audit`

## Standards
- Enforce strict typing (Python 3.10+ type hints).
- No direct commits to main; create feature branches.
- Ensure all PRs have a description.

Template: Specialist Agent---
name: db-specialist
description: Handles all database schema migrations and query optimization.
target: github-copilot
infer: false
tools: ["read", "edit", "execute", "github/*"]
metadata:
  role: "specialist"
  domain: "database"
---

You are the **Database Specialist**. You are an expert in PostgreSQL, SQLAlchemy, and Alembic.

## Scope
- Write SQLAlchemy models in `src/database/models.py`.
- Generate and apply Alembic migrations.
- Optimize SQL queries for performance.

## Constraints
- **Do not** modify frontend code (JS/React).
- **Do not** change API routes.
- Ensure all queries are `async`.
- Always verify migration rollback scripts.

## Definition of Done
- Migrations pass locally (`alembic upgrade head`).
- Models have full docstrings.
- No Pydantic validation errors in tests.

