Generate a structured test plan based on the product requirements and API specification. Your goal is to produce mechanical completeness (all paths, all boundaries, all permissions) so the QA human can focus on exploratory and business-scenario testing.

## Step 1 — Gather context

1. Read `requirements/<sprint>/product-requirements.md` — extract Acceptance Criteria
2. Read `specs/api-specification.md` — extract all endpoints, request/response shapes, error codes
3. Read `adr/` — understand any architectural decisions that affect testing (e.g., async processing, caching, permission model)
4. Check `tests/` for existing test plans to understand the project's testing conventions

## Step 2 — Generate test cases by category

### Category 1: Acceptance Criteria Coverage
For each Acceptance Criterion in the requirements doc, generate at least one test case that verifies it.

### Category 2: API Contract Tests
For each endpoint in the spec:
- Happy path (valid input → expected output)
- Missing required fields
- Invalid field types / formats
- Invalid field values (negative numbers, future dates where past is expected, etc.)
- Correct HTTP status codes
- Correct error codes and messages

### Category 3: Boundary Analysis
For each input field:
- Empty string / null
- Maximum allowed length + 1
- Minimum value - 1
- Special characters (XSS probe: `<script>alert(1)</script>`)
- Unicode / emoji (if applicable)

### Category 4: Permission Matrix
For each endpoint, test every relevant role:
- Unauthenticated request → 401?
- Authenticated but wrong role → 403?
- Authenticated correct role → 200?
- Owner vs non-owner on resource-specific operations

### Category 5: Business Flow Tests
Full user journey tests for the main flows (derived from requirements):
- New user registration → first use → core action
- Error recovery flows (what happens after a failed step)
- State transitions (if entities have status machines)

## Step 3 — Output format

```markdown
# Test Plan — Sprint N / Feature Name
Generated: YYYY-MM-DD

## Coverage Summary
- Acceptance Criteria: N items → N test cases
- Endpoints: N endpoints → N test cases
- Boundary cases: N
- Permission matrix: N roles × N endpoints = N test cases
- Business flows: N

---

## Category 1: Acceptance Criteria

| TC-ID | Criterion | Steps | Expected | Priority |
|-------|-----------|-------|----------|----------|
| TC-001 | [AC text] | 1. ... 2. ... | [result] | P0 |

---

## Category 2: API Contract

| TC-ID | Endpoint | Scenario | Input | Expected Status | Expected Body |
|-------|----------|----------|-------|-----------------|---------------|

---

## Category 3: Boundary Cases

| TC-ID | Endpoint | Field | Boundary | Expected |
|-------|----------|-------|----------|----------|

---

## Category 4: Permission Matrix

| TC-ID | Endpoint | Role | Expected Status |
|-------|----------|------|-----------------|

---

## Category 5: Business Flows

[Numbered step-by-step flows]

---

## Out of Scope
[Explicitly list what is NOT covered in this test plan and why]

---

## QA Additions Needed
[Areas where Agent cannot generate test cases — needs QA's business knowledge]
- Exploratory testing areas: ...
- Edge cases that require domain knowledge: ...
```

## What NOT to do
- Do not mark test cases as Pass/Fail — that's the QA's job during execution
- Do not decide which test cases to skip — present all cases, let QA prioritize
