Confirm a gap item as officially decided. This command is for humans only — it marks a gap as `confirmed` with a recorded decision.

## Usage

```
/confirm-gap GAP-2026-001 "Decision text here"
```

Or run without arguments to be prompted interactively.

## What this does

1. Read `gap/gap-registry.md`
2. Find the gap item with the given ID
3. Fill in the **决策 / Decision** field with the provided decision text
4. Change `status` to `confirmed`
5. Add `confirmed_by` and `confirmed_date` fields to the YAML block
6. If the item is fully implemented, optionally change to `closed`

## Rules

- Do not invent or suggest the decision — only record what the human explicitly states
- The decision text should be a clear, actionable statement (e.g., "Use client-side canvas watermark for MVP. Server-side to be revisited in Sprint 3 if bypass becomes an issue.")
- After confirming, check if this decision unblocks other gap items and note them in the output

## YAML update format

Add these fields to the gap's YAML block:
```yaml
status: confirmed
decision_date: YYYY-MM-DD
confirmed_by: pm | be | both
```

And fill the **决策 / Decision** section with the provided text.

## Output

```
Gap GAP-XXXX confirmed
======================
Title: [title]
Decision: [decision text]
Confirmed by: [who]

Unblocked items:
- GAP-YYYY: [title] (was waiting on this decision)

Next steps:
- [any follow-up actions implied by the decision]
```

Then offer to commit and push the change.
