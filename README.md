# Agent Dev Flow

A role-based slash command framework for Claude Code that structures human-agent collaboration across the full SDLC. Agents generate, humans confirm.

---

## Background

Software development involves at least six roles passing work across seven handoff points — requirements → design → development → review → testing → deployment → postmortem. Each handoff leaks context, decisions, and responsibility.

Three root causes:

1. **Translation cost** — PM writes in product language, engineers in technical language. No structured mechanism bridges them.
2. **Context breaks** — Architecture decisions live in chat or verbal agreements. No single source of truth.
3. **No decision archive** — "Why was it designed this way?" has no reliable answer six months later.

This framework provides each role with a dedicated agent persona and purpose-built commands to address these gaps systematically.

---

## How It Works

Each role has a directory under `roles/` containing:

- `profile.md` — The agent's persona, responsibilities, behavior rules, and boundaries
- `commands/` — Slash command definitions (`.md` files) the agent can execute

The document flow is the context chain:

```
Requirements (PM)
    ↓
Gap Registry + ADR (Tech Lead)
    ↓
API Spec + Implementation Plan (Backend / Frontend)
    ↓
PR Description (Developer)
    ↓
Test Plan (QA)
    ↓
Deploy Checklist (DevOps)
    ↓
Postmortem (All)
```

---

## Roles & Commands

| Role | Commands |
|------|----------|
| **PM** | `pr-changed`, `pm-review-gaps` |
| **Tech Lead** | `split-tasks`, `gen-sprint-retro`, `update-from-retro` |
| **Backend** | `draft-api-spec`, `task-start`, `write-tests`, `check-conventions`, `be-review-gaps` |
| **Frontend** | `review-api-spec`, `task-start` |
| **QA** | `gen-test-plan`, `gen-boundary-cases`, `gen-permission-matrix`, `regression-scope`, `format-bug` |
| **DevOps** | `gen-deploy-checklist`, `gen-rollback-plan`, `smoke-test-checklist`, `check-config-diff` |
| **Shared** | `review-pr`, `draft-adr`, `write-pr-desc`, `confirm-gap`, `task-start` |

---

## Usage

### Option A — Copy commands into your project

Copy the relevant `roles/<role>/commands/*.md` files into your project's `.claude/commands/` directory. Claude Code will pick them up as slash commands automatically.

```bash
# Example: add DevOps commands to your project
cp agent-dev-flow/roles/devops/commands/*.md your-project/.claude/commands/
```

### Option B — Reference as a template

Open any `roles/<role>/profile.md` as the system prompt context when starting a role-specific Claude Code session. Then run the commands defined in that role's `commands/` directory.

### Option C — Use the Gap Registry pattern

For PM-backend alignment, adopt the Gap Registry workflow:

1. PM updates requirements → runs `/pr-changed` → new gaps are `raised`
2. BE agent runs `/be-review-gaps` → fills technical analysis → status: `be_reviewed`
3. PM agent runs `/pm-review-gaps` → fills product response → status: `pm_reviewed`
4. Human runs `/confirm-gap <ID> "decision"` → status: `confirmed`
5. After shipping → manually set to `closed`

---

## Core Rules

- **Agents generate, humans confirm.** Any decision affecting others requires human sign-off before taking effect.
- **No Chinese in code.** Comments, `@DisplayName`, and doc files are exempt.
- **Document flow is the context chain.** Every agent reads upstream outputs before producing downstream outputs.
- **When classification is uncertain, choose the conservative option.** (e.g., mark a diff as SUSPICIOUS rather than EXPECTED)

---

## Structure

```
agent-dev-flow/
├── README.md
├── CLAUDE.md               # Iron rules for all agents
├── background.md           # Full design rationale
├── workflow/
│   ├── 00-overview.md      # Full flow diagram + role×command matrix
│   ├── 01-requirements.md
│   ├── 02-design.md
│   ├── 03-development.md
│   ├── 04-review.md
│   ├── 05-testing.md
│   ├── 06-deployment.md
│   └── 07-postmortem.md
└── roles/
    ├── shared/commands/    # Cross-role commands
    ├── pm/
    ├── tech-lead/
    ├── backend/
    ├── frontend/
    ├── qa/
    └── devops/
```
