# Implementation Plan V2: Local LLM CLI Agents (Best-Practice, Adopt-First)

## 1. Objective

Build a local-first CLI agent system on top of LM Studio for:

1. Writing and modifying code.
2. Reviewing code and pull requests.
3. Generating text and PDF documents.
4. Triage and filtering of mailbox content.

Primary requirement: **reuse proven open-source tooling** instead of rebuilding core capabilities.

## 2. Current baseline

- LM Studio is installed and working locally.
- Models available: **Qwen 3.5 27B** and **GPT-OSS 20B**.
- One model is loaded at a time in LM Studio.
- Hardware is sufficient for local inference.
- Repo is greenfield, so integration-first architecture is feasible.

## 3. Best-practice guardrails (must-have)

1. **Start simple:** use workflow-style orchestration first, then add autonomy only where needed.
2. **Human-in-the-loop by default:** require approval for edits, shell actions, and mailbox mutations.
3. **Use strict tool schemas:** clear arguments, bounded behavior, explicit errors.
4. **Use structured outputs:** JSON schema for review findings and triage decisions.
5. **Least privilege:** per-tool permissions (filesystem/network/shell/mail).
6. **Eval before automation:** quality gates before enabling auto-actions.
7. **Observability by default:** log prompts, tool calls, outcomes, and failures.
8. **Model routing by task:** do not force one model for all workloads.

## 4. Adopted components (reuse, not rebuild)

| Capability | Preferred adoption | Why |
|---|---|---|
| Local serving + compatibility | LM Studio + `lms` CLI | Native local serving, OpenAI-compatible and Anthropic-compatible endpoints |
| Terminal coding agent | `openai/codex` with LM Studio (`--oss`) | Strong CLI UX; direct LM Studio integration path |
| Terminal coding alt | `Aider-AI/aider` | Mature git-first coding workflow, local model support |
| Agent framework (advanced) | `OpenHands/OpenHands` | Mature coding-agent runtime, CLI/headless modes |
| PR review automation | `qodo-ai/pr-agent` | Purpose-built review, CLI and CI integrations |
| CI policy checks | `continuedev/continue` | Source-controlled AI checks in repo |
| Skills/plugins protocol | MCP (`modelcontextprotocol`) | Widely adopted standard for external tools |
| Build custom MCP tools | `PrefectHQ/fastmcp` | Fast, typed, production-oriented MCP development |
| Text/PDF output | `jgm/pandoc` (optional WeasyPrint) | Battle-tested document conversion pipeline |
| Workflow automation (optional) | `n8n-io/n8n` | Mature orchestration for mailbox and business workflows |

## 5. Target architecture

```text
User CLI
  -> Task Router (coding | review | docs/pdf | mail)
     -> Agent Runner (Codex/Aider/OpenHands or custom wrapper)
        -> LM Studio endpoint(s)
        -> Tool layer (Git, Filesystem, Test, Mail, Docs/PDF)
        -> MCP servers (skills/plugins)
  -> Safety Gate (approval + policy checks)
  -> Observability (logs, traces, evals)
```

Design policy: **thin orchestration layer**, heavy reuse of proven tools.

## 6. Phase-by-phase implementation

## Phase 0 — Environment hardening and benchmark baseline

1. Standardize LM Studio runtime settings and model identifiers.
2. Create baseline benchmark packs:
   - coding/edit tasks
   - PR review precision tasks
   - text/PDF generation tasks
   - email triage classification tasks
3. Define pass/fail criteria and reporting format.

**Exit criteria:** repeatable local benchmark run for both models.

## Phase 1 — Coding agent CLI (first production milestone)

1. Configure Codex with LM Studio (`codex --oss`) for primary coding flow.
2. Configure Aider with LM Studio as fallback/alternative path.
3. Add mandatory safety workflow:
   - branch-per-task
   - preview diff before apply
   - optional auto-lint/auto-test hooks
4. Document operator commands and known limitations per model.

**Exit criteria:** reliable daily coding loop from terminal on local models.

## Phase 2 — Code review agent

1. Deploy PR-Agent for PR review/describe/improve tasks.
2. Add Continue checks for policy-style PR gates (security, conventions, risk checks).
3. Normalize output schema:
   - severity
   - file/line
   - evidence
   - recommendation
4. Enforce “evidence-required findings” to reduce hallucinated comments.

**Exit criteria:** repeatable PR review quality with manageable false positives.

## Phase 3 — Text and PDF generation pipeline

