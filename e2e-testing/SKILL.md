---
name: e2e-testing
description: Formalizes the procedure for writing and running E2E tests against merged code, validating acceptance criteria, reporting bugs, and routing the issue to its next step.
---

# E2E Testing — Shared Skill

> This skill formalizes the procedure for writing and running E2E tests against merged code, validating acceptance criteria, reporting bugs, and routing the issue to its next step.

**Loaded by:** QA Engineer agent.

---

## When This Skill Applies

When a test ticket (`[TEST] E2E — ...`) is unblocked by Paperclip because all its implementation dependencies are `done` (merged). The ticket was pre-assigned to you by the PO during decomposition — Paperclip wakes you automatically when all blockers resolve.

---

## Test Stack

The monorepo uses three E2E frameworks, one per target:

| Target | Framework | Test location | Run command |
|---|---|---|---|
| Backend API (GraphQL) | Supertest | `e2e/api/*.spec.ts` | `pnpm test:e2e:api` |
| Web interfaces (craftman-interface, admin-interface) | Playwright | `e2e/craftman-web/*.spec.ts`, `e2e/admin-web/*.spec.ts` | `pnpm test:e2e:web` |
| Mobile app (craftman-mobile-app) | Maestro | `e2e/mobile/flows/*.yaml` | `pnpm test:e2e:mobile` |

Run all E2E tests:
```bash
pnpm test:e2e
```

### Framework verification

Before writing any test, verify the framework exists for the target stack:

- **Backend**: check `supertest` in root `package.json` devDependencies, check `e2e/api/` directory.
- **Web**: check `@playwright/test` in root `package.json` devDependencies, check `e2e/craftman-web/` or `e2e/admin-web/` directory.
- **Mobile**: check `maestro` is available in the environment, check `e2e/mobile/flows/` directory.

If the framework for your target does not exist: escalate to Lead Dev (Iron Law 2). **STOP.**

---

## Iron Laws

1. **NO TEST WITHOUT READING ACCEPTANCE CRITERIA** — always read the PM story's acceptance criteria before writing any test. Tests validate the spec, not your assumptions.
2. **NO TEST WITHOUT VERIFYING THE FRAMEWORK** — before writing any E2E test, verify the E2E framework exists for the target stack (see Test Stack above). If the framework is missing, escalate to Lead Dev — do NOT set it up unilaterally.
3. **EVERY CRITERION GETS TWO TESTS** — for each acceptance criterion, write at least one nominal path test and one edge case test. No criterion goes untested.
4. **NO APPROVAL WITHOUT EVIDENCE** — never approve without running the tests and seeing them pass. "The code looks correct" is not QA approval.
5. **STRUCTURED BUG REPORTS ONLY** — when a bug is found, use the Bug Report Format. "It doesn't work" is not a bug report.
6. **USE THE RIGHT TOOL FOR THE RIGHT LAYER** — backend tickets use Supertest, web tickets use Playwright, mobile tickets use Maestro. Never use Playwright to test API endpoints or Supertest to test UI.

---

## The Process

### Phase 1: Receive & Understand

- Read the **test ticket**: acceptance criteria to validate, stack layers under test, test scenarios, depends_on.
- Read the **parent PM story**: full acceptance criteria, user story, NFRs, out of scope.
- Read the merged code changes for each implementation ticket listed in `depends_on` (check the merged commits on main).
- Identify which acceptance criteria need E2E coverage and which stack layers are involved.
- **Identify the target stacks** — this determines which frameworks to use (Supertest, Playwright, Maestro).

**Gate:** you can list every acceptance criterion, the test scenarios for each, and which framework each test uses. If you cannot, re-read the story. Do not proceed.

### Phase 2: Setup

1. Pull latest main:
   ```bash
   git checkout main && git pull origin main
   ```
2. Create a test branch:
   ```bash
   git checkout -b test/e2e-<ticket-id>
   ```
