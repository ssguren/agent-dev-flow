Draft an Architecture Decision Record (ADR) for the given topic. Your job is to research the context, list options with trade-offs, and produce a complete ADR — except for the Decision section, which only a human can fill.

## Usage
```
/draft-adr <topic>
```
Example: `/draft-adr payment-order-model`

## Step 1 — Gather context

1. Read existing ADRs in `adr/` or `docs/adr/` to understand the numbering sequence and established decisions
2. Read `gap/gap-registry.md` — is there a confirmed Gap that triggered this ADR?
3. Read relevant sections of `specs/api-specification.md` and `CLAUDE.md`
4. If the topic overlaps with an existing ADR, note it explicitly

## Step 2 — Research options

For the given topic, identify 2-4 concrete implementation options. For each option:
- What does it do?
- What are the pros?
- What are the cons?
- What does it require (dependencies, effort, approvals)?
- What does it preclude (what can't you do if you choose this)?

## Step 3 — Output the ADR

Use this exact structure:

```markdown
# ADR-<NNN>: <Title>

**Status**: Draft
**Date**: YYYY-MM-DD
**Raised by**: [role]
**Triggered by**: [Gap ID or requirement reference]

---

## Context

[2-3 paragraphs: what is the problem, why does a decision need to be made now,
what are the constraints (technical, business, timeline)]

---

## Options

### Option A: <Name>
**Description**: ...
**Pros**: ...
**Cons**: ...
**Effort**: [S/M/L]
**Requires**: ...

### Option B: <Name>
[same structure]

### Option C: <Name> (if applicable)
[same structure]

---

## Analysis

[A comparison table or brief narrative comparing the options on the dimensions that matter most for this decision]

| Criterion | Option A | Option B | Option C |
|-----------|----------|----------|----------|
| ... | | | |

---

## Decision

> **⚠️ This section must be filled by a human. Do not generate content here.**
>
> Chosen option: ___
> Reason: ___
> Decided by: ___
> Date: ___

---

## Consequences

[To be filled after decision is made]

**If Option A**:
- ...

**If Option B**:
- ...

---

## References
- [Link to Gap Registry item if applicable]
- [Link to relevant requirements doc]
- [External references]
```

## Step 4 — Suggest next steps

After outputting the ADR draft, print:
```
ADR Draft Complete
==================
File: adr/ADR-<NNN>-<topic>.md
Triggered by: [Gap ID or N/A]

Next steps:
1. Tech Lead reviews and fills the Decision section
2. Run /confirm-gap <gap-id> if this resolves an open Gap
3. If Option [X] is chosen, the following tasks are unblocked: [list]
```
