---
name: engineering-review
description: Review a software story, pull request, code diff, implementation plan, or technical walkthrough against a strict engineering-readiness standard. Assess product intent, ambiguity, end-to-end flow, functional correctness, testing, roles/permissions, backend/data filtering, per-function/class generality, maintainability, extensibility, coding-standard compliance (required SDK usage), and code explainability.
model: opus
---

# Engineering Review

Run a **principal engineer code review** on your working branch. Compare against `dev` (or `main` if dev doesn't exist), trace every change end-to-end, demand proof for every requirement, challenge design decisions, verify coding standards compliance, and confirm you can defend the approach end-to-end.

**Do not treat "code exists" as proof that the requirement is satisfied.** Every claim must be verified. The review is strict: each Pass must be earned, each Risk named, each standard violation flagged with a concrete pattern.

---

## Workflow

### 1. Establish the diff and product intent

**What you do:**
- Fetch the current branch name and target branch (dev, or main if dev doesn't exist).
- Pull the full diff: all files changed, all lines modified.
- Read the original story/ticket if it exists (look for `CONTEXT.md`, branch name, or Jira/Linear links).
- State the **intended user problem** and **expected behavior** in plain language. If the intent is unclear from the evidence, mark it and move forward anyway; the review will surface it.

**Done when:** You have the diff, the product intent is stated (even if speculative), and you know which files matter most.

---

### 2. Trace end-to-end: UI → state → API → backend → DB → response → UI

**What you do:**
For each major change, follow the full execution path:
- **UI layer** (frontend component, state change, user action)
- **State/hook** (React hooks, local state, cache invalidation)
- **API call** (tRPC query/mutation, query params, payload shape)
- **Backend handler** (route, service, business logic)
- **Database** (query, filter, where clause, what data is selected)
- **Response** (shape, filtering, transformation)
- **UI consumption** (does the response land in the right place? Does the UI react?)

Mark each link. If a link is **missing, indirect, or unverified**, flag it as a **risk**.

**Examples of broken links:**
- Filter param passed to API but never applied in the query.
- Permission check missing from the backend (service assumes caller is authorized).
- Response shape doesn't match the frontend's expectation.
- State change happens but UI doesn't re-render (cache miss, wrong query key, no invalidation).

**Done when:** Every major requirement has a visible chain from UI to DB and back, with no black boxes.

---

### 3. Identify ambiguities and product contradictions

**What you do:**
List questions that could materially change behavior, UX, architecture, or testing:
- Are there contradictions between the story and the implementation?
- Is the expected behavior defined? If not, is the implementation guessing?
- Are there hidden assumptions (e.g., "the user has only one role" or "the connection is always valid")?
- Do the acceptance criteria match what was built?

**Done when:** You have listed 2–5 ambiguities or marked product understanding as clear.

---

### 4. Challenge each major decision

**What you do:**
For each significant change, ask _why_ it was built that way. Specifically:

- **Data structure**: Why that shape? Could it be flatter, or is nesting justified? Is it forward-compatible?
- **Filtering logic**: AND or OR? Does it prevent over-fetching or under-fetching? Tested in combination?
- **Permission check**: Where is it? Frontend-only is not a check. Backend? Which layer? Can a caller bypass it?
- **State management**: Local state, hook, cache key, invalidation? Is cache invalidation correct (over-invalidate or under-invalidate)?
- **API design**: Is the param list growing? Should it be an object? Is versioning needed?
- **Scalability**: Does this break at 10k rows? 100k? What's the query cost?
- **Error handling**: What fails silently? What throws? Is the fallback user-friendly?

**Do not accept "it works" as an answer.** Demand a _reason_ for each choice.

**Done when:** You have asked at least one challenge question per major component, and you understand the reasoning (or you've surfaced a gap).

---

### 4. Verify standards and conventions

**What you do:**
- Check `CLAUDE.md` for project conventions (auth pattern, schema ownership, SDK rule, linting).
- Check the codebase for parallel implementations: how do other similar features do this?
- Assess: Is this approach aligned? If it diverges, is the divergence justified?
- For scalability: Are there known limits (API rate limits, query timeouts, pagination caps)? Does this design respect them?

**Done when:** You can state whether the implementation follows codebase conventions, and if not, why the deviation is acceptable.

---

### 5. Audit testing readiness

**What you do:**
Identify what tests _should_ exist to prove the implementation works:
- **Happy path**: Main user flow, realistic data.
- **Filters in isolation**: Each filter alone, against realistic volume.
- **Filters in combination**: AND/OR behavior verified. Role A + Role B + Filter X. Does it include or exclude as expected?
- **Edge cases**: Empty result set, single result, 1000 results. Permissions boundary.
- **Error paths**: API fails, backend throws, DB is slow. What does the UI show?
- **Data mutations**: After create/update, is the cache right? Does the list refresh?

List the tests that _don't exist_ and would expose bugs if they did.

**Done when:** You have a checklist of 5+ tests that would validate this implementation, and you know which ones are missing.

---

### 6. Review generality, maintainability, and extensibility per function/class

**What you do:**
For each new or materially changed function/class:
- **Generality**: Is this overly specific to today's fields or use case? Would adding a new field require changing the signature or callers?
- **Options/config abstraction**: Should this accept an options object instead of growing the param list? Is a config abstraction justified by real coupling reduction or near-term churn, or is it premature?
- **Reusability**: Should this move to a service, helper, or SDK? Or remain inline because it's one-off logic?
- **Duplication**: Is similar logic repeated in 2+ places? Where should it live?
- **Tight coupling**: Does this change require updating 10+ callers? Is there an abstraction that would help?
- **Clarity**: Would a future reader understand what this does? Are variable names, conditions, and return values clear?

**Do not over-engineer.** Prefer a specific design only when it reduces real coupling or prevents near-term churn. A general design that no one uses is waste.

**Done when:** You have reviewed each major function/class and stated whether it should remain specific, accept an options object, be extracted to a helper/service, or otherwise be generalized.

---

### 7. Enforce coding standards

**What you do:**
Read `CLAUDE.md` and `references/coding-standards.md`. Check the implementation against:
- **SDK rule**: Are internal endpoints called directly, or through the SDK/client abstraction? Direct calls violate the golden rule.
- **Schema ownership**: Do the right services own the migrations and types?
- **Auth pattern**: Kinde roles? Fail-fast zod env? Auth context checks at the boundary?
- **Hono router convention**: Per-router file? One endpoint per procedure?
- **React Compiler**: No manual `useCallback`, `useMemo`, `React.memo` (let the compiler do it).
- **Linting/formatting**: Does `bun run quality` pass?
- **Naming**: Do variables, functions, classes follow project patterns?

For each violation, show the concrete code and the preferred pattern.

**Done when:** You have checked against all relevant standards and either marked compliance or listed violations with evidence.

---

### 8. Verify explainability

**What you do:**
List the exact parts you must be able to explain in a real PR review:
- Why the `resolveJobFilter` logic works (especially AND/OR behavior in combinations).
- Where the permission check lives and why that's the right place.
- How the cache key is constructed and why it's stable across re-renders.
- What happens if the API returns an empty array (is it cached? retried? shown as "no results"?).
- Why the query WHERE clause includes `filter.orgIds` but not always (if that's the case).
- Any null/undefined coercion and why it's safe.
- How SDK/client abstractions are used and why direct endpoint calls are avoided.

**Done when:** You can point to 5–10 specific lines or patterns and explain them cold, under challenge.

---

### 9. Produce the engineering review report

**What you do:**
Write a report in this format. Use **Pass**, **Risk**, and **Missing evidence** strictly: Pass only where code or tests prove the claim; Risk where it might work but has a concern; Missing evidence where you can't verify. Do not assume unseen code or tests exist.

```
# Engineering Review

**Verdict:** Ready / Ready with risks / Not ready

## 1. Product Intent
State the user problem, goal, and expected behavior in plain language.
**Status:** Pass / Risk / Missing evidence

## 2. Ambiguities and Product Questions
List only questions that could materially change behavior, UX, architecture, or testing.

## 3. End-to-End Implementation Flow
Show the actual or expected chain: UI → state → query/hook → API → backend → business logic → database → response → UI
Mark any broken or unverified link.

## 4. Requirement Coverage
For each important requirement, show:
| Requirement | Evidence | Status: Pass / Risk / Missing evidence |

## 5. Testing Readiness
Check individual scenarios, combinations, realistic test data, roles/permissions, empty/no-result cases, invalid input, and failure paths. Identify tests that would expose hidden bugs.

## 6. Generality, Maintainability, and Extensibility
For each new or changed function/class: state whether it should remain specific, accept an options/config object, move into a reusable helper/service/SDK, or otherwise be generalized. Call out duplication, tight coupling, tightly coupled signatures, repeated logic, and likely near-term extension problems.

## 7. Coding Standards
Show each violation with concrete evidence and the preferred pattern. Enforce SDK usage; direct endpoint calls are violations.

## 8. Explainability Check
List 5–10 specific parts the developer must be able to explain during review, especially data flow, parameters, conditions, null/undefined handling, query construction, SDK usage, and failure behavior.

## 9. Top Fixes Before Review
Give the smallest prioritized set of concrete fixes or evidence to add before the PR review.

## 10. Questions the Reviewer May Ask
Provide 5–10 likely review questions based specifically on the submitted implementation.
```

**Done when:** The report is complete, honest, and actionable. Every Pass is earned. Every Risk is named. Every standard violation is concrete.

---

## Completion Criterion

The review is **done** when:
1. ✅ Product intent is stated (even if speculative) and ambiguities are listed.
2. ✅ The diff is traced end-to-end with no black boxes.
3. ✅ Every requirement has an evidence reference and a status (Pass / Risk / Missing evidence).
4. ✅ Every major decision has been challenged with a _why_, and you understand (or have surfaced a gap in) the reasoning.
5. ✅ Each new or changed function/class has been reviewed for generality, reusability, and coupling; state whether it should be generalized or remain specific.
6. ✅ Coding standards are verified; SDK violations are concrete and flagged.
7. ✅ Testing gaps are listed (not all tests need to exist, but you know what's missing and why it matters).
8. ✅ A full report is produced in the format above with honest Pass/Risk/Missing evidence verdicts.
9. ✅ You can explain 5–10 critical parts cold, under challenge.

If you cannot complete any of these, the review is **not ready**, and the report should say so and name the blocker.
