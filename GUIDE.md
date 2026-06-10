# Claude Certified Architect – Foundations (CCAF) Exam Guide

## Overview
- Validates practitioners can make informed tradeoffs implementing real-world Claude solutions
- Tests: Claude Code, Claude Agent SDK, Claude API, Model Context Protocol (MCP)
- Format: Multiple choice only; 1 correct + 3 distractors per question
- Scoring: Scaled 100–1,000; minimum passing score **720**
- No penalty for guessing; unanswered = incorrect
- Scenario-based: 4 of 6 scenarios presented at random

## Exam Domains & Weightings
| Domain | Weight |
|---|---|
| 1. Agentic Architecture & Orchestration | 27% |
| 2. Tool Design & MCP Integration | 18% |
| 3. Claude Code Configuration & Workflows | 20% |
| 4. Prompt Engineering & Structured Output | 20% |
| 5. Context Management & Reliability | 15% |

## Exam Scenarios (6 total, 4 selected at random)
1. **Customer Support Resolution Agent** – Agent SDK + MCP tools (get_customer, lookup_order, process_refund, escalate_to_human); 80%+ first-contact resolution
2. **Code Generation with Claude Code** – Slash commands, CLAUDE.md, plan mode vs direct execution
3. **Multi-Agent Research System** – Coordinator + specialized subagents (web search, document analysis, synthesis, report generation)
4. **Developer Productivity with Claude** – Built-in tools (Read, Write, Bash, Grep, Glob) + MCP integration
5. **Claude Code for CI/CD** – Automated code review, test generation, PR feedback; structured output
6. **Structured Data Extraction** – JSON schema validation, edge case handling, downstream integration

---

## Domain 1: Agentic Architecture & Orchestration (27%)

### 1.1 Agentic Loops
- Loop lifecycle: send request → inspect `stop_reason` (`"tool_use"` vs `"end_turn"`) → execute tool → append result → next iteration
- Tool results appended to conversation history so model reasons about next action
- **Anti-patterns**: parsing natural language to determine loop termination; arbitrary iteration caps as primary stop; checking assistant text content as completion indicator
- Model-driven decision-making vs pre-configured decision trees

### 1.2 Multi-Agent Orchestration (Coordinator–Subagent)
- Hub-and-spoke: coordinator manages all inter-subagent communication, error handling, routing
- Subagents have **isolated context** — do not inherit coordinator's conversation history automatically
- Coordinator responsibilities: task decomposition, delegation, result aggregation, subagent selection based on complexity
- Risk: overly narrow task decomposition → incomplete coverage of broad topics
- Route all subagent communication through coordinator for observability

### 1.3 Subagent Invocation, Context Passing & Spawning
- `Task` tool = mechanism for spawning subagents; `allowedTools` must include `"Task"` for coordinator to invoke subagents
- Subagent context must be **explicitly provided** in the prompt — no automatic inheritance
- `AgentDefinition`: descriptions, system prompts, tool restrictions per subagent type
- Fork-based session management for divergent approaches from shared baseline
- Spawn parallel subagents by emitting **multiple Task tool calls in a single coordinator response**
- Pass web search results + document analysis outputs directly in synthesis subagent's prompt
- Use structured data formats to separate content from metadata (source URLs, page numbers)

### 1.4 Multi-Step Workflows with Enforcement & Handoff
- **Programmatic enforcement** (hooks, prerequisite gates) vs prompt-based guidance
- When deterministic compliance required (e.g., identity verification before financial ops), prompt instructions alone have non-zero failure rate
- Implementing programmatic prerequisites: block `process_refund` until `get_customer` returns verified ID
- Structured handoff summaries for escalation: customer ID, root cause, refund amount, recommended action

### 1.5 Agent SDK Hooks
- `PostToolUse` hook: intercepts tool results for transformation before model processes them
- Hooks intercept outgoing tool calls to enforce compliance rules (e.g., block refunds above threshold)
- **Hooks = deterministic guarantees; prompts = probabilistic compliance**
- Use `PostToolUse` to normalize heterogeneous data formats (Unix timestamps, ISO 8601, numeric status codes)

