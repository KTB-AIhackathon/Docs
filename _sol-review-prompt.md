# Review task (read-only)

You are an independent architecture reviewer. Do not rubber-stamp. Be terse. No compliments. Point at concrete risks.

IMPORTANT: Do NOT read or execute files under ~/.Codex/, ~/.agents/, ~/.claude/, .Codex/skills/, or agents/. Stay in this repo only.

Read:
- `Docs/README.md` (product spec)
- `Blocki-AI/DESIGN.md` (current v0.1 design by the FastAPI/LangGraph owner)

## Team opinion to evaluate (GitHub-first, calendar out, Notion later)

Architecture:
```
User → Spring (auth/users/API gateway) → FastAPI+LangGraph → GitHub MCP (per-user token) → save result; Notion later
```

Proposed 6 sub-agents:
- Orchestrator: parse request, pick agents
- Repo Analyzer: repo structure, commits, issues, PRs
- Progress Tracker: summarize + change detection
- README Writer
- Portfolio Builder
- Action Executor: real GitHub writes / PRs

Pipeline A (first): chat/request → orchestrator → analyze → progress → writer → preview → user approve → Action Executor

Pipeline B: cron per user → change detect → propose README/portfolio update → auto-apply OR notify

Principles: per-user token isolation; analyze → propose → approve → execute; auto-write is optional; MCP is a tool, business logic in LangGraph.

## New locked team decisions (must incorporate)

1. Portfolio AND resume are filled from `.md` templates (not free-form cards only).
2. Historical portfolio/resume versions are logged in Notion later. FastAPI may return markdown; Spring may store markdown in DB. Either is allowed. Notion is not in MVP path.
3. Calendar is out of this design.
4. Owner of FastAPI/LangGraph/GitHub MCP: one person. Spring team separate. Notion teammate later.

## What I need from you

1. Verdict on the 6-agent split vs DESIGN.md's F1 JobIngress / F2 GitHubCollect / F3 ArtifactBuilder. Which agents survive a split-gate (2+ change reasons AND reuse/test/cadence)? Which should merge?
2. Should there be an LLM Orchestrator, or should Spring send a typed `job_type`?
3. How should `.md` templates work (who owns files, placeholder contract, where versions live)?
4. Approval/execute boundary: what FastAPI returns vs what Spring stores vs when GitHub write is allowed.
5. Token isolation: who stores PAT, how FastAPI receives it, what must never happen.
6. Concrete IN/OUT for the surviving modules (field-level).
7. Implementation order for a hackathon MVP that still demos "agent manages my repo".
8. Top 5 failure modes (auth leak, MCP rate limit, hallucinated stack, accidental write, etc.).

Output format:
- KEEP / CUT / CHANGE (bullets)
- Recommended module list
- Field-level contracts
- Ordered MVP steps
- Risks
