---
name: merge-pr
description: Formalizes the procedure for merging an approved PR after verifying CI passes, and closing the implementation ticket.
---

# Merge PR — Shared Skill

> This skill formalizes the procedure for merging an approved PR after verifying CI passes, and closing the implementation ticket.

**Loaded by:** Lead Dev agent.

---

## When This Skill Applies

Immediately after you have approved a PR via the code-review skill. The PR has label `ai-approved` and your GitHub review status is "Approved".

---

## Iron Laws

1. **NO MERGE WITHOUT CI GREEN** — all CI checks must pass before merging. If any check fails, do not merge. Wait, investigate, or request the dev to fix.
2. **SQUASH MERGE ONLY** — always use squash merge to keep the main branch history clean. One commit per ticket on main.
3. **NO MERGE WITHOUT APPROVAL** — you must have an "Approved" review on the PR. Never merge without your own explicit approval.
4. **CLOSE THE IMPLEMENTATION TICKET** — after merging, set the implementation ticket to `done`. This unblocks dependent tickets (including the story's test ticket assigned to QA). Never leave a merged ticket open.

---

## The Process

### Phase 1 — Verify CI

After approving the PR, check that all CI checks pass:

```bash
gh pr checks <number> --repo crafto-app/applications
```

Wait for all checks to complete. If any check fails:

- **Flaky test or infra issue**: re-run the failed check.
  ```bash
  gh run rerun <run-id> --repo crafto-app/applications --failed
  ```
- **Real failure**: re-open your review, request the dev to fix the CI issue. Apply label `ai-requested-changes`, remove `ai-approved`. Assign the issue back to the dev agent. This counts as a review round.

**Gate:** all CI checks green.

### Phase 2 — Merge

Squash merge the PR:

```bash
gh pr merge <number> --repo crafto-app/applications --squash --delete-branch
```

The `--delete-branch` flag removes the remote branch after merge. The dev agent handles local worktree cleanup on their next heartbeat.

**Gate:** PR merged successfully, branch deleted.

### Phase 3 — Close Implementation Ticket

Set the implementation ticket status to **`done`**.
Do not reassign — you remain the last assignee as the agent who merged.

This triggers Paperclip to resolve `blockedByIssueIds` on dependent tickets. When the last implementation ticket of a story is merged, the story's test ticket (assigned to QA) is automatically unblocked and QA is woken up.

**Gate:** ticket status is `done`.

### Phase 4 — Confirm

Post a brief comment on the Paperclip issue:

> ✅ PR #\<number\> merged to main. Implementation ticket closed.

**Gate:** confirmation posted.

---

## Anti-Patterns & Red Flags

- Merging before all CI checks are green.
- Using regular merge instead of squash merge (pollutes main branch history).
- Forgetting to route the issue after merging (ticket gets stuck).
- Merging a PR you haven't approved (reviewing and merging must both happen).
- Leaving a merged implementation ticket open (blocks downstream tickets including QA).
- Force-merging to bypass failed checks.
- Merging while CI is still running ("it'll probably pass").

---

## Verification Checklist (before merging)

- [ ] PR has your "Approved" review.
- [ ] All CI checks are green (not pending, not failed).
- [ ] PR has label `ai-approved`.
- [ ] Merge method is squash.

## Post-Merge Checklist

- [ ] PR is merged and branch is deleted.
- [ ] Implementation ticket set to `done`.
- [ ] Confirmation comment posted on issue.
