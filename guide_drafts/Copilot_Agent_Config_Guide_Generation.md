# The Architect’s Handbook for GitHub Copilot Custom Agents: Configuration, Orchestration, and Ecosystem Integration

## Executive Summary

The software development lifecycle is undergoing a fundamental transformation, shifting from a paradigm of manual coding assisted by stochastic completion tools to one defined by deterministic, agentic workflows. In this new era, the role of the developer evolves from a writer of syntax to an architect of intelligence. GitHub Copilot Custom Agents represent the foundational infrastructure of this shift, enabling engineering teams to encapsulate specialized knowledge, architectural patterns, and governance protocols into persistent, version-controlled artifacts.

This report provides an exhaustive technical analysis of the configuration, syntax, and orchestration of Custom Agents within the GitHub ecosystem. It is designed as a definitive guide for AI agents and human architects tasked with synthesizing .agent.md configuration files. By treating "behavior as code," organizations can standardize complex tasks—ranging from architectural review and test generation to documentation compliance—across distributed engineering teams, thereby reducing cognitive load and enforcing consistency at the algorithmic level.

The analysis that follows explores the hierarchical precedence models that govern agent resolution, the nuanced syntax of YAML frontmatter that defines agent capabilities, and the prompt engineering architectures required to ground agents in repository-specific contexts. Furthermore, it examines the integration of the Model Context Protocol (MCP), a mechanism that transforms the coding agent from an isolated text processor into a connected platform orchestrator capable of interfacing with external databases, cloud infrastructure, and issue tracking systems.

## Chapter 1: The Agentic Paradigm and File System Architecture

The implementation of Custom Agents is not merely a feature addition; it is an architectural decision to embed intelligence directly into the repository structure. Unlike ephemeral chat sessions where context is lost upon closure, Custom Agents persist as part of the repository's

infrastructure. This persistence is achieved through a strict file-system hierarchy that the Copilot runtime scans to discover, load, and prioritize agent definitions. Understanding this physical architecture is the first step in mastering agent deployment.

### 1.1 The Repository-Level Scope: .github/agents/

For agents designed to operate within the specific context of a single project, the definition file serves as the primary integration point. The Copilot runtime enforces a convention-over-configuration approach, looking specifically for files within the .github/agents/ directory.1 This location is significant; the .github folder has traditionally been the home for workflows (GitHub Actions), issue templates, and dependabot configurations. Placing agents here signals that they are integral operational components of the repository, subject to the same version control and code review rigor as the application source code.

The naming convention of these files is critical. While the system accepts .md files, the .agent.md extension is strongly recommended and widely adopted as the standard.3 This specific extension aids in disambiguation, allowing both human developers and IDE tooling to instantly recognize the file's purpose. The filename itself—specifically the string preceding the .agent.md extension—serves as the default unique identifier for the agent. For example, a file named refactor-specialist.agent.md will effectively reserve the identifier refactor-specialist within the local scope.

Agents defined at this level possess a scope restricted to the repository in which they reside. They can read files, analyze directory structures, and execute commands (if permitted) only within the boundaries of that specific codebase. This isolation is a security feature, preventing a project-specific agent from inadvertently modifying or accessing context from adjacent projects in a developer's workspace.

### 1.2 The Organization and Enterprise Scope: .github-private