1. Use local model for drafting/summarization/transformation.
2. Convert markdown -> PDF/DOCX/HTML using Pandoc.
3. Add template-based report generation for repeatable outputs.

**Exit criteria:** reliable text-to-PDF pipeline with standardized templates.

## Phase 4 — Email triage and filtering (read-only first)

1. Integrate mailbox via Gmail Workspace MCP or IMAP adapter.
2. Start with read-only operations:
   - classify
   - summarize
   - suggest rules/actions
3. Enable write actions only after confidence thresholds and user approval.
4. Keep kill-switch to immediately revert to read-only mode.

**Exit criteria:** accurate triage recommendations without unsafe side effects.

## Phase 5 — Skills/plugins expansion via MCP

1. Register needed MCP servers (git, files, mail, docs, search).
2. Build missing custom skills via FastMCP.
3. Enforce per-server permission manifests and execution timeouts.
4. Add compatibility tests for each enabled server.

**Exit criteria:** stable, permissioned plugin ecosystem.

## Phase 6 — Reliability, governance, and scale

1. Add regression eval suite for all scenarios.
2. Add observability dashboards for latency, error rate, success quality.
3. Add operational controls:
   - model fallback policies
   - action approval policies
   - emergency disable toggles
4. Publish runbooks and incident-response playbooks.

**Exit criteria:** production-ready operations with controlled risk.

## 7. Scenario-specific recommendations

### A) Writing code
- Use Codex first for interactive code workflows.
- Keep Aider available for git-structured, repo-map heavy tasks.
- Require tests/lint before finalizing edits.

### B) Reviewing code
- Use PR-Agent for PR-focused review automation.
- Use Continue checks for policy enforcement in CI.
- Keep deterministic review settings (low temperature).

### C) Creating text and PDF files
- Generate content in markdown.
- Convert with Pandoc to PDF/DOCX.
- Use templates rather than bespoke renderers.

### D) Reviewing and filtering mailbox
- Begin with read-only triage.
- Add action mode only after eval target is met.
- Keep human approval gate for archive/move/reply actions.

## 8. “Do not reinvent” decisions

1. Do not build a custom inference server while LM Studio meets needs.
2. Do not build a coding agent core from scratch before Codex/Aider/OpenHands pilots.
3. Do not build a PR reviewer from zero before PR-Agent/Continue adoption.
4. Do not build a custom PDF engine before Pandoc-based pipeline.
5. Do not invent a custom plugin protocol; use MCP.

## 9. Evaluation and acceptance criteria

- **Code edit quality:** task completion rate, regression rate, edit accuracy.
- **Review quality:** precision/recall of meaningful findings, false-positive rate.
- **Mail triage quality:** classification accuracy, unsafe action rate (must remain zero in read-only phase).
- **Ops quality:** latency SLOs, tool failure rate, recoverability.

Progression rule: no phase auto-advances without meeting its quality gates.

## 10. Immediate execution backlog (recommended order)

1. `env-baseline` - Lock LM Studio profiles and runtime defaults.
2. `codex-lmstudio` - Enable Codex local flow and validate coding loop.
3. `aider-lmstudio` - Enable fallback coding flow for comparison.
4. `review-stack` - Integrate PR-Agent + Continue checks.
5. `docs-pdf-stack` - Add markdown-to-PDF pipeline with Pandoc.
6. `mail-readonly` - Implement read-only triage.
7. `mcp-skillset` - Add MCP servers and permission policies.
8. `eval-suite` - Add end-to-end evaluations and release gates.

## 11. Key references used for this V2 plan

- LM Studio API/CLI and integrations:
  - https://lmstudio.ai/docs/app/api
  - https://lmstudio.ai/docs/developer/openai-compat
  - https://lmstudio.ai/docs/developer/rest
  - https://lmstudio.ai/docs/integrations/codex
  - https://lmstudio.ai/docs/integrations/claude-code
- Agent design best practices:
  - https://www.anthropic.com/engineering/building-effective-agents
- Reused OSS projects:
  - https://github.com/openai/codex
  - https://github.com/Aider-AI/aider
  - https://github.com/OpenHands/OpenHands
  - https://github.com/qodo-ai/pr-agent
  - https://github.com/continuedev/continue
  - https://github.com/modelcontextprotocol/servers
  - https://github.com/PrefectHQ/fastmcp
  - https://github.com/jgm/pandoc
  - https://github.com/n8n-io/n8n
