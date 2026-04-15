---
name: dev-workflow
description: Complete procedural workflow from ticket assignment to opening a PR, including clean baseline verification, implementation planning, scope management, testing, and commit standards.
---

# Dev Workflow — Shared Skill

This skill is loaded by Paperclip into the Backend Dev, Frontend Dev, and Mobile Dev agents. It provides the complete procedural workflow from receiving a ticket to opening a PR. Role-specific content (Identity, Stack, Target User) remains in each agent's CLAUDE.md.

---

## When This Skill Applies

When you are assigned a ticket by the PO, or re-assigned by the Lead Dev after review comments.

---

## Iron Laws

1. **ONE TICKET PER HEARTBEAT** — if multiple tickets are assigned, pick the highest-priority one. Defer others with a comment and set them back to `todo`. Never handle multiple tickets simultaneously.

2. **NO IMPLEMENTATION WITHOUT CLEAN BASELINE** — `pnpm test` must pass on the unmodified branch before writing any code. If the clean baseline fails, stop and report.

3. **NO CODE WITHOUT A PLAN** — read and understand the full implementation plan before writing any code. If the plan is unclear or incomplete, post a comment on the ticket asking the PO or Lead Dev for clarification. Do not guess.

4. **STAY IN SCOPE** — do not modify files outside the implementation plan. No speculative improvements, no opportunistic refactoring, no "while I'm here" changes. If you discover something that needs fixing outside the plan, create a comment on the ticket noting it — do not fix it.

5. **ONE BRANCH, ONE PR, ONE TICKET** — never combine multiple tickets into a single branch or PR. Each ticket gets its own worktree, branch, and PR.

6. **NO PR WITHOUT GREEN CHECKS** — `pnpm lint`, `pnpm typecheck`, `pnpm test`, and `pnpm build` must all pass before opening a PR. No exceptions. Do not open a PR "to see if CI passes."

7. **TEST FIRST** — write failing tests before implementation code. Verify the test fails for the right reason, then implement the minimum code to make it pass, then refactor. The test-driven-development external skill has the full red-green-refactor protocol.

8. **CO-AUTHOR EVERY COMMIT** — every commit message must end with `Co-Authored-By: Paperclip <noreply@paperclip.ing>`.

---

## Scoped-Wake Fast Path

If `PAPERCLIP_TASK_ID` is set in the environment, that is your ticket for this heartbeat — do not fetch your inbox.

Before anything else, sync your local repo:
```bash
git checkout main && git pull origin main
```

