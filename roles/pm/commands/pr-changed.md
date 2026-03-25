A product requirement document has been updated. Your job is to diff the changes and automatically raise new gap items in the Gap Registry if the changes introduce new technical questions or conflicts with the current backend design.

## Usage

Run after PM has updated any requirement document (high-level-PR.md, product-design.md, sprint backlog, etc.):
```
/pr-changed
```

Or specify a file:
```
/pr-changed be-docs/specs/01-product-design.md
```

## What to do

1. **Find what changed**: Run `git diff HEAD~1 -- <file>` (or the specified file) to see recent changes. If no git diff is available, ask the user to paste the changed sections.

2. **Read current backend state**: Skim `be-docs/03-backend-architecture.md` and `be-docs/todo.md` to understand what's already implemented.

3. **Check existing gaps**: Read `gap/gap-registry.md` to avoid raising duplicate items.

4. **For each changed requirement, evaluate**:
   - Does this change conflict with an already-implemented backend feature?
   - Does this introduce a new feature with no backend coverage?
   - Does this change a previously confirmed decision (reopen a closed gap)?
   - Is there a technical risk or dependency the PM may not be aware of?

5. **Raise new gap items** for any issues found. Use the template at the bottom of `gap/gap-registry.md`. Set `status: raised`, `raised_by: be`.

6. If a previously confirmed gap's decision is contradicted by the new requirement, reopen it: add a note to the **决策 / Decision** field explaining the conflict, change `status` back to `raised`.

## Output

```
PR Change Analysis
==================
File changed: [filename]
Changes detected: [N additions, N deletions]

New gap items raised:
- GAP-2026-XXX: [title]
- ...

Existing gaps affected:
- GAP-2026-XXX: [title] — [how it's affected]

No action needed:
- [changes that are already covered or have no backend impact]
```

Then ask if gap-registry.md should be committed and pushed.
