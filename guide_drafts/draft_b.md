# GitHub Copilot Custom Agents: Authoring Agent Profiles in `.github/agents/`

This guide is written **for AI agents** (and the humans supervising them) who need to **create or refine GitHub Copilot Custom Agent profiles** (“custom agents”) that live in a repo under `/.github/agents/`, using GitHub’s documented agent-profile format and behavior. 

It’s also intentionally aligned to the **structure style** in your two uploaded examples—one “lead/orchestrator” and one “specialist”—so you can follow the same format while still being doc-accurate.  

---

## 1) What GitHub means by “custom agents” (and what an “agent profile” is)

**Custom agents** are defined by **Markdown files (“agent profiles”) with YAML frontmatter** that specify:

* an identity (`name`, `description`)
* behavioral instructions (your prompt text in Markdown)
* optional tool scoping (`tools`)
* optional auto-selection control (`infer`)
* optional environment targeting (`target`)
* and, for org/enterprise profiles, optional MCP server configuration (`mcp-servers`) 

Assigning a custom agent to a task (or selecting it in an agent picker) is how its behavior gets instantiated. 

**Where you can use them:** GitHub’s docs say once created, custom agents are available wherever Copilot coding agent is available, including GitHub.com surfaces (agents panel/tab, issue assignment, PRs), Copilot CLI, and IDEs (with some features in preview depending on IDE). 

---

## 2) Where agent profiles live (repo vs org/enterprise)

### Repository-level agents

Put profiles here:

* `/.github/agents/<CUSTOM-AGENT-NAME>.agent.md` (this is what GitHub’s UI template creates) 
* Docs also refer to repo-level profiles as `/.github/agents/CUSTOM-AGENT-NAME.md` (same concept: repo-local agent profiles) 

**Practical recommendation:** Use the `.agent.md` suffix for consistency with GitHub’s creation flow and templates. 

**Filename constraints:** the filename (before `.agent.md`) may only contain: `. - _ a-z A-Z 0-9`. 

### Organization/enterprise-level agents

These go in a `.github-private` repository at:

* `/agents/<CUSTOM-AGENT-NAME>.md` (note: *not* under `.github/` at org/enterprise level) 

---

## 3) Agent profile anatomy (YAML frontmatter + Markdown prompt)

An agent profile is:

1. `---` YAML frontmatter `---`
2. followed by Markdown content that becomes the agent’s prompt. 

### 3.1 Minimal shape (docs-accurate)

GitHub says the simplest profile includes **name, description, prompt**, and optionally tools. 

### 3.2 Character limit

The prompt (Markdown below the frontmatter) can be **up to 30,000 characters**. 

### 3.3 YAML properties you can use (GitHub.com “coding agent”)

Below are the properties that matter most for **Copilot coding agent on GitHub.com** (the environment targeted by `target: github-copilot` in your examples).  

| Property      |                                        Type | Required | Meaning / behavior                                                                                                                |
| ------------- | ------------------------------------------: | -------: | --------------------------------------------------------------------------------------------------------------------------------- |
| `description` |                                      string |        ✅ | Required short explanation of what the agent does.                                                                                |
| `name`        |                                      string |        ❌ | Optional. If omitted, defaults to filename (without `.md` / `.agent.md`).                                                         |
| `tools`       | list of strings (or comma-separated string) |        ❌ | Tool allowlist. If omitted, agent gets **all available tools**. Supports `["*"]`, `[]`, or specific lists.                        |
| `infer`       |                                     boolean |        ❌ | If `false`, agent must be manually selected (no auto-selection). Default is `true` if omitted.                                    |
| `target`      |                 `vscode` | `github-copilot` |        ❌ | Restrict where it shows up. Omit for both environments.                                                                           |
| `metadata`    |                      object of string pairs |        ❌ | Freeform annotation for your own labeling/filters.                                                                                |
| `mcp-servers` |                                      object |        ❌ | **Org/enterprise** profiles can configure MCP servers here. Repo-level profiles can’t configure MCP servers directly in-profile.  |

### 3.4 Properties that are *ignored* on GitHub.com

GitHub’s **custom agents configuration** reference explicitly states that these IDE-focused properties are currently **not supported for Copilot coding agent on GitHub.com** and are ignored:

* `model`
* `argument-hint`
* `handoffs` 

This matters because you may want cross-environment profiles; you can include fields that only apply in IDEs and GitHub.com will ignore them safely.