Then go straight to [Environment Verification](#environment-verification).

**One ticket per heartbeat**: if multiple tickets are assigned to you, pick the highest-priority one only. Post a comment on the others:
> ⏸ Deferring this ticket — already working on #\<ticket-id\>. Will pick this up once that PR is open.
Set their status back to `todo` and unassign yourself. Notify the PO.

Ensure your current worktree is fully committed and pushed before picking up a new ticket:
```bash
git status          # must be clean
git push origin HEAD
```

**Checkout**: the Paperclip system will checkout the issue before you start work. If checkout returns 409, the ticket is owned by another agent — stop immediately, do not retry, pick a different task.

---

## Environment Verification

Before creating a worktree, verify your environment:

```bash
node --version       # must match .nvmrc (Node.js 22 LTS)
pnpm --version       # must be 10.x
git status           # must be clean — no uncommitted changes
gh auth status       # must be authenticated
```

If Node.js version is wrong, run `nvm use`. If git status is dirty, commit or stash before proceeding.

---

## Understanding the Work

1. Read the ticket: stack layer, description, `depends_on`.
2. Read the implementation plan in full — every step, every file path, every edge case.
3. Read the files listed in the plan to understand existing patterns.
4. Verify all `depends_on` tickets are merged before starting.
5. **Discovery check**: before implementing, ask yourself:
   - Is the plan feasible given what I see in the code?
   - Are there any ambiguities or conflicts between the plan and the current codebase?
   - Can I describe the implementation in concrete steps?
   If anything is unclear, post a comment on the ticket asking the PO or Lead Dev. Do NOT guess.
6. Do not begin until you have a clear picture of what to change and why.

Once you have started implementation, set the issue status to **`in_progress`**.

---

## Git Worktree Setup

Use git worktrees to keep each ticket isolated in its own directory.

### 1. Locate or create the worktree directory

```bash
ls -d .worktrees 2>/dev/null && echo "exists" || echo "not found"
```

- If `.worktrees/` exists, use it.
- If not, create it — it is already listed in `.gitignore`.

Verify it is ignored:
```bash
git check-ignore -q .worktrees && echo "ignored — ok" || echo "NOT ignored — fix .gitignore first"
```

If not ignored, add `.worktrees/` to `.gitignore` and commit before continuing.

### 2. Create the worktree

```bash
BRANCH_NAME="<type>/<ticket-id>-<short-description>"

git worktree add .worktrees/$BRANCH_NAME -b $BRANCH_NAME
cd .worktrees/$BRANCH_NAME
```

Verify Node.js version in the worktree:
```bash
nvm use   # if .nvmrc exists
```

The new branch must contain NO commits ahead of main:
```bash
git log --oneline origin/main..HEAD   # must be empty
```

Verify all `depends_on` commits are visible in the history.

### 3. Install dependencies and verify clean baseline

```bash
pnpm install
pnpm test        # all tests must pass on the unmodified branch
```

If tests fail on the clean baseline, report the failures and ask whether to proceed before implementing anything. Do NOT start implementation on a broken baseline (Iron Law 2).

### 4. Report readiness

```
Worktree ready at .worktrees/<branch-name>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

---

## Implementation Discipline

### Follow existing patterns

Before writing new code, read 2-3 existing implementations of similar scope in the same package. Match the style, structure, naming, and conventions.

### Test-Driven Development

For every new function, endpoint, resolver, activity, or component:

1. **Red**: write a failing test first. Run it. Verify it fails for the expected reason.
2. **Green**: implement the minimum code to make the test pass.
3. **Refactor**: clean up while keeping tests green.

Do not write implementation code without a corresponding test (Iron Law 7).

### Codegen

If you add or modify GraphQL queries, mutations, or types:
```bash
pnpm codegen    # run at workspace root
```
Commit the generated files in the same commit as your changes. Verify the generated types are correct before running tests.

### Mid-implementation checkpoints

After completing each major step in the implementation plan:
```bash
pnpm test       # verify no regressions
```
If tests fail, fix before proceeding to the next step. Do not accumulate broken steps.

### Stay in scope

Follow the implementation plan step by step. If you need to modify a file not listed in the plan, stop and document why in a comment on the ticket. Do not make "while I'm here" improvements (Iron Law 4).

---

## Progress Visibility

For implementations spanning more than one session, post a progress comment on the issue before stopping:
> 🔄 In progress — completed: \<what is done\>. Next: \<what remains\>.

Keep the issue status as **`in_progress`** and the assignee unchanged until implementation is fully complete.

---

## If You Are Blocked

**Before starting** — if `depends_on` is not merged:
1. Do NOT start implementation.
2. Post a comment:
   > 🚫 Blocked — waiting for #\<ticket-id\> to be merged before starting.
3. Set status to **`blocked`**.
4. Stop — Paperclip will wake you when resolved (`issue_blockers_resolved`).

**Mid-implementation** — if a blocker arises:
1. Commit WIP: `git commit -m "wip: <description>\n\nCo-Authored-By: Paperclip <noreply@paperclip.ing>"`
2. Post a comment describing the blocker precisely.
3. Set status to **`blocked`**.
4. Assign to Lead Dev (technical blocker) or leave for PO (sequencing blocker).

---

## Verification Checklist

Before opening a PR, walk through this checklist. Every item must pass.

### Plan completion
- [ ] Every step in the implementation plan is completed
- [ ] No files outside the plan were modified (unless documented in a ticket comment)
- [ ] All edge cases from the plan are handled

### Test coverage
- [ ] Tests cover the implementation as specified in the ticket's Test Checklist
- [ ] Coverage thresholds met:
  - Statements: **80%**
  - Lines: **80%**
  - Functions: **80%**
  - Branches: **70%**
- [ ] Run `pnpm test:coverage` and verify

### Code quality
- [ ] `pnpm install` ran at worktree root — `pnpm-lock.yaml` is up to date and committed
- [ ] `pnpm lint` — 0 errors
- [ ] `pnpm typecheck` — 0 errors
- [ ] `pnpm test` — 0 failures
- [ ] `pnpm build` — succeeds
- [ ] No TODO/FIXME/HACK comments introduced
- [ ] Codegen ran if GraphQL was modified (`pnpm codegen`)

---

## Before Opening a PR

Run the full verification sequence from the worktree root:

```bash
pnpm install     # sync pnpm-lock.yaml — commit any changes before continuing
pnpm lint        # Biome linting — must pass with 0 errors
pnpm typecheck   # TypeScript type checking — must pass with 0 errors
pnpm test        # Vitest test suite — all tests must pass
pnpm build       # Production build — must succeed
```

If `pnpm install` modifies `pnpm-lock.yaml`, commit the updated file before pushing. A stale lockfile will break CI.

Do not open a PR if any of these fails (Iron Law 6).

---

## PR Format

```
Title: <type>(<scope>): <short description>

## Summary
- <What was implemented>
- <Key technical decisions made (if any)>

## Ticket
[<issue-ref>](https://paperclip.staging.crafto.fr/CRA/issues/<issue-ref>)
<!-- paperclip-issue-id: <issue-uuid> -->

## Test Coverage
- [ ] Unit tests: <what is covered>
- [ ] Integration tests: <what is covered>
- [ ] Component tests: <what is covered> (frontend/mobile only, omit if not applicable)

## Notes for Reviewer
<Any context the Lead Dev should know when reviewing>
```

`<scope>` is the primary package name: `internal-api`, `craftman-gateway`, `admin-gateway`, `craftman-mobile-app`, `craftman-interface`, `admin-interface`, or the activities/workflow package name.

---

## PR Labels

Use GitHub labels to reflect the current state of the PR. Apply with:
```bash
gh pr edit <number> --add-label "<label>" --remove-label "<label>"
```

| State | Label to apply |
|-------|---------------|
| PR opened, awaiting Lead Dev review | `pr-opened` |
| You are fixing review comments | `ai-is-fixing` |
| Fix committed and pushed, awaiting re-review | `ai-fix-done` |

---

## When You Are Done

Once your implementation is complete and the Verification Checklist passes:

1. Push your branch and open a PR using the PR Format above.
2. Add label `pr-opened` to the PR.
3. Set the Paperclip issue status to **`todo`**.
4. **Assign the Paperclip issue to the Lead Dev** on Paperclip — this triggers their heartbeat.
5. **Stop here** — do not merge, do not make further changes.
6. Wait for the Lead Dev's review — they will reassign the issue to you on Paperclip if changes are needed.

### Worktree cleanup

Once the PR is **merged**, remove the worktree:
```bash
git worktree remove .worktrees/<branch-name>
git branch -d <branch-name>
```

Do NOT clean up the worktree before the PR is merged — review comments may require further commits.

---

## Handling Review Comments

When the Lead Dev requests changes (you will be reassigned the issue on Paperclip), follow the **fix-review-comments** skill loaded by Paperclip. It covers: reading all comments before fixing, single-commit fixes, re-verification, round tracking, and handoff back to the Lead Dev.

---

## Red Flags — Stop & Escalate

If you encounter any of these, stop implementation and escalate:

- You are modifying files not listed in the implementation plan and cannot justify why
- Tests are failing and you cannot diagnose the cause after 15 minutes of investigation
- The implementation plan contradicts what you see in the codebase (e.g., a file doesn't exist, a pattern is different)
- You need to change a shared package (`@crafto/types`, `@crafto/sdk`) in a way not described in the plan
- A `depends_on` ticket was merged but its changes are not what the plan expected
- You are tempted to skip tests because "this is too simple to test"
- Review comments conflict with the original implementation plan
- You are on your second round of review fixes and still cannot satisfy the reviewer's comments
- A dependency is missing and cannot be installed

**How to escalate:**
1. Post a comment on the ticket describing the issue precisely.
2. Set the issue status to **`blocked`**.
3. Assign to the Lead Dev (technical issue) or PO (sequencing issue).
4. Stop — do not attempt to work around the issue.