3. Verify the E2E framework exists for the target stack (see Test Stack section above).
   - If the framework is missing: escalate to Lead Dev (Iron Law 2). **STOP.**
4. Install dependencies:
   ```bash
   pnpm install
   ```
5. Set issue status to `in_progress`.

**Gate:** branch created, framework verified, dependencies installed. If any step failed, do not proceed.

### Phase 2b: Staging Deployment Verification

Before writing any test, verify that **all** implementation code is deployed on staging. The test ticket depends on multiple implementation tickets — each may have been merged at different times.

1. **Identify the merge commits** for each implementation ticket in `depends_on` (from their merged PRs):
   ```bash
   gh pr view <pr-number> --repo crafto-app/applications --json mergeCommit --jq '.mergeCommit.oid'
   ```
   Repeat for each implementation ticket. Record all merge commit SHAs.

2. **Check which image is deployed on staging** for each impacted service:
   ```bash
   kubectl get deployment <app-deployment> -n staging -o jsonpath='{.spec.template.spec.containers[0].image}'
   ```
   The image tag is a commit SHA from the CI build. The deployed image may contain commits **newer** than the ones you are testing — that is expected. What matters is that every merge commit is **included** in the deployed image.

3. **Verify each merge commit is included in the deployed image**:
   ```bash
   git merge-base --is-ancestor <merge-commit-sha> <deployed-image-tag> && echo "included" || echo "NOT included"
   ```
   Run this for **every** merge commit SHA. All must return `included`.
   - If any returns `NOT included`: the staging image was built before that merge. Wait for the CI pipeline to complete and ArgoCD to sync. Re-check after 5 minutes.

4. **Verify the pods are healthy**:
   ```bash
   kubectl get pods -n staging -l app=<app-name>
   ```
   All pods must be `Running` and `Ready`. If pods are in `CrashLoopBackOff` or `Pending`, do not start testing — escalate to DevOps.

5. **Verify the endpoints respond** (for backend/web):
   ```bash
   curl -sf https://<staging-url>/health || echo "UNHEALTHY"
   ```

**If staging is not ready after 15 minutes**: escalate to DevOps with a comment on the ticket describing what you observed (image tags, missing commits, pod status, endpoint response). Set status to `blocked`. **STOP.**

**Gate:** merge commit is included in the deployed staging image, pods are healthy, endpoint responds.

### Phase 3: Write & Run Tests

Identify the stack layer and use the corresponding framework.

#### Backend API (Supertest)

For tickets targeting `internal-api`, `craftman-gateway`, `admin-gateway`, or `webhook-api`:

- Write tests in `e2e/api/<ticket-id>.spec.ts`.
- Use Supertest to send GraphQL queries/mutations against the running API.
- Verify response status, data shape, and business logic.
- Test error cases: invalid input, missing auth, permission denied.

```bash
pnpm test:e2e:api
```

#### Web Interfaces (Playwright)

For tickets targeting `craftman-interface` or `admin-interface`:

- Write tests in `e2e/craftman-web/<ticket-id>.spec.ts` or `e2e/admin-web/<ticket-id>.spec.ts`.
- Use Playwright to navigate the UI, fill forms, click buttons, assert visible state.
- Test from the craftsman's perspective: clear errors, intuitive flows.

```bash
pnpm test:e2e:web
```

#### Mobile App (Maestro)

For tickets targeting `craftman-mobile-app`:

- Write flows in `e2e/mobile/flows/<ticket-id>.yaml`.
- Maestro flows are declarative YAML — describe taps, swipes, text input, assertions.
- Think from the craftsman's field perspective:
  - One hand occupied, large tap targets.
  - Voice-first interactions — verify voice triggers produce the correct result.
  - Poor connectivity — test offline/slow network behavior if relevant.

```bash
pnpm test:e2e:mobile
```

#### General guidance (all stacks)

For each acceptance criterion:
1. Write a nominal path test (happy path as the craftsman would use it).
2. Write an edge case test (invalid input, missing data, boundary condition).