### 1.6 Task Decomposition Strategies
- Fixed sequential pipelines (prompt chaining) vs dynamic adaptive decomposition based on intermediate findings
- Prompt chaining: analyze each file individually → cross-file integration pass
- Adaptive plans: generate subtasks based on discoveries at each step
- Use prompt chaining for predictable multi-aspect reviews; dynamic decomposition for open-ended investigations

### 1.7 Session State, Resumption & Forking
- `--resume <session-name>` to continue a prior named conversation
- `fork_session` for independent branches from shared analysis baseline
- Inform resumed session about specific file changes for targeted re-analysis rather than full re-exploration
- Starting fresh with injected structured summary is more reliable than resuming when prior tool results are stale

---

## Domain 2: Tool Design & MCP Integration (18%)

### 2.1 Effective Tool Interfaces
- Tool descriptions = primary mechanism LLMs use for tool selection; minimal descriptions → unreliable selection
- Include: input formats, example queries, edge cases, boundary explanations
- Ambiguous/overlapping descriptions cause misrouting (e.g., `analyze_content` vs `analyze_document` with near-identical descriptions)
- System prompt wording impacts tool selection: keyword-sensitive instructions create unintended associations
- Split generic tools into purpose-specific tools with defined input/output contracts

### 2.2 Structured Error Responses for MCP Tools
- `isError` flag pattern for communicating tool failures back to agent
- Error types: transient (timeouts), validation (invalid input), business (policy violations), permission
- Uniform "Operation failed" responses prevent appropriate recovery decisions
- Return structured metadata: `errorCategory`, `isRetryable` boolean, human-readable description
- `retriable: false` flags for business rule violations
- Distinguish access failures (retry decisions) from valid empty results (successful query, no matches)

### 2.3 Tool Distribution Across Agents
- Too many tools (e.g., 18 instead of 4–5) degrades tool selection reliability
- Agents with out-of-specialization tools tend to misuse them
- Scoped tool access: give agents only tools needed for their role
- `tool_choice` options: `"auto"`, `"any"`, forced `{"type": "tool", "name": "..."}}`
- `tool_choice: "any"` guarantees model calls a tool rather than returning conversational text
- Replace generic tools with constrained alternatives (e.g., `fetch_url` → `load_document` that validates document URLs)

### 2.4 MCP Server Integration
- Project-level: `.mcp.json` (shared team tooling) vs user-level: `~/.claude.json` (personal/experimental)
- Environment variable expansion in `.mcp.json` (e.g., `${GITHUB_TOKEN}`) for credential management without committing secrets
- Tools from all configured MCP servers discovered at connection time, available simultaneously
- MCP resources: expose content catalogs (issue summaries, documentation hierarchies, DB schemas) to reduce exploratory tool calls
- Prefer community MCP servers for standard integrations (Jira); custom servers for team-specific workflows

### 2.5 Built-in Tools (Read, Write, Edit, Bash, Grep, Glob)
- **Grep**: content search (file contents for patterns — function names, error messages, import statements)
- **Glob**: file path pattern matching (find files by name or extension, e.g., `**/*.test.tsx`)
- **Read/Write**: full file operations; **Edit**: targeted modifications using unique text matching
- When Edit fails (non-unique text matches) → use Read + Write as fallback
- Build codebase understanding incrementally: Grep for entry points → Read to follow imports, not all files upfront

---

## Domain 3: Claude Code Configuration & Workflows (20%)

### 3.1 CLAUDE.md Hierarchy & Organization
- Hierarchy: user-level (`~/.claude/CLAUDE.md`) → project-level (`.claude/CLAUDE.md` or root `CLAUDE.md`) → directory-level (subdirectory `CLAUDE.md`)
- User-level settings apply only to that user — not shared via version control
- `@import` syntax for referencing external files to keep CLAUDE.md modular
- `.claude/rules/` directory for topic-specific rule files (e.g., `testing.md`, `api-conventions.md`, `deployment.md`)
- Use `/memory` command to verify which memory files are loaded

