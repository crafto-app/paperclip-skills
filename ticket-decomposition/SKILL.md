---
name: ticket-decomposition
description: Formalizes the procedure for receiving a spec'd story, decomposing it into atomic implementation tickets, sequencing them by dependency order, and assigning to the correct dev agents.
---

# Ticket Decomposition — Shared Skill

This skill formalizes the procedure for receiving a spec'd story, decomposing it into atomic implementation tickets, sequencing them by dependency order, and assigning to the correct dev agents.

**Loaded by:** PO agent.

---

## When This Skill Applies

When a story issue is assigned to you by the Lead Dev with status `todo` and a technical specification comment is present on the issue.

---

## Iron Laws

### 1. NO TICKET WITHOUT DEDUP

**Before creating any ticket for a story, first call:**
```
GET /api/companies/{companyId}/issues?parentId={storyId}
```
If **any non-cancelled children** (status: todo, in_progress, blocked, in_review, done) already exist for this story, **stop immediately — do not create any ticket**. The story was already decomposed. Your job is to monitor and unblock the existing tickets, not create new ones.

Only if zero non-cancelled children exist, proceed with ticket creation. Then, for each individual ticket you are about to create, also call `GET /api/companies/{companyId}/issues?q={ticket-title-keywords}` to check for cross-story duplicates. If a match exists, reference it instead.

This gate runs without exception, every time, before touching any story.

### 2. ONE TICKET = ONE STACK LAYER

Never create a ticket that spans multiple unrelated layers. A story impacting internal-api AND gateway AND mobile must produce at least 3 tickets. The only exception: if two changes in the same layer are tightly coupled AND trivial (< 20 lines total), they may be combined in one ticket.

### 3. NO TICKET WITHOUT IMPLEMENTATION PLAN

Every ticket must include the full Implementation Plan section (Context, Architecture Rules, Files to Modify, Files to Create, Implementation Steps, Edge Cases, Test Checklist). "Implement as per spec" is not an implementation plan.

### 4. DEPENDENCY ORDER IS ABSOLUTE

Internal-api before gateway, activities before workflows, backend before frontend. Never assign a downstream ticket before its upstream dependency is merged. Dependency rules:

- Gateway tickets depend on their internal API tickets.
- Temporal Workflow tickets depend on their Activities tickets.
- Temporal Activities tickets that call internal-api depend on the relevant internal API tickets.
- Frontend/mobile tickets depend on the relevant gateway tickets (unless purely client-side).

### 5. WIP LIMIT = 1

Never assign more than 1 active ticket per dev agent at a time, regardless of dependency relationships. Assign the next ticket only once the current reaches `in_review` (PR opened, issue assigned to Lead Dev). Even independent tickets must be assigned sequentially — assigning multiple simultaneously causes the agent to handle several in one heartbeat, breaking the one-branch-one-PR rule.

### 6. NO HANDOFF WITHOUT VERIFICATION

The Verification Checklist must pass on the full set of tickets before any assignment. No exceptions.

---

## The Process

### Phase 1: Receive & Validate

Read the story and the Lead Dev's technical specification comment.

**Actions:**

1. Read the story issue: context, user story, acceptance criteria, NFRs, out of scope.
2. Read the Lead Dev's tech spec comment in full.
3. Verify the spec is complete: no TBD, no open design decisions, all layers covered.
4. If the spec is incomplete, reassign the issue to the Lead Dev with a comment listing exactly what is missing. Set status to `in_progress`. Stop.

#### Gate: Receive & Validate

- [ ] Story and spec fully read
- [ ] Spec is complete — no TBD, no open design decisions
- [ ] All acceptance criteria are covered by the spec

→ If spec is incomplete: reassign to Lead Dev with specific feedback. STOP.
→ If all pass: proceed to Phase 2.

---

### Phase 2: Analyze Stack Layers

From the spec's "Stack Layers Impacted" section, map each layer to the correct dev agent.

**Agent Assignment Table:**

| Stack Layer | Agent |
|---|---|
| `internal-api` | Backend Dev |
| `craftman-gateway` / `admin-gateway` | Backend Dev |
| `temporal-activities:<package>` | Backend Dev |
| `temporal-workflow:<name>` | Backend Dev |
| `craftman-mobile-app` | Mobile Dev |
| `craftman-interface` / `admin-interface` | Frontend Dev |

Never assign a mobile or frontend ticket to the Backend Dev, and vice versa.

#### Gate: Analyze Layers

- [ ] Every layer from the spec maps to exactly one agent
- [ ] No ambiguity in agent assignment

→ If any ambiguity: clarify with Lead Dev before proceeding.
→ If all pass: proceed to Phase 3.

---

### Phase 3: Create Tickets

For each layer (in dependency order), create an implementation ticket. Then create a single test ticket for the story.

#### 3a. Implementation Tickets

**Actions:**