**Gate:** all tests written and executed, results recorded. Do not evaluate until every test has run.

### Phase 4: Evaluate Results

- **All tests pass:** proceed to approval.
- **Tests fail:** determine if it is a bug in the implementation or a test issue.
  - If implementation bug: write a Bug Report for each failure.
  - If test issue: fix the test and re-run.

**Gate:** clear verdict — approved or blocked. No ambiguity allowed.

### Phase 5: Handoff

**If approved (all tests pass):**

1. Commit and push E2E tests on the test branch.
2. Open a PR for the E2E tests (title: `test(e2e): <ticket-id> — <short description>`).
3. Post on the Paperclip issue:
   > QA Approved — all acceptance criteria validated, E2E tests written. PR: #\<number\>
4. Set issue status to `todo`.
5. Assign to CTO for final validation.
6. **Stop — this run is complete.** Do not look for additional test tickets. End your response immediately so Paperclip can terminate this run cleanly and start the next queued run.

**If blocked (bugs found):**

1. For each bug, **create a new Paperclip bug ticket** using the Bug Report Format below. Assign it to the dev agent responsible for the impacted stack layer (Backend Dev / Frontend Dev / Mobile Dev) with status `todo` — this triggers their wakeup.
2. Set the test ticket's `blockedByIssueIds` to all newly created bug ticket IDs.
3. Set the test ticket status to `blocked`.
4. Do NOT re-run tests until all bug tickets are `done` (Paperclip will wake you automatically when blockers resolve).
5. **Stop — this run is complete.** Do not look for additional test tickets. End your response immediately so Paperclip can terminate this run cleanly and start the next queued run.

---

## Bug Report Format

Use this exact structure. One report per bug.

```
## Bug Report

### Summary
<One-line description of the issue>

### Acceptance Criterion Violated
<Which criterion from the PM story this breaks>

### Stack Layer
<backend / craftman-web / admin-web / mobile>

### Steps to Reproduce
1. <Step 1>
2. <Step 2>
3. ...

### Expected Result
<What should happen>

### Actual Result
<What actually happens>

### Severity
- [ ] Blocker — prevents the feature from working at all
- [ ] Major — significant deviation from spec, workaround exists
- [ ] Minor — cosmetic or low-impact deviation
```

---

## Progress Visibility

For testing spanning more than one session, post a progress update on the issue:

> In progress — completed: \<what is tested\>. Next: \<what remains\>.

---

## Anti-Patterns & Red Flags

- Writing tests without reading the acceptance criteria first.
- Approving without actually running the tests ("the code looks correct").
- Testing only the happy path and skipping edge cases.
- Creating an E2E framework without Lead Dev approval.
- Writing vague bug reports ("it doesn't work").
- Re-running tests after a failure without the dev fixing the bug tickets first.
- Reporting bugs as comments instead of creating separate bug tickets (bugs need their own lifecycle).
- Approving when tests are flaky ("it passed the second time").
- Using Playwright to test API endpoints (use Supertest).
- Using Supertest to test UI flows (use Playwright or Maestro).
- Writing Maestro flows for web interfaces (use Playwright).
- Starting tests without verifying the merge commit is deployed on staging.
- Assuming the staging image matches the merge commit exactly (it may contain newer commits — check ancestry, not equality).

---

## Verification Checklist

Before closing out the task, confirm every item:

- [ ] Staging deployment verified: merge commit included in deployed image, pods healthy, endpoint responding.
- [ ] Every acceptance criterion has at least one nominal test and one edge case test.
- [ ] Correct framework used for the stack layer (Supertest / Playwright / Maestro).
- [ ] All tests executed (not just written).
- [ ] Test results are clear: all pass OR bugs reported.
- [ ] Bug reports use the structured format (if any bugs).
- [ ] E2E test code committed and PR opened.
- [ ] Issue routed correctly: approved → CTO (`todo`) / blocked → bug tickets created, test ticket `blocked`.
