Prepare context before starting a development task. Works for both backend and frontend — output adapts based on the project's tech stack in CLAUDE.md.

## Usage
```
/task-start <task-id>
```
Or run without arguments for interactive mode (will ask for task description).

## Step 1 — Find the task

1. Look for `docs/<sprint>/implementation-plan.md` or similar task list files
2. Find the task matching `<task-id>` — extract title, description, acceptance criteria, dependencies
3. If no task list exists, ask the user to paste the task description

## Step 2 — Load relevant context

For the identified task:
1. Find the related API endpoints in `specs/api-specification.md`
2. Find related ADR decisions in `adr/` that affect this task
3. Read the project `CLAUDE.md` for:
   - Tech stack (Java/Spring Boot? Node/TypeScript? React? Vue?)
   - Package structure conventions
   - Naming conventions
   - Any existing patterns to follow
4. Check if similar features already exist in the codebase — note the pattern to follow

## Step 3 — Generate implementation scaffold

Based on the tech stack detected from CLAUDE.md:

**If Java / Spring Boot (DDD structure)**:
```
Output:
- Domain layer: entity/value object signatures (no implementation)
- Application layer: service method signatures
- Adapter layer: controller method signatures + DTO class
- Infrastructure layer: repository interface

For each: class name, method signatures, key annotations
```

**If TypeScript / Node.js**:
```
Output:
- Type definitions (request/response interfaces)
- Service function signatures
- Controller/handler signatures
- Key middleware/validation hints
```

**If React / Vue (Frontend)**:
```
Output:
- Component file structure
- Props interface (TypeScript)
- API call abstraction (where it should live, not in the component)
- State shape
- Key lifecycle hooks / effects
```

## Step 4 — Output

```
Task Context: <task-id> — <title>
======================================

Acceptance Criteria:
- ...

Related API Endpoints:
- <Method> <Path> (from spec)

Related ADR Decisions:
- ADR-XXX: <decision summary> (relevant to this task because: ...)

Implementation Scaffold:
[generated scaffold based on tech stack]

⚠️ Notes & Gotchas:
- [specific things to watch out for based on the context]
- [any constraint from ADR or CLAUDE.md that affects implementation]

Dependencies:
- This task depends on: [other task IDs or external readiness]
- Other tasks depending on this: [if identifiable]

Suggested first step:
[one concrete next action to start]
```

## What NOT to do
- Do not write complete business logic — just the scaffold
- Do not make architectural decisions — surface them as questions if encountered
- Do not change existing code patterns without noting it explicitly