---

## 4) Tool scoping that actually works (and how to write `tools:` correctly)

### 4.1 How `tools:` is processed

GitHub documents three primary patterns: 

* **All tools**: omit `tools` or set `tools: ["*"]`
* **No tools**: `tools: []`
* **Specific allowlist**: `tools: ["read", "edit", "search"]` (plus MCP tools by name)

Unrecognized tool names are ignored, which is specifically designed to keep profiles compatible across environments and products. 

### 4.2 Tool aliases (the ones you’ll use constantly)

GitHub provides tool aliases (case-insensitive), and maps them to the underlying coding agent tools. Key ones: 

* `read` (view file contents)
* `edit` (apply edits; underlying edit mechanics may vary)
* `search` (grep/glob-like search)
* `execute` (run shell commands)
* `agent` (invoke a different custom agent)
* `web` (documented, but “currently not applicable” for coding agent on GitHub.com) 

### 4.3 MCP tools in `tools:` (repo vs org/enterprise)

* **Repo-level:** You *cannot* define MCP servers inside the agent profile, but the agent can still access MCP tools configured in the repo’s Copilot settings; you can allowlist those tools by name in `tools:`. 
* **Org/enterprise-level:** You *can* configure MCP servers in the profile via `mcp-servers`. 

### 4.4 Namespacing MCP tools (and wildcarding)

GitHub documents:

* `some-mcp-server/some-tool`
* `some-mcp-server/*` to allow all tools from that server 

### 4.5 “Out-of-the-box” MCP servers (good to know)

GitHub’s reference calls out built-in MCP servers you can reference with namespacing: 

* `github/*` (read-only tools; token scoped to source repo)
* `playwright/*` (localhost-only browsing; token scoped)

Your lead example explicitly enables both as part of its toolset. 

---

## 5) Designing multi-agent teams (lead/orchestrator + specialists)

GitHub supports invoking custom agents via the `agent` tool alias. 
Your uploaded orchestrator profile demonstrates the exact pattern you want in a “team” design:

* a **lead** agent that decomposes work and delegates to specialists 
* specialists with narrow scopes and crisp quality gates 

### 5.1 Pattern: Lead agent as a “router + reviewer”

Strong lead-agent responsibilities (mirroring your example):

* **Orchestration:** break requests into PR-sized chunks with acceptance criteria 
* **Delegation:** map request-types to named specialists and invoke them via `agent` 
* **Verification:** review specialist outputs and run tests / smoke checks using `execute` 

### 5.2 Pattern: Specialist agents as “single-phase owners”

Your notebook specialist is a great “single phase” example:

* declares phase ownership (“Phase 4”) 
* lists concrete deliverables (exact modules/files) 
* includes constraints and quality gates 

### 5.3 When to set `infer: false`

Docs: `infer` controls whether Copilot can auto-select an agent; `false` requires manual selection. 
Common team design heuristics:

* **Specialists:** `infer: false` (avoid accidental auto-selection)
* **Lead/router:** either `infer: false` (manual “I’m orchestrating now”) *or* `true` if you want Copilot to auto-pick it for broad tasks (depends on noise tolerance)

Your examples set `infer: false` on both, which is consistent with “manual, intentional selection.”  

---

## 6) Writing the prompt body: a high-performing structure (aligned to your examples)

Below is a “section blueprint” that matches your orchestrator/specialist style, but is generalized so an AI agent can fill it in per-repo.

### 6.1 Recommended headings for **any** agent profile prompt

1. **Role statement (1–3 sentences)**

   * what the agent is
   * what success looks like
   * what’s out of scope

2. **Primary goals (bulleted)**

   * list 3–7 objectives that can be validated

3. **Scope boundaries**

   * explicit “do not” rules (e.g., “don’t refactor unrelated code”)

4. **Inputs to consult**

   * name the repo docs/files the agent should read first (README, ARCHITECTURE.md, CONTRIBUTING.md, etc.)

5. **Workflow**

   * Analyze → Plan → Execute → Review/Repair → Final test
   * your lead example does this well 

6. **Repo conventions**

   * language/tooling constraints
   * directory layout
   * typing/linting/test expectations 

7. **Quality gates / Definition of done**

   * measurable checks
   * what to run with `execute`
   * what artifacts must exist 

