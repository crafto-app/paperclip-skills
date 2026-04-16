---
name: fix-review-comments
description: Formalizes the procedure for receiving review feedback from the Lead Dev, fixing all comments in a single commit, re-verifying, and handing back for re-review.
---

# Fix Review Comments — Shared Skill

This skill formalizes the procedure for receiving review feedback from the Lead Dev, fixing all comments in a single commit, re-verifying, and handing back for re-review.

**Loaded by:** Backend Dev, Frontend Dev, Mobile Dev agents.

---

## When This Skill Applies

This skill activates when you are reassigned a Paperclip issue by the Lead Dev after a code review with status `todo`, and the PR has the label `ai-requested-changes`.

---

## Iron Laws

1. **NO FIX WITHOUT READING ALL COMMENTS FIRST** — Read every review comment before making a single change. Understand the full picture. Fixing comments one by one leads to contradictory fixes.
2. **NO PARTIAL FIXES — ONE COMMIT FOR ALL** — Address all review comments in a single commit. Do not push partial fixes. The Lead Dev reviews the fix as a whole, not incrementally.
3. **NO SILENCE — RESPOND TO EVERY COMMENT** — Reply to each GitHub PR comment explaining what you changed. If you disagree with a comment, explain why clearly and respectfully. Never silently ignore a comment.
4. **NO FIX WITHOUT RE-VERIFICATION** — After fixing, re-run the full Verification Checklist (lint, typecheck, test, build, coverage). Do not hand back a fix that breaks something else.
5. **NO FIX WITHOUT KNOWING YOUR ROUND** — Track which review round this is. If this is your second fix and it still does not satisfy the Lead Dev, the issue escalates to the CTO. There is no third round.

---

## The Process

### Phase 1: Receive & Prepare

1. Switch to the correct worktree: `cd .worktrees/<branch-name>`
2. Pull latest from the branch (in case of any CI-triggered changes).
3. Apply label `ai-is-fixing` to the PR.
4. Read ALL review comments — do not start fixing yet.

**Gate:** All comments read, full picture understood.

### Phase 2: Analyze Comments

1. Categorize each comment: bug fix, missing test, style issue, architecture concern, scope question.
2. If any comment conflicts with the original implementation plan, flag it — post a question on the PR before fixing. Wait for clarification.
3. If a comment requires changes outside the plan scope, note it in your reply and ask the Lead Dev whether to proceed.

**Gate:** All comments categorized, no unresolved conflicts.

### Phase 3: Fix

1. Address all comments in a single commit.
2. Follow the implementation plan and existing patterns.
3. If the Lead Dev suggested a specific fix, follow it unless you have a strong technical reason not to (explain in your reply).
4. Include `Co-Authored-By: Paperclip <noreply@paperclip.ing>` in the commit message.

**Gate:** All comments addressed in one commit.

### Phase 4: Verify

Re-run the full Verification Checklist:

- `pnpm lint` — 0 errors
- `pnpm typecheck` — 0 errors
- `pnpm test` — 0 failures
- `pnpm build` — succeeds
- Coverage thresholds: 80% statements/lines/functions, 70% branches

**Gate:** All checks pass.

### Phase 5: Hand Back — END OF RUN

1. Reply to each GitHub PR comment explaining what you changed (or why you disagree).
2. Push the fix commit.
3. Apply label `ai-fix-done`, remove `ai-is-fixing`.
4. Assign Paperclip issue back to Lead Dev — triggers their heartbeat.
5. Set issue status to `todo`.
6. **Stop — this run is complete.** Do not check for additional tickets. Do not process your inbox. Do not start another task. End your response immediately so Paperclip can terminate this run cleanly and start the next queued run.

**Gate:** PR handed back, labels correct, issue reassigned.

---

## Round Tracking

Track how many times you have been asked to fix this PR. Check:

```bash
gh pr view <number> --repo crafto-app/applications --json reviews --jq '[.reviews[] | select(.state == "CHANGES_REQUESTED")] | length'
```

- **Round 1:** Normal fix cycle. Follow the process above.
- **Round 2:** This is your last chance. If the Lead Dev still requests changes after this, the issue escalates to the CTO automatically. Be extra thorough.
- **Round 3+:** Should never happen. If you are reassigned a third time, post a comment noting the escalation should have happened and assign to CTO.

---

## Anti-Patterns & Red Flags

- Fixing comments one by one and pushing multiple commits (violates Iron Law 2).
- Ignoring a review comment because you disagree (violates Iron Law 3 — reply and explain instead).
- Pushing a fix without re-running verification (violates Iron Law 4).
- Starting to fix before reading all comments (violates Iron Law 1).
- Making additional changes beyond what was requested ("while I'm fixing, let me also...").
- Silently reverting your own previous work instead of addressing the reviewer's concern.
- Pushing a fix that introduces a new test failure.

---

## Verification Checklist

Before handing back, confirm every item:

- [ ] Every review comment is addressed — no comment left without a reply
- [ ] All fixes are in a single commit with proper `Co-Authored-By: Paperclip <noreply@paperclip.ing>`
- [ ] `pnpm lint` — 0 errors
- [ ] `pnpm typecheck` — 0 errors
- [ ] `pnpm test` — 0 failures
- [ ] `pnpm build` — succeeds
- [ ] Coverage thresholds met (80% statements/lines/functions, 70% branches)
- [ ] Labels correct: `ai-fix-done` applied, `ai-is-fixing` removed
- [ ] Paperclip issue assigned to Lead Dev with status `todo`