1. Run the Dedup Gate for this ticket (Iron Law 1).
2. Write the ticket using the Ticket Format (see below).
3. Copy the relevant sections from the Lead Dev's spec verbatim into the ticket's Architecture Rules and Implementation Steps.
4. Verify the ticket targets exactly one stack layer (Iron Law 2).
5. Verify the Implementation Plan is complete (Iron Law 3).

#### 3b. Test Ticket (one per story)

Create a single E2E test ticket for the story. This ticket is assigned to the QA Engineer and covers all acceptance criteria from the PM story.

1. Run the Dedup Gate for this ticket.
2. Write the ticket using the Test Ticket Format (see below).
3. Set `depends_on` to **all implementation tickets** of this story — the QA cannot start until every implementation ticket is merged.

#### Ticket Atomicity Rules

Each ticket must:

- Target **one specific layer** of the stack.
- Be implementable by a dev agent in a focused session without requiring simultaneous changes in multiple unrelated layers.
- Have a clear, verifiable Definition of Done.

**Examples of correct decomposition:**

- ✅ `[TICKET] Add quote status field to internal API quote entity`
- ✅ `[TICKET] Expose quote status endpoint on API gateway`
- ✅ `[TICKET] Add SendQuoteNotification activity to notifications-activities package`
- ✅ `[TICKET] Update SendQuoteWorkflow to call SendQuoteNotification activity`
- ✅ `[TICKET] Display quote status badge on quote detail screen (mobile)`
- ❌ `[TICKET] Implement quote status feature` (too broad — spans multiple layers)

#### Gate: Create Tickets

- [ ] Every implementation ticket targets exactly one stack layer
- [ ] Every implementation ticket has a complete Implementation Plan
- [ ] One test ticket exists for the story, covering all acceptance criteria
- [ ] Test ticket depends on all implementation tickets
- [ ] Dedup Gate passed for every ticket

→ If any fails: fix before proceeding.
→ If all pass: proceed to Phase 4.

---

### Phase 4: Sequence & Set Dependencies

Set `depends_on` and `blockedByIssueIds` for each ticket.

**Dependency Order:**

1. **Internal API** (`internal-api`) — data models, business logic, internal endpoints
2. **Gateway** (`craftman-gateway` or `admin-gateway`) — depends on internal API
3. **Temporal Activities packages** — depends on internal API if they call it
4. **Temporal Workflows** — depends on the activities they call
5. **Frontend/Mobile** — depends on the relevant gateway (unless purely client-side)

**Actions:**

1. For each implementation ticket, identify its upstream dependencies.
2. Set `blockedByIssueIds` using the Paperclip API:
   ```
   POST /api/companies/{companyId}/issues
   { ..., "status": "blocked", "blockedByIssueIds": ["<blocking-ticket-id>"] }
   ```
3. For the test ticket, set `blockedByIssueIds` to **all implementation ticket IDs** of this story.
4. Set first-in-line implementation tickets (no dependencies) to status `todo`.

#### Gate: Sequence

- [ ] Every implementation ticket has explicit `depends_on` (or "none")
- [ ] No downstream ticket is unblocked when its upstream is not done
- [ ] Dependency order matches: internal-api → gateway → activities → workflows → frontend/mobile
- [ ] Test ticket is blocked by all implementation tickets

→ If any fails: fix dependencies.
→ If all pass: proceed to Phase 5.

---

### Phase 5: Verify

Run the full Verification Checklist (see below) on all tickets.

#### Gate: Verify

- [ ] All checklist items pass

→ If any fails: fix before proceeding.
→ If all pass: proceed to Phase 6.

---

### Phase 6: Assign

Assign tickets to dev agents respecting WIP limits and dependency order.

**Actions:**

1. For each unblocked implementation ticket (status `todo`), assign to the correct dev agent:
   ```
   PATCH /api/issues/{issueId}
   { "assigneeAgentId": "<dev-agent-id>", "status": "todo" }
   ```
2. Assign the test ticket to the **QA Engineer** (it stays `blocked` until all implementation tickets are `done` — Paperclip will wake the QA automatically):
   ```
   PATCH /api/issues/{issueId}
   { "assigneeAgentId": "<qa-agent-id>" }
   ```
3. Respect WIP = 1 for dev agents: if an agent already has an active ticket, do not assign another.
4. For blocked tickets: Paperclip will automatically wake the assignee (`PAPERCLIP_WAKE_REASON=issue_blockers_resolved`) when blockers resolve — no manual monitoring needed.
5. Set the story issue status to **`done`**.

#### Gate: Assign

- [ ] No dev agent has more than 1 active ticket
- [ ] Only unblocked implementation tickets are assigned
- [ ] Test ticket assigned to QA (status `blocked` until all impl tickets are done)
- [ ] Story issue status set to `done`