### 3.2 Custom Slash Commands & Skills
- Project-scoped commands: `.claude/commands/` (shared via version control)
- User-scoped commands: `~/.claude/commands/` (personal)
- Skills in `.claude/skills/` with `SKILL.md` files; frontmatter: `context: fork`, `allowed-tools`, `argument-hint`
- `context: fork`: runs skill in isolated sub-agent context, prevents skill output from polluting main conversation
- Skills = on-demand invocation for task-specific workflows; CLAUDE.md = always-loaded universal standards
- Personal skill customization: `~/.claude/skills/` with different names to avoid affecting teammates

### 3.3 Path-Specific Rules for Conditional Convention Loading
- `.claude/rules/` files with YAML frontmatter `paths` fields containing glob patterns
- Path-scoped rules load only when editing matching files → reduces irrelevant context and token usage
- Glob patterns over directory-level CLAUDE.md for conventions spanning multiple directories (e.g., test files spread throughout codebase)

### 3.4 Plan Mode vs Direct Execution
- **Plan mode**: complex tasks with large-scale changes, multiple valid approaches, architectural decisions, multi-file modifications
- **Direct execution**: simple, well-scoped changes (single-file bug fix with clear stack trace, adding one validation check)
- Plan mode enables safe codebase exploration before committing to changes
- Explore subagent: isolates verbose discovery output and returns summaries to preserve main conversation context
- Combine: plan mode for investigation + direct execution for implementation

### 3.5 Iterative Refinement Techniques
- Concrete input/output examples = most effective way to communicate expected transformations
- Test-driven iteration: write test suites first, iterate by sharing test failures
- Interview pattern: have Claude ask questions to surface considerations before implementing
- Provide all interacting issues in single message; fix independent issues sequentially
- Use interview pattern for unfamiliar domains (cache invalidation strategies, failure modes)

### 3.6 CI/CD Pipeline Integration
- `-p` (or `--print`) flag: non-interactive mode for automated pipelines
- `--output-format json` + `--json-schema` CLI flags for structured output in CI contexts
- CLAUDE.md provides project context (testing standards, fixture conventions, review criteria) to CI-invoked Claude Code
- Session context isolation: Claude that generated code is less effective reviewing its own changes → use independent review instance
- Include prior review findings in context when re-running to avoid duplicate comments

---

## Domain 4: Prompt Engineering & Structured Output (20%)

### 4.1 Explicit Criteria to Reduce False Positives
- Explicit criteria over vague instructions (e.g., "flag comments only when claimed behavior contradicts actual code behavior" vs "check that comments are accurate")
- General instructions like "be conservative" fail to improve precision vs specific categorical criteria
- High false positive rates undermine developer trust even in accurate categories
- Define which issues to report (bugs, security) vs skip (minor style, local patterns)
- Temporarily disable high-FP categories while improving prompts

### 4.2 Few-Shot Prompting
- Few-shot examples = most effective technique for consistently formatted, actionable output when detailed instructions produce inconsistent results
- Demonstrate ambiguous-case handling (tool selection for ambiguous requests, branch-level test coverage gaps)
- Few-shot examples enable generalization to novel patterns (not just pre-specified cases)
- Effective for reducing hallucination in extraction tasks (informal measurements, varied document structures)
- 2–4 targeted examples for ambiguous scenarios showing reasoning for why one action was chosen over alternatives

### 4.3 Structured Output with Tool Use & JSON Schemas
- `tool_use` with JSON schemas = most reliable approach for guaranteed schema-compliant output; eliminates JSON syntax errors
- `tool_choice: "auto"` → model may return text instead of calling a tool
- `tool_choice: "any"` → model must call a tool but can choose which
- Forced tool selection → model must call specific named tool
- Strict JSON schemas eliminate syntax errors but **do not prevent semantic errors** (e.g., line items not summing to total)
- Design schema fields as optional/nullable when source documents may not contain the information (prevents fabrication)

### 4.4 Extraction Patterns for Unstructured Documents
- Role of schema design for handling heterogeneous formats
- Null/empty field handling to avoid fabrication