In large-scale engineering organizations, defining agents repeatedly for every repository violates the DRY (Don't Repeat Yourself) principle and leads to configuration drift. To address this, GitHub provides a mechanism for global agent definition via a centralized "hub" repository. This repository must be named .github-private.5

The structure within this special repository differs slightly from the standard model. Agents must be placed in an /agents/ directory at the root, rather than nested inside a .github folder.5 This subtle distinction is crucial for architects to note during the setup phase. Agents defined here are broadcast to all repositories within the organization or enterprise, provided the user has read access to the .github-private repository.

The visibility of the .github-private repository acts as an access control mechanism. If set to "Internal," all enterprise members can utilize the shared agents. If set to "Private," access can be granularly controlled, allowing organizations to restrict powerful or sensitive agents (e.g., a

@security-audit agent with access to vulnerability databases) to specific teams.

### 1.3 Resolution Logic and Precedence

When a user invokes an agent in the chat interface—for example, by typing @docs—the Copilot runtime must determine which definition to load. The system employs a "shadowing" or "closest-definition-wins" logic to resolve conflicts.7 This hierarchy is designed to allow local overrides of global standards.

The precedence order is strictly defined: 1. Repository Level: Highest priority. An agent defined in the current repository overrides all others with the same name. 2. Organization Level: Medium priority. Agents defined in the organization's .github-private repo. 3. Enterprise Level: Lowest priority. Agents defined at the top-level enterprise scope.

This inheritance model supports powerful governance patterns. An enterprise might define a standard @test-agent that uses a generic testing framework. However, a specific legacy team working on a monolith with a custom test runner can define their own @test-agent in their repo's .github/agents/ directory. For developers on that team, @test-agent will use the local, specialized logic, while developers on other teams continue to use the global standard. This capability ensures that standardization does not come at the cost of flexibility.

## Chapter 2: Configuration Syntax and The YAML Frontmatter

The Custom Agent configuration file is a hybrid artifact. It consists of a YAML frontmatter block, which handles machine-readable configuration and metadata, and a Markdown body, which contains the natural language system prompt.5 This dual structure allows the file to serve both the deterministic needs of the runtime environment and the probabilistic needs of the Large Language Model (LLM).

### 2.1 The YAML Schema

The frontmatter is delimited by triple dashes (---) at the start of the file. Each field within this block modifies the agent's behavior, capabilities, or availability.

| Property | Type | Required | Architectural analysis & behavioral implications |
| --- | --- | --- | --- |
| `name` | String | No | The display name used in the UI. While optional, explicitly setting this is best practice. If omitted, the filename is used. Consistency between filename and name reduces cognitive load.2 |
| `description` | String | Yes | This is functionally the most critical metadata field. It acts as a semantic index. The routing model uses the semantic embedding of this description to determine when to suggest this agent to a user.7 A vague description leads to poor discoverability. |
| `tools` | List/String | No | Defines the "action space" of the agent. It can be a wildcard ["*"] or a restrictive list ["read", "edit"]. This field is the primary mechanism for implementing the Principle of Least Privilege.7 |
| `target` | String | No | Specifies the runtime environment: vscode or github-copilot (Cloud). Defaults to both. This is essential when using environment-specif ic features like handoffs or cloud-only MCP servers.7 |
| `infer` | Boolean | No | Controls the agent's autonomy. If true (default), the main Coding Agent can invoke this agent as a subroutine without explicit user action. Setting this to false creates a "Passive Agent" that only acts when directly addressed.7 |
| `mcp-servers` | Object | No | Org/Enterprise Only. Defines Model Context Protocol servers. This allows the agent to carry its own integration configuration, rather than relying on the user's local setup.7 |
| `handoffs` | List | No | VS Code Only. Defines a state machine for workflow transitions, allowing the agent to suggest the next step in a process (e.g., Planner -> Implementer).10 |
| `model` | String | No | VS Code Only. locks the agent to a specific model architecture (e.g., gpt-4, claude-3.5-sonnet) . Useful for agents requiring high reasoning capabilities vs. speed.2 |

### 2.2 Tooling Configuration and Granular Permissions

The tools property allows architects to define exactly what an agent can and cannot do. This is not merely a feature switch; it is a security boundary.

- read: Grants the ability to traverse the file system and read file contents. In the backend, this maps to tools like view or NotebookRead.7 Almost all agents require this to function.
- edit: Grants the ability to modify files. This includes capabilities like str_replace or rewriting entire files. A "Reviewer" agent might deliberately exclude this tool to ensure it remains a passive observer.7
- search: Enables codebase exploration via grep and glob. Without this, an agent is "blind" to anything not explicitly added to its context window.
- execute: High Risk. This tool allows the agent to execute shell commands (Bash/PowerShell). While powerful for tasks like running tests or linters, it creates a vector for arbitrary code execution. Agents with this permission must be rigorously prompted to avoid destructive commands (e.g., rm -rf).1
- agent: A meta-tool that allows the agent to delegate tasks to other custom agents. This capability is the foundation of the "Router" pattern, where a master agent dispatches work to specialists.7
Architectural Recommendation:

Always start with the principle of least privilege. A documentation agent does not need execute. A planning agent does not need edit.

### 2.3 Environmental Divergence: The target Property

A critical insight for configuration is the divergence between the VS Code runtime and the GitHub.com (Cloud) runtime. The target property allows an agent to be specialized for one environment, or effectively "forked" into two versions.

- target: vscode: This target unlocks local-first features. The handoffs property, which creates UI buttons for workflow transitions, is currently only supported in VS Code. Similarly, the model property allows selection of specific models (like o1-preview or claude-3.5-sonnet) available in the user's IDE.9
- target: github-copilot: This target optimizes for the cloud environment. Crucially, it supports the definition of mcp-servers directly in the YAML for organization-level agents. In VS Code, MCP servers are typically configured in the user's local settings file, but the cloud environment has no "local settings," so the agent must carry its own configuration.9
For hybrid agents intended to work everywhere, omitting the target property is standard. However, this often means sacrificing advanced features like Handoffs to ensure compatibility.

## Chapter 3: Context Engineering and System Prompt Architecture

The Markdown body of the .agent.md file contains the System Prompt. This text is the cognitive architecture of the agent. It defines the persona, the rules of engagement, and the specific knowledge required to operate within the repository. Analysis of thousands of agent configurations reveals that effective prompts are not merely lists of instructions but structured documents that engineer the context for the LLM.1

### 3.1 The 30,000 Character Constraint

The prompt body is limited to 30,000 characters.5 This constraint forces architects to be concise and prioritize high-value information. It implies that one cannot simply dump the entire project documentation into the prompt. Instead, the prompt must act as a map, guiding the agent on how to find information using its read and search tools, rather than containing the information itself.

### 3.2 The Three-Tier Boundary Model

To prevent hallucination, scope creep, and destructive actions, the most effective agents implement a rigorous boundary system. This model categorizes instructions into three distinct

tiers of imperativeness.1

Tier 1: Always Do (Directives) These are non-negotiable operational requirements. They define the "Happy Path" of the agent's workflow.

- Mechanism: "Always write tests to tests/unit/ using Jest."
- Mechanism: "Always follow the naming convention defined in CONTRIBUTING.md."
- Insight: These instructions reduce the search space for the LLM, making its output more deterministic.
Tier 2: Ask First (Guardrails) These instructions cover operations with high collateral impact or irreversibility. They force the agent to pause and seek human consensus.

- Mechanism: "Ask for confirmation before modifying database schema migrations."
- Mechanism: "Ask before adding new dependencies to package.json."
- Insight: This mimics the behavior of a senior engineer mentoring a junior: "You can touch the code, but check with me before you change the database."
Tier 3: Never Do (Constraints) These are hard limitations on the agent's action space. They are critical for security and stability.

- Mechanism: "Never output secrets or API keys."
- Mechanism: "Never modify code in the legacy/ directory."
- Insight: Negative constraints are harder for LLMs to follow than positive instructions due to how attention mechanisms work. It is often more effective to frame these as "Avoid X" or to combine them with strict persona definitions.
### 3.3 Dynamic Context Injection and Meta-Prompting

One of the unique requirements of the custom agent configuration is that it must be "customized for the structure and intent of the repo." This requires a process of "Meta-Prompting"—using Copilot to analyze the repository and generate the agent configuration itself.1

The Repository Analysis Algorithm: To generate a truly context-aware agent, one must extract specific metadata from the codebase. The following prompt strategy simulates this analysis: 1. Extract Project Structure: Identify the locations of source code (src/, lib/), tests (tests/, spec/), and documentation (docs/). 2. Identify Tech Stack: Determine not just the language, but the dialect and version. (e.g., "Java 17 with Spring Boot 3" vs "Java 8 with EE").

3. Extract Command Dictionary: The agent needs to know how to validate its work. The prompt should scrape package.json or Makefile to find the exact commands for building, testing, and linting.

Example Meta-Prompt for Agent Generation:

"Analyze this repository to understand the project structure, testing framework (Jest/Mocha), and build system. Create a .agent.md file for a @test-specialist. The agent should know that tests are located in src/__tests__, the command to run them is npm test, and that we use ts-mockito for mocking. Include 'Always/Ask/Never' boundaries appropriate for a CI/CD environment."

By grounding the agent in the physical reality of the repository (file paths, commands), the architect bridges the gap between the LLM's general knowledge and the project's specific constraints.

## Chapter 4: Ecosystem Integration via Model Context Protocol (MCP)

The Model Context Protocol (MCP) represents the extensibility layer of GitHub Copilot, allowing agents to interface with external systems such as SQL databases, issue trackers (Jira/Linear), and cloud infrastructure (AWS/Azure).6 This integration transforms the agent from a code-editor assistant into a platform orchestrator.

### 4.1 Configuring mcp-servers in YAML

For organization-level agents, MCP servers are defined directly in the configuration file using the mcp-servers property. This allows an agent to "carry" its tools with it, regardless of the user's local environment.

Syntax Structure and Variable Injection: The configuration schema supports robust variable injection to handle authentication securely.7

```yaml
mcp-servers:
 postgres-db:
  type: local

  command: npx
  args: ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
  env:
   DB_PASS: ${{ secrets.DB_PASSWORD }}
```

- type: Typically local (executes a binary/container) or sse (connects to a remote stream).
- command & args: The execution string. This mirrors how one would run the server on the command line.
- env: This is critical for security. The syntax ${{ secrets.VAR }} enables the agent to access repository secrets without hardcoding credentials in the markdown file. This follows the same security model as GitHub Actions secrets.7
### 4.2 Tool Scoping and Namespacing

When an MCP server is attached, its tools are namespaced to prevent collisions. If the server is named github, a tool create_issue is accessed via the identifier github/create_issue.7

In the tools list of the agent configuration, the architect can exercise granular control:

- tools: ["*"]: Enable all tools from all configured servers. This is the default but least secure option.
- tools: ["postgres-db/query_table", "github/get_issue"]: Enable only specific capabilities.
This granular scoping is essential for the Principle of Least Privilege. A @read-only-agent should explicitly not have access to postgres-db/delete_table or github/close_issue. This configuration ensures that even if the LLM hallucinates an intent to delete data, the underlying tool execution layer will block the action.

## Chapter 5: Orchestration, Handoffs, and Multi-Agent Patterns

As agent complexity grows, a single prompt often becomes insufficient to handle the nuance of an entire workflow. The "Orchestrator Pattern" emerges, where complex tasks are decomposed into a sequence of operations performed by specialized sub-agents.11

### 5.1 The Handoffs Mechanism (VS Code)

VS Code supports explicit state transitions called "Handoffs." These are defined in the YAML frontmatter and appear as interactive UI elements (buttons) to the user upon the completion of a turn.10 This feature formalizes the workflow, guiding the user through a predefined process.

Configuration Example:

```yaml
handoffs:
 - label: "Generate Tests"
  agent: "test-specialist"
  prompt: "Generate unit tests for the code created in the previous step. Ensure 100% branch
coverage."
  send: true
 - label: "Review Implementation"
  agent: "security-reviewer"
  prompt: "Review the above code for vulnerabilities, specifically focusing on SQL injection risks."
```

- label: The text displayed on the button.
- agent: The name (or filename ID) of the target agent.
- prompt: The context passed to the next agent. This allows the output of Agent A to become the input of Agent B.
- send: If true, the prompt is executed immediately. If false, the user has a chance to edit the prompt before sending. This "Human-in-the-Loop" design is vital for critical transitions.10
This mechanism enables "Assembly Line" architectures: Architect Agent -> Implementation Agent -> Testing Agent -> Security Agent.

### 5.2 The Router Pattern and the agent Tool

Beyond UI-driven handoffs, agents can theoretically use the agent tool alias to invoke other agents programmatically during their execution.7 This allows for a "Router" or "Meta-Agent" pattern.

A Router Agent's system prompt would be structured as a triage logic: "You are a triage agent. Analyze the user request. If it concerns documentation, invoke @docs-agent. If it concerns testing, invoke @test-agent. Do not attempt to answer the question yourself." For this to work, the sub-agents must have infer: true set in their configuration, allowing them to be called as subroutines. This pattern mimics the "Mixture of Experts" (MoE) model in AI architecture, where a gating network (the Router) directs input to the most specialized expert.

### 5.3 Cross-Agent Memory and Continuity

GitHub Copilot employs a "Cross-agent memory system" that persists context between interactions.1 This is a hidden infrastructure layer that changes how agents should be prompted.

- Mechanism: The system uses Just-in-Time verification. It stores facts with file citations (e.g., "The API base URL is defined in src/config.ts:15"). When an agent is invoked later, the system verifies if the file still exists and if the content matches the citation before injecting the memory into the context window.
- Implication: Agents do not need to be constantly reminded of established patterns if those patterns have been executed and "remembered" previously in the repository. The prompt should simply encourage the agent to "Recall previous architectural decisions" or "Check memory for established patterns."
## Chapter 6: Governance, Security, and Enterprise Management

For large organizations, the proliferation of custom agents introduces a governance challenge. Unregulated agents might hallucinate security policies, bypass established workflows, or use unauthorized tools.

### 6.1 Centralized Management via .github-private

By utilizing the .github-private repository, Enterprise administrators can enforce a "Golden Path" for agent usage.

- Shared Utilities: An organization can define a single @security-audit agent that has access to the organization's proprietary CVE database via a custom MCP server.13 Because this agent is defined in the private repo, its configuration (including the connection details for the CVE database) is managed centrally.
- Version Control as Policy: Since agents are files, they are subject to standard Pull Request workflows. Changes to the @compliance-agent can be protected by code owner rules, requiring approval from the Security Team before any modification is merged. This ensures that the agent's instructions—its "policy"—cannot be altered by unauthorized developers.
### 6.2 Monitoring and Audit Logs

Enterprises can track agent usage via the "AI controls" dashboard and audit logs.5

- Metric: Administrators can filter audit logs by actor:Copilot to see which agents are being invoked and how frequently.
- Optimization: This data feeds back into the prompt engineering lifecycle. If the @refactor-agent is rarely used, or if its sessions are frequently abandoned, it indicates a
need for prompt tuning or better description metadata to improve discoverability.

## Chapter 7: Practical Implementation: The Agent Archetypes

This section provides complete, docs-accurate templates for the most common agent archetypes. These templates are synthesized from the best practices identified in the research 1 and are designed to be dropped directly into a .github/agents/ directory.

### 7.1 The Documentation Specialist (docs.agent.md)

Use Case: Maintaining README.md, writing JSDoc/Docstrings, and updating CONTRIBUTING.md.

```md
---
name: docs-specialist
description: Expert technical writer for maintaining project documentation and API references.
tools: ["read", "search", "edit"]
target: vscode
infer: true
---

# Identity
You are a senior technical writer with expertise in and. You prioritize clarity, conciseness, and accuracy.
You follow the Diátaxis framework (separating content into Tutorials, How-to, Reference, and
Explanation).

# Context
- Documentation lives in `/docs`.
- API references are generated from source comments.
- The project requires all public functions to have JSDoc/Docstring comments.

# Capabilities & Boundaries
## Always Do
- Use Markdown for all output.
- Run `npm run lint:docs` (if available) to verify formatting after edits.
- Check existing documentation (`/docs`) to maintain style consistency before writing new content.

## Ask First
- Before creating new top-level documentation files.
- Before deleting sections of existing documentation that might be referenced elsewhere.

## Never Do
- Never modify logical code files (`.ts`, `.py`) except for adding comments.
- Never invent API features that do not exist in the source code.
- Never use absolute paths in links; always use relative paths.

# Workflow
1. Read the source code to understand functionality.
2. Read the existing documentation to understand the structure.
3. Draft the documentation update.
4. Verify that links (relative paths) are valid.
```

### 7.2 The Testing Specialist (tests.agent.md)

Use Case: Increasing code coverage, fixing flaky tests, and generating regression suites.

```md
---
name: test-specialist
description: QA engineer focused on unit testing, integration testing, and coverage analysis.
tools: ["read", "search", "edit", "execute"]
handoffs:
 - label: "Review Security"
    agent: "security-agent"
  prompt: "Review the new tests to ensure they do not expose sensitive data in logs."
---

# Identity
You are a Quality Assurance engineer. You believe in "Test Driven Development" and high code
coverage. You are skeptical and always look for edge cases.

# Repository Standards
- Framework: Jest (for unit), Playwright (for E2E).
- Test Location: `__tests__` directories adjacent to source files.
- Naming Convention: `*.test.ts`.

# Boundaries
## Always Do
- Write deterministic tests (mock time, random seeds).
- Use `describe` blocks to group tests logically.
- Run tests using `npm test -- <filename>` after writing them to ensure they pass. *Note: You have the
execute tool, use it.*

## Ask First
- Before adding new testing libraries or dependencies.

## Never Do
- Never remove a failing test without understanding why it fails.
- Never use `any` types in test mocks unless strictly necessary.
- Never modify production code (`src/`) to make a test pass.

# Tool Usage
- Use `execute` to run the test suite: `npm test`.
- Use `read` to analyze the implementation before writing the test.
```

### 7.3 The Refactoring Specialist (refactor.agent.md)

Use Case: Modernizing legacy code, splitting large functions, and applying design patterns.

```md
---
name: refactor-specialist
description: Specialized agent for architectural refactoring and legacy code modernization.
tools: ["read", "edit", "search", "execute"]
target: vscode
infer: false
---

# Identity
You are a Principal Software Architect. You value maintainability, readability, and the SOLID principles.
You prefer small, composable functions over large, monolithic ones.

# Context
- The codebase uses.
- We are migrating from to.

# Capabilities & Boundaries
## Always Do
- Ensure that refactored code passes all existing tests.
- Break down large functions into smaller helper functions.
- Add type definitions where they are missing.

## Ask First
- Before changing the public API signature of a module (this breaks consumers).
- Before deleting "dead code" (it might be used dynamically).

## Never Do
- Never change the business logic behavior—only the structure.
- Never introduce new external dependencies for a simple utility.

# Workflow
1. Read the file to be refactored.
2. Search for usages of the exported functions to understand impact.
3. Propose a plan (Chain of Thought).
4. Apply the refactoring iteratively.
5. Run tests (`npm test`) to verify no regressions.
```

## Chapter 8: Future Trajectories and Spec-Driven Development

The evolution of Custom Agents is moving toward "Spec-Driven Development".1 In this model, the agents.md file becomes the operational hub for the project, and the agents act as the executors of the specification.

### 8.1 The AGENTS.md Standard

An emerging pattern in the ecosystem is the use of a root-level AGENTS.md (plural) file. While the Copilot runtime looks for individual .agent.md files in .github/agents/, the community is adopting a root-level AGENTS.md file as a unified "README for Robots".4

This file contains high-level architectural summaries, style guides, and project maps specifically formatted for LLM consumption. Custom agents are then instructed in their individual system prompts to "Always read AGENTS.md before starting a task." This creates a shared context layer, ensuring that the @test-agent and the @docs-agent both operate from the same understanding of the project's core philosophy.

### 8.2 Self-Healing and Recursive Improvement

With the execute tool and the infer property, agents are approaching a state of closed-loop iteration. A properly configured agent can now: 1. Write code (edit). 2. Run tests (execute). 3. Read the error log (read). 4. Self-correct the code (edit). 5. Pass the tests.

This loop, defined purely through the configuration file and prompt, represents the transition from "Copilot" (assistant) to "Autopilot" (agent). The architect's role is to define the constraints of this loop (e.g., "Max 3 retry attempts") to prevent infinite resource consumption.

## Conclusion

GitHub Copilot Custom Agents represent a pivotal maturity point in AI-assisted development. By moving context and capability definitions from the ephemeral chat window into persistent, version-controlled configuration files, engineering teams can engineer reliable, repeatable, and governable AI workflows.

The architect's role is to define these agents not as magic boxes, but as deterministic software components. Through precise YAML configuration, context-aware prompting, and the strategic use of MCP and Handoffs, developers can build a "synthetic team" that scales expertise across the organization. The .agent.md file is no longer just documentation; it is the source code of the development process itself.

## Appendix A: Configuration Reference Cheat Sheet

File Location (Repo)                             .github/agents/*.agent.md

File Location (Org)                              .github-private/agents/*.agent.md

Max Prompt Length                                30,000 characters

Supported Tools                                  read, edit, search, execute, agent

Env Variables                                    ${{ secrets.VAR }} supported in

mcp-servers

Handoffs Support                           VS Code Only (currently)

MCP Config Support                         Org/Enterprise Only (via YAML), Repo via settings

File Extension                             .agent.md (Recommended), .md (Supported)

Works cited

1. How to write a great agents.md: Lessons from over 2,500 repositories - The GitHub Blog, accessed January 18, 2026, https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-le ssons-from-over-2500-repositories/ 2. accessed January 18, 2026, https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/cre ate-custom-agents#:~:text=Open%20GitHub%20Copilot%20Chat%20in,custom %20agent%20profile%20in%20the%20. 3. Your first custom agent - GitHub Docs, accessed January 18, 2026, https://docs.github.com/en/copilot/tutorials/customization-library/custom-agents/ your-first-custom-agent 4. awesome-copilot/AGENTS.md at main · github/awesome-copilot ..., accessed January 18, 2026, https://github.com/github/awesome-copilot/blob/main/AGENTS.md 5. About custom agents - GitHub Docs, accessed January 18, 2026, https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-custom-agents 6. Custom agents for GitHub Copilot - GitHub Changelog, accessed January 18, 2026, https://github.blog/changelog/2025-10-28-custom-agents-for-github-copilot/ 7. Custom agents configuration - GitHub Docs, accessed January 18, 2026, https://docs.github.com/en/copilot/reference/custom-agents-configuration 8. AGENTS.md, accessed January 18, 2026, https://agents.md/ 9. October 2025 (version 1.106) - Visual Studio Code, accessed January 18, 2026, https://code.visualstudio.com/updates/v1_106 10.Custom agents in VS Code, accessed January 18, 2026, https://code.visualstudio.com/docs/copilot/customization/custom-agents 11. Agent mode 101: All about GitHub Copilot's powerful mode - The GitHub Blog, accessed January 18, 2026, https://github.blog/ai-and-ml/github-copilot/agent-mode-101-all-about-github-c opilots-powerful-mode/

12.orchestrator-agent-creation-guide.md - GitHub Gist, accessed January 18, 2026, https://gist.github.com/gc-victor/1d3eeb46ddfda5257c08744972e0fc4c 13.GitHub's official MCP Server, accessed January 18, 2026, https://github.com/github/github-mcp-server