**Assignment complete — this run is complete.** Do not process additional stories from your inbox. Do not look for more work. End your response immediately so Paperclip can terminate this run cleanly and start the next queued run.

---

## Ticket Format

```
Title: [TICKET] <specific technical task>

## Parent Story
<Link or reference to the PM story>

## Stack Layer
<internal-api | craftman-gateway | admin-gateway | temporal-activities:<package-name> | temporal-workflow:<name> | craftman-mobile-app | craftman-interface | admin-interface>

## Description
<What needs to be implemented, in technical terms. Reference specific files, modules, or entities if known.>

## Depends On
- <ticket ID or "none">

## Implementation Plan

### Context
<Why this change is needed. Reference the parent story and the Lead Dev's technical spec.>

### Architecture Rules (relevant to this ticket)
<Patterns, conventions, and constraints from the codebase that apply.
Reference the Lead Dev's spec — copy the relevant sections verbatim.>

### Files to Modify
- `<path/to/file>` — <what to change and why>

### Files to Create
- `<path/to/new/file>` — <what it contains and why>

### Implementation Steps
1. <Step 1 — specific action>
2. <Step 2 — specific action>
...

### Edge Cases & Attention Points
- <Security concern, performance consideration, or tricky logic to handle>

### Test Checklist
- [ ] Unit test: <what to test>
- [ ] Integration test: <what to test>
- [ ] No test needed because: <reason> (only if truly not applicable)

## Definition of Done
- [ ] <specific verifiable criterion>
- [ ] Unit tests written and passing
- [ ] Integration tests written and passing (if applicable)
- [ ] PR created, pushed, and Lead Dev tagged for review
```

---

## Test Ticket Format

One test ticket per story. Assigned to QA, blocked by all implementation tickets.

```
Title: [TEST] E2E — <story short description>

## Parent Story
<Link or reference to the PM story>

## Depends On
- <impl ticket 1 ID>
- <impl ticket 2 ID>
- ...
(all implementation tickets of this story)

## Acceptance Criteria to Validate
<Copy the full acceptance criteria from the PM story verbatim>

## Stack Layers Under Test
<List which layers are impacted: backend API, craftman-web, admin-web, mobile>

## Test Scenarios
For each acceptance criterion:
- Nominal path: <brief description>
- Edge case: <brief description>

## Definition of Done
- [ ] E2E tests written and passing for all acceptance criteria
- [ ] At least one nominal + one edge case test per criterion
- [ ] Test code committed and PR opened
- [ ] QA verdict posted on ticket (approved or bugs reported)
```

---

## Prerequisites Check

Before creating implementation tickets, check whether the story requires external prerequisites (managed services, third-party API subscriptions, credentials). If so:

1. Create a prerequisite ticket first with label `prerequisite`.
2. Submit a Paperclip Board approval request if human action is needed.
3. Mark all dependent implementation tickets as blocked until the prerequisite is resolved.

---

## Progress Visibility

For decomposition tasks spanning more than one session, post a progress comment on the story issue before stopping:

> 🔄 In progress — completed: \<what is done\>. Next: \<what remains\>.

Keep the issue status as **`in_progress`** and the assignee unchanged until decomposition is fully complete.

---

## Anti-Patterns & Red Flags

Stop and reassess if you notice yourself:

- Creating a single ticket that spans multiple stack layers ("implement the feature end to end").
- Assigning a gateway ticket before the internal-api ticket is merged.
- Assigning multiple tickets to the same dev agent simultaneously.
- Writing "implement as per spec" instead of a full Implementation Plan.
- Skipping the Dedup Gate because "these are new tickets for a new story."
- Setting a downstream ticket to `todo` when its upstream dependency is still `in_progress`.
- Creating tickets without verifying the spec is complete (missing layers, TBD sections).
- Forgetting to set `blockedByIssueIds` on dependent tickets.
- Creating more tickets than necessary — if two changes in the same layer are trivial and tightly coupled, combine them.
- Assigning a mobile ticket to the Backend Dev or a backend ticket to the Mobile Dev.

---

## Verification Checklist

Before assigning any ticket, verify the full set:

- [ ] Every acceptance criterion from the story is covered by at least one implementation ticket
- [ ] Every implementation ticket targets exactly one stack layer
- [ ] Every implementation ticket has a complete Implementation Plan (context, architecture rules, files, steps, edge cases, tests)
- [ ] One test ticket exists for the story, covering all acceptance criteria
- [ ] Test ticket `depends_on` all implementation tickets
- [ ] Test ticket assigned to QA
- [ ] Dependencies are explicit: `depends_on` and `blockedByIssueIds` are set
- [ ] Dependency order matches: internal-api → gateway → activities → workflows → frontend/mobile
- [ ] No duplicate tickets (Dedup Gate passed for each)
- [ ] WIP limit respected: no dev agent has more than 1 active ticket
- [ ] Agent assignment matches the stack layer
- [ ] Story issue status set to `done`