### 4.5 Batch API Usage
- Batch API: non-blocking, latency-tolerant workloads (overnight reports, weekly audits, nightly test generation)
- **Not** appropriate for blocking workflows (pre-merge checks) — use synchronous API
- Batch API does **not** support multi-turn tool calling within a single request
- `custom_id` fields for correlating batch request/response pairs
- Synchronous API for blocking pre-merge checks; batch for overnight/weekly analysis
- Calculate batch submission frequency from SLA constraints (e.g., 4-hour windows to guarantee 30-hour SLA with 24-hour batch processing)

### 4.6 Multi-Instance & Multi-Pass Review
- Self-review limitation: model retains reasoning context from generation → less likely to question its own decisions
- Independent review instances (without prior reasoning context) are more effective at catching subtle issues
- Multi-pass review: per-file local analysis passes + cross-file integration passes → avoids attention dilution
- Running verification passes where model self-reports confidence alongside each finding

---

## Domain 5: Context Management & Reliability (15%)

### 5.1 Conversation Context Management
- Progressive summarization risks: condensing numerical values, percentages, dates, customer-stated expectations into vague summaries
- **"Lost in the middle" effect**: models reliably process info at beginning and end of long inputs; may omit findings from middle sections
- Tool results accumulate in context consuming tokens disproportionately (e.g., 40+ fields per order lookup when only 5 are relevant)
- Pass complete conversation history in subsequent API requests to maintain coherence
- Extract transactional facts (amounts, dates, order numbers) into persistent "case facts" block included in each prompt
- Trim verbose tool outputs to only relevant fields before they accumulate
- Place key findings summaries at the **beginning** of aggregated inputs; use explicit section headers

### 5.2 Escalation & Ambiguity Resolution
- Escalation triggers: customer explicitly requests human, policy exceptions/gaps, inability to make meaningful progress
- Escalate immediately when customer explicitly demands it (don't attempt resolution first)
- Sentiment-based escalation and self-reported confidence scores = unreliable proxies for actual complexity
- Multiple customer matches → request additional identifiers, not heuristic selection
- When policy is ambiguous or silent → escalate (e.g., competitor price matching when policy only addresses own-site adjustments)

### 5.3 Error Propagation Across Multi-Agent Systems
- Structured error context (failure type, attempted query, partial results, alternative approaches) enables intelligent coordinator recovery
- Distinguish access failures (timeouts → retry decisions) from valid empty results (successful query, no matches)
- Generic error statuses hide context from coordinator
- **Anti-patterns**: silently suppressing errors (returning empty results as success); terminating entire workflow on single failure

### 5.4 Self-Evaluation & Confidence Scoring
- Self-evaluation (asking model to rate its own output) can improve reliability
- Calibrated confidence alongside findings enables routing decisions

---

## Key Configuration Reference
| Config | Scope | Purpose |
|---|---|---|
| `~/.claude/CLAUDE.md` | User-level | Personal preferences, not version-controlled |
| `.claude/CLAUDE.md` / root `CLAUDE.md` | Project-level | Team standards, version-controlled |
| Subdirectory `CLAUDE.md` | Directory-level | Directory-specific conventions |
| `.claude/rules/` | Path-scoped | Conditional loading via glob patterns |
| `.claude/commands/` | Project commands | Team slash commands |
| `~/.claude/commands/` | User commands | Personal slash commands |
| `.mcp.json` | Project MCP | Shared team MCP servers |
| `~/.claude.json` | User MCP | Personal/experimental MCP servers |

## Critical Distinctions (High-Exam-Frequency)
- **Hooks vs Prompts**: Hooks provide deterministic guarantees; prompts provide probabilistic compliance
- **`stop_reason: "tool_use"` vs `"end_turn"`**: Continue loop vs terminate
- **`tool_choice: "auto"` vs `"any"` vs forced**: May text / must call any tool / must call specific tool
- **Batch vs Synchronous API**: Latency-tolerant non-blocking vs blocking pre-merge checks
- **Plan mode vs Direct execution**: Architectural/multi-file vs single well-scoped change
- **Session resumption vs fresh start**: Prior context valid → resume; stale tool results → fresh + injected summary
- **Project-level vs User-level config**: Version-controlled team settings vs personal settings
- **Programmatic enforcement vs prompt guidance**: Financial/compliance workflows require programmatic
- **Subagent context isolation**: Subagents do NOT automatically inherit coordinator's conversation history