8. **Delegation / escalation (lead only)**

   * task-to-agent routing table
   * rules like “you cannot become another agent; you must invoke the agent tool” 

### 6.2 Tool-usage rules worth encoding explicitly

Because tool availability can be broad by default, it’s often better to encode “how to use tools responsibly” in the prompt body, even if you allowlist many tools.

Examples (generalized):

* “Always `read` before `edit`.”
* “Prefer minimal diffs; avoid drive-by refactors.”
* “Use `search` to locate canonical patterns before implementing new ones.”
* “Use `execute` to run the narrowest test suite that proves the change.”
* “If web access is not applicable/available, don’t rely on it.” (GitHub documents `web` as currently not applicable for coding agent.) 

---

## 7) A repo-adaptive authoring algorithm (for AI agents generating these profiles)

When you (the AI agent) are asked to create a set of custom agents “based on repo structure and intent,” follow this repeatable pipeline:

### Step A — Discover the repo’s “shape”

1. Read root docs: README, CONTRIBUTING, SECURITY, ARCHITECTURE (if present)
2. Identify:

   * primary language(s)
   * build system / package manager
   * test runner(s) and commands
   * CI workflows (look in `.github/workflows/`)
   * folder structure ownership (e.g., `src/`, `packages/`, `apps/`, `docs/`)
3. Extract conventions to hard-code in the lead agent’s “Repo Conventions” section.

### Step B — Propose a team map

Start small; add specialists only where it reduces cognitive load. Common partitions:

* **lead/router**
* **core architecture**
* **domain feature area A**
* **domain feature area B**
* **tests/QA**
* **docs**
* **security**
* **UI** (if applicable)

Your lead example’s mapping-table approach is ideal here. 

### Step C — Choose tools (principle of least privilege)

* If you *want* the agent to modify code: include `edit` and usually `read` + `search`.
* If you want it to validate behavior: include `execute`.
* If it must delegate: include `agent`.
* Add MCP tools (`github/*`, `playwright/*`) only when needed and when the environment supports them. GitHub documents these MCP server names and wildcard usage. 

### Step D — Control discoverability

* Set `infer: false` for “dangerous” agents (broad code-writing powers), or narrow specialists.
* Consider leaving `infer: true` only for “safe” or highly unambiguous agents (e.g., docs-only, test-only).

### Step E — Write prompts with enforceable constraints

Prefer:

* explicit deliverables (“create `src/foo/bar.py` with X, update tests”)
* clear non-goals
* quality gates that map to actual commands

---

## 8) Copy/paste templates (lead + specialist), modeled after your examples

These templates intentionally mirror the *shape* of your uploaded files (YAML → role → goals → workflow/quality gates), but with placeholders you can fill per repository and per team design.  

> **Important:** GitHub.com coding agent ignores some IDE-only fields (`model`, `handoffs`, etc.), so keep the GitHub.com template limited to documented properties unless you deliberately want cross-IDE behavior. 

---

### 8.1 Lead / Orchestrator template (`.github/agents/<repo>-lead.agent.md`)

```markdown
---
name: <repo>-lead
description: Leads <PROJECT>; decomposes work, delegates to specialists, enforces repo standards, and verifies changes.
target: github-copilot
infer: false
tools: ["agent", "read", "search", "edit", "execute", "github/*"]
metadata:
  project: "<PROJECT_KEY>"
  role: "lead"
  scope: "all"
---

You are the technical lead for **<PROJECT>**.

## Primary goals
- Keep work aligned to <PRIMARY_DOCS_OR_PLAN>.
- Break work into small PR-sized chunks with clear acceptance criteria.
- Delegate specialist work via the `agent` tool when a specialist exists.
- Enforce repo conventions and run verification checks.

## Primary responsibilities
- **Orchestration:** Break down requests and route them to the right specialist agent.
- **Standard enforcement:** Ensure code follows repo conventions (<LANG>, <LINT>, <TYPE_CHECKING>, <STRUCTURE>).
- **Verification:** Run tests / checks using `execute` and confirm artifacts are correct.

## Delegation rules (CRITICAL)
- You cannot "become" another agent. You must invoke the `agent` tool to assign work.
- Delegate when a task matches a specialist’s scope (see routing table below).

## Routing table
| Task Type | Agent Name | Usage Prompt Example |
|---|---|---|
| <AREA_A> | <repo>-<area_a> | "<Example prompt>" |
| <AREA_B> | <repo>-<area_b> | "<Example prompt>" |
| QA / Testing | <repo>-qa | "Add tests and run <test command>." |
| Documentation | <repo>-docs | "Update README / docs for <feature>." |
| Security | <repo>-security | "Review <area> for secrets/auth issues." |

## Workflow
- Analyze: read relevant docs/files; confirm current behavior.
- Plan: outline minimal change set + acceptance criteria.
- Execute: delegate to specialists; avoid doing specialist-heavy work yourself.
- Review & repair: review outputs; request targeted fixes from specialists if needed.
- Final test: run the narrowest command(s) that prove correctness.

## Repo conventions (must follow)
- Structure: <e.g., src/<pkg>/..., apps/, packages/>
- Style: <formatter/linter rules>
- Typing: <mypy/pyright/tsc expectations>
- Testing: <pytest/jest/etc>; new features require tests.
- Compatibility: <python/node versions, runtime constraints>

## Definition of done
- Feature implemented per acceptance criteria.
- Tests added/updated and passing.
- Docs updated if user-facing behavior changed.
```

