---
name: code-review
description: Formalizes the procedure for reviewing PRs created by dev agents, requesting changes or approving, and managing the review cycle with escalation.
---

# Code Review — Shared Skill

> This skill formalizes the procedure for reviewing PRs created by dev agents, requesting changes or approving, and managing the review cycle with escalation.

**Loaded by:** Lead Dev agent.

---

## When This Skill Applies

This skill applies when a Paperclip issue is assigned to you with status `todo`. That assignment is the trigger — the dev agent reassigns the issue to you after opening a PR (first review) or after pushing a fix (re-review). You do not need to poll GitHub for this.

The associated PR will have label `pr-opened` (first review) or `ai-fix-done` (re-review). Use these labels for context only — they do not replace the Paperclip assignment as the source of truth for your work queue.

---

## Iron Laws

1. **NO REVIEW WITHOUT READING THE PLAN** — Always read the Paperclip ticket and implementation plan before reviewing code. Never review code in isolation. The plan is the contract; the code is the delivery.

2. **NO REVIEW WITHOUT READING THE DIFF** — Read the full PR diff, not just the files you expect to see. Check for unexpected changes, missing files, and scope creep. A partial review is worse than no review.

3. **NO COMMENT WITHOUT ACTION** — Every review comment must reference a specific line and suggest a concrete fix. "This doesn't look right" is not actionable. Style preferences are not review comments — Biome handles formatting.

4. **NO THIRD ROUND OF CHANGES** — If you have already requested changes twice and the fix is still inadequate, escalate to CTO. Do not enter an infinite review loop. Two rounds is the maximum.

5. **NO APPROVAL WITHOUT VERIFICATION** — Before approving, verify that all CI checks pass (tests, lint, typecheck, build) by checking CI status. Do not approve a PR with failing CI. Do not approve to "move things along" when you have concerns.

---

## The Process

### Phase 1: Identify the PR

1. Find the PR from the issue body (link in "Ticket" field, or search):
   ```bash
   gh pr list --repo crafto-app/applications --head <branch>
   ```
2. Apply label `ai-is-reviewing` to the PR, remove `pr-opened` / `ai-fix-done`.
3. Set the Paperclip issue status to `in_progress`.

**Gate:** PR identified, labels set, issue status updated.

### Phase 2: Understand Context

1. Read the Paperclip ticket in full — description, acceptance criteria, implementation plan.
2. Read the test checklist from the ticket.
3. Identify the scope: which files, which modules, which services are expected to change.

**Gate:** You can articulate in one sentence what the PR should do and what it should not touch.

### Phase 3: Review the Code

1. Read the full PR diff:
   ```bash
   gh pr diff <number> --repo crafto-app/applications
   ```
2. Check plan compliance — every planned step is implemented, nothing is missing.
3. Verify test coverage matches the test checklist from the ticket.
4. Check for security issues (injection, auth bypass, exposed secrets).
5. Check for performance issues (N+1 queries, missing indexes, unbounded loops).
6. Verify no files are modified outside the plan scope.
7. Verify GraphQL schema changes are backward-compatible (or explicitly flagged as breaking).
8. Verify error handling is appropriate (not swallowing errors, not exposing internals).
9. Verify code follows existing patterns in the codebase.

**Gate:** Review is complete, decision is made (approve, request changes, or escalate).

### Phase 4: Act on Decision

Before acting, count previous review rounds:

```bash
gh pr view <number> --repo crafto-app/applications --json reviews --jq '[.reviews[] | select(.state == "CHANGES_REQUESTED")] | length'
```

#### Path A — Changes Needed (round 1 or 2)

**Step 1 — Wake the developer (Paperclip, mandatory):**
Reassign the issue to the dev agent and set status to `todo`. This is what triggers their heartbeat — nothing else does:
```
PATCH /api/companies/{companyId}/issues/{issueId}
{ "assigneeAgentId": "<dev-agent-id>", "status": "todo" }
```
The dev agent ID is on the issue (`assigneeAgentId`) — do not guess it.

**Step 2 — Record the review on GitHub (traceability):**
Post a `CHANGES_REQUESTED` review — not just a comment. Every inline comment must reference a specific line and suggest a concrete fix:
```bash
gh pr review <number> --repo crafto-app/applications --request-changes --body "<summary of what must be fixed>"
gh pr edit <number> --repo crafto-app/applications --add-label "ai-requested-changes" --remove-label "ai-is-reviewing"
```

> The GitHub review and label are for traceability only. The Paperclip reassignment in Step 1 is the sole mechanism that wakes the dev agent.

#### Path B — Approved

1. Approve the PR on GitHub:
   ```bash
   gh pr review <number> --repo crafto-app/applications --approve
   ```
2. Apply label `ai-approved`, remove all other state labels.
3. Proceed to the **merge-pr** skill to merge the PR.

#### Path C — Escalation (round 3+)

1. Do **not** request changes a third time.
2. Apply label `human-review-required`, remove all other state labels.
3. Assign the Paperclip issue to CTO.
4. Set issue status to `blocked`.
5. Post a comment on the PR:
   > :warning: Review loop — 2 rounds of fixes requested, implementation still does not meet requirements. Core issue: \<description\>. Escalating to CTO. What the implementation does: \<summary\>. What I expect: \<expectation\>.

### Phase 5: Track

After acting, verify the final state is consistent:

- PR labels match the action taken.
- Paperclip issue status matches the action taken.
- Paperclip issue assignee matches the action taken.

**Gate:** Labels, status, and assignee are all consistent.

---

## Review Quality Criteria

When reviewing code, check each of the following:

- Implementation matches the plan (files, steps, edge cases)
- Test coverage matches the Test Checklist from the ticket
- No files modified outside the plan scope
- No security vulnerabilities (injection, auth bypass, exposed secrets)
- No performance regressions (N+1 queries, missing indexes, unbounded loops)
- GraphQL schema changes are backward-compatible (or explicitly flagged as breaking)
- Error handling is appropriate (not swallowing errors, not exposing internals)
- Code follows existing patterns in the codebase

---

## PR Label State Machine

| State                              | Label                    | Who sets it |
| ---------------------------------- | ------------------------ | ----------- |
| PR opened, awaiting review         | `pr-opened`              | Dev agent   |
| Review in progress                 | `ai-is-reviewing`        | Lead Dev    |
| Changes requested                  | `ai-requested-changes`   | Lead Dev    |
| Dev is fixing                      | `ai-is-fixing`           | Dev agent   |
| Fix committed, awaiting re-review  | `ai-fix-done`            | Dev agent   |
| Approved                           | `ai-approved`            | Lead Dev    |
| Escalated to human                 | `human-review-required`  | Lead Dev    |

---

## Anti-Patterns and Red Flags

- Reviewing code without reading the implementation plan first
- Requesting changes for style preferences (Biome handles formatting)
- Approving a PR without checking CI status
- Entering a third round of change requests instead of escalating
- Leaving vague comments without line references or concrete suggestions
- Approving to "move things along" when you have concerns
- Not verifying that the developer addressed ALL comments from the previous round

---

## Verification Checklist

Before approving, confirm every item:

- [ ] Implementation matches the plan — every planned step is present
- [ ] Test coverage matches the ticket's Test Checklist
- [ ] No files modified outside the plan scope (or justified in PR description)
- [ ] CI checks pass (lint, typecheck, test, build)
- [ ] No security concerns
- [ ] No performance concerns
- [ ] Review comments from previous rounds (if any) are all addressed
- [ ] PR description is clear and references the ticket
