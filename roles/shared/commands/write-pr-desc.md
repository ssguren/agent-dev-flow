Generate a Pull Request description based on the current git diff and the project context. The goal is to give reviewers enough context to do a meaningful review without asking the author "what does this PR do?"

## Step 1 — Gather context

1. Run `git diff main...HEAD --stat` to see which files changed
2. Run `git log main...HEAD --oneline` to see commit messages
3. Read the current branch name (often hints at the task ID)
4. Read `docs/<sprint>/implementation-plan.md` or similar — find the task this PR addresses
5. Read `specs/api-specification.md` if the PR touches API endpoints
6. Read the project `CLAUDE.md`

## Step 2 — Generate PR description

```markdown
## What does this PR do?

[2-3 sentences. What user-facing behavior changes? What backend behavior changes?
Link to the task / requirement it implements.]

Implements: [Task ID / Gap ID / Requirement reference]

---

## Why this approach?

[1-2 sentences explaining any non-obvious design choices. If this follows an ADR, reference it.]

Related ADR: [ADR-XXX or N/A]

---

## Changes

### Backend
- [File or module]: [what changed and why]
- ...

### Frontend (if applicable)
- ...

### Database / Config
- [ ] No schema changes
- [ ] Schema change: `<migration file>` (idempotent: yes/no)
- [ ] New config key: `<key>` (default: `<value>` / no default — must be configured)

---

## How to test

[Step-by-step instructions for the reviewer to verify the change locally or on staging]

1. ...
2. Expected result: ...

**Test accounts / data needed**: [or N/A]

---

## Checklist

- [ ] Tests added / updated
- [ ] API spec updated (if endpoint changed)
- [ ] No hardcoded secrets / IPs
- [ ] No debug output left in code
- [ ] No Chinese in code (variables / strings / logs)

---

## Notes for reviewer

[Optional: anything you want the reviewer to pay attention to, known limitations, or things you're not sure about]
```

## What NOT to do
- Do not fabricate the "Why this approach" section if the reason isn't clear from context — leave it as `[Author to fill]`
- Do not list every single line changed — summarize by logical area
- Do not write the checklist items as already checked — leave them for the author to verify