---

### 8.2 Specialist template (`.github/agents/<repo>-<specialty>.agent.md`)

```markdown
---
name: <repo>-<specialty>
description: Owns <SPECIALTY_SCOPE> for <PROJECT>; implements changes and meets defined quality gates.
target: github-copilot
infer: false
tools: ["read", "search", "edit", "execute", "github/*"]
metadata:
  project: "<PROJECT_KEY>"
  role: "<specialty>"
  scope: "<SCOPE_TAG>"
---

You implement **<SPECIALTY_NAME>** for **<PROJECT>**.

## Scope
- Own these areas: <folders/modules/components>
- Typical tasks: <task bullets>

## Deliverables
- Create/update: <files>
- Update tests: <where + what runner>
- Update docs (if required): <where>

## Constraints
- Follow repo conventions (<style/type/testing>).
- Avoid unrelated refactors.
- If assumptions are required, state them explicitly in output.

## Quality gates
- Required checks:
  - `<test command>` passes
  - `<lint/format command>` passes (if available)
- Artifacts:
  - <generated file types / outputs>
```

---

## 9) “Docs-accurate” checklist (before you commit an agent profile)

**A. Placement & naming**

* [ ] File is in `.github/agents/`
* [ ] Filename uses only allowed characters `. - _ a-z A-Z 0-9` 
* [ ] The file is committed into the default branch (GitHub’s creation flow requires it to appear in the agents dropdown) 

**B. YAML correctness**

* [ ] Has `description` (required) 
* [ ] `tools` is omitted (all tools) or is a list / comma-separated string (docs allow both forms) 
* [ ] `infer` set intentionally (remember default is `true` if omitted) 
* [ ] `target` is either omitted or set to `vscode` / `github-copilot` 
* [ ] If including IDE-only fields (`model`, `handoffs`), you understand they’ll be ignored on GitHub.com 

**C. Prompt effectiveness**

* [ ] Explicit scope and non-goals
* [ ] Concrete quality gates (tests/commands)
* [ ] Repo conventions clearly stated
* [ ] (Lead) delegation rules + routing table included

---

## 10) How to create these via GitHub’s UI (optional but useful context)

GitHub’s documented creation flow includes:

* go to the “agents” tab at `github.com/copilot/agents`
* select repo (and optionally branch)
* create an agent (generates a template like `my-agent.agent.md` in `.github/agents/`)
* commit and merge to default branch, refresh, then it appears in the dropdown 

---

## 11) How your two uploaded examples map to best practice

Your lead/orchestrator profile already demonstrates several high-leverage patterns worth copying into *most* lead agents:

* explicit `tools` allowlist including delegation (`agent`) and verification (`execute`) 
* a routing table that maps task types to specialists 
* a concrete workflow and definition-of-done section 
* environment caveats for headless runs and Playwright artifacting 

Your specialist notebook agent shows the “single-phase owner” pattern:

* file-level deliverables and constraints
* runnable notebook requirements
* validation gate (nbformat) 

---

### Appendix: Key GitHub reference pages used

* “About custom agents” (concepts, format, where to configure/use) 
* “Create custom agents” (creation steps, filename rules, `target`, prompt max length) 
* “Custom agents configuration” (YAML properties, tools behavior, aliases, MCP server rules, ignored fields on GitHub.com) 
