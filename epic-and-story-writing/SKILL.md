---
name: epic-and-story-writing
description: Formalizes the procedure for receiving Board requests, writing Epics, decomposing them into atomic user Stories, and handing off to the Lead Dev for technical specification.
---

# Epic & Story Writing — Shared Skill

This skill formalizes the procedure for receiving Board requests, writing Epics, decomposing them into atomic user Stories, and handing off to the Lead Dev for technical specification.

**Loaded by:** PM agent.

---

## When This Skill Applies

When you receive a Board request (feature, enhancement, strategic goal), or need to create stories from existing documentation.

---

## Iron Laws

1. **NO STORY WITHOUT DEDUP** — before creating any story issue, call `GET /api/companies/{companyId}/issues?q={story-title-keywords}` and review results. If a non-cancelled issue with a similar title already exists (any status), do NOT create a duplicate — reference the existing issue instead. If the feature is already implemented (merged PR exists), skip entirely. This gate runs for every story regardless of origin (Board request, docs/stories/, subtask breakdown).

2. **ONE STORY = ONE ATOMIC USER NEED** — never bundle multiple distinct features or behaviours into a single story. One story describes a single coherent user action or outcome. If a story has a cross-stack technical impact, that is acceptable — but it must describe one user need.

3. **NO STORY WITH UNRESOLVED PREREQUISITES** — if a story requires external prerequisites that a human must handle (managed service subscription, API keys, legal/compliance validation, strategic decisions), do NOT write the story yet. Write a prerequisite issue first and submit a Board approval request. Only proceed with the story once the prerequisite is resolved.

4. **NO VAGUE ACCEPTANCE CRITERIA** — every acceptance criterion must be mechanically testable by a QA agent. "Works well", "good UX", "performant" are not criteria. Each criterion is a verifiable yes/no check.

5. **NO HANDOFF WITHOUT SIZE CHECKLIST** — the Story Size Checklist must pass for every story before setting status to In Review. No exceptions.

---

## The Process

### Phase 1: Clarify

Receive the Board request. Read it carefully. Identify any ambiguity, missing context, or implicit assumptions.

**Actions:**
- If the request is ambiguous, post a clarifying comment on the Board issue and WAIT for the answer before proceeding. Do not guess.
- If the request references external services, subscriptions, or third-party integrations, flag these immediately as potential prerequisites.
- Identify which V1 success metric(s) this request maps to:
  - App deployed to production (iOS + Android)
  - Full quote creation via voice in < 2 minutes
  - 0 critical incidents in 30 rolling days
  - Functional CI/CD with automated tests

#### Gate: Clarify

- [ ] Request is unambiguous — all questions answered
- [ ] External prerequisites identified (if any)
- [ ] V1 metric mapping identified

> If any question is unanswered: wait. Do NOT proceed.
> If all pass: proceed to Phase 2.

### Phase 2: Write Epic

Write the Epic with business context grounded in craftsmen's real-world needs.

**Actions:**
- Frame the Epic from the craftsman's perspective: field use, offline resilience, voice-first UX, non-technical users.
- Reference the company V1 goal in the Epic description.
- Identify all user-facing behaviours covered by this Epic.

#### Gate: Write Epic

- [ ] Epic links to at least one V1 success metric
- [ ] Context is grounded in craftsmen's real-world workflow (not abstract)
- [ ] All user-facing behaviours are listed

> If any fails: revise before proceeding.
> If all pass: proceed to Phase 3.

### Phase 3: Decompose into Stories

Break the Epic into user Stories. Deliver in small batches: 2-4 stories max per batch.

**Actions:**
1. List all distinct user-facing behaviours from the Epic.
2. For each behaviour, draft one story using the Story Format (see below).
3. For each story, run the Dedup Gate (Iron Law 1) before creating the issue.
4. Apply Story Prioritisation Rules to sequence the batch.
5. List stories in priority order in the batch comment.

#### Gate: Decompose

- [ ] Every story describes exactly one atomic user need (Iron Law 2)
- [ ] Dedup Gate passed for every story (Iron Law 1)
- [ ] Prerequisites resolved or tracked as prerequisite issues (Iron Law 3)
- [ ] Batch size is 2-4 stories

> If any fails: fix before proceeding.
> If all pass: proceed to Phase 4.

### Phase 4: Verify

Run the Story Size Checklist on every story in the batch.

#### Story Size Checklist

Before finalising a story, verify:

- [ ] A single user action or outcome is described (not two combined)
- [ ] The Lead Dev could write the technical spec without needing to split the story further
- [ ] Out of Scope explicitly excludes anything that belongs in a separate story
- [ ] The story does not assume unresolved prerequisites
- [ ] Non-functional constraints are documented or explicitly absent

#### Gate: Verify

- [ ] Every story passes all 5 checklist items (Iron Law 5)
- [ ] Every acceptance criterion is testable (Iron Law 4)

> If any fails: revise the story before proceeding.
> If all pass: proceed to Phase 5.

### Phase 5: Handoff

**Actions:**
1. Set each story issue status to **In Review**.
2. **Assign each story issue to the Lead Dev** for technical specification.
3. Post a batch comment listing all stories in priority order with a one-line summary for each.
4. Do NOT assign directly to the PO — the Lead Dev specs first.

#### Gate: Handoff

- [ ] All stories have status In Review
- [ ] All stories assigned to Lead Dev
- [ ] Priority order communicated in batch comment

**Handoff complete — this run is complete.** Do not look for additional Board requests. Do not start another task. End your response immediately so Paperclip can terminate this run cleanly and start the next queued run.

---

## Story Format

```
Title: [STORY] <user action or outcome>

## Context
<Why this matters to craftsmen. What problem it solves in their day-to-day work.>

## User Story
As a <craftsman / admin / ...>, I want to <action> so that <outcome>.

## Acceptance Criteria
- [ ] <measurable criterion 1>
- [ ] <measurable criterion 2>
- [ ] <measurable criterion 3>
...

## Non-Functional Requirements (if applicable)
- Performance: <e.g., "voice-to-quote round trip < 2s on 3G">
- Offline: <e.g., "draft saved locally if network unavailable">
- Security: <e.g., "quote PDF requires authentication">
Omit this section if no specific NFR applies to this story.

## Out of Scope
- <What is explicitly NOT included in this story>

## Prerequisites
- <Any external service, credential, or decision needed before this can be implemented>
  → If a human action is required: submit a Board approval request and reference it here
```

---

## Story Prioritisation Rules

When sequencing stories within or across batches, apply this priority order:

1. **Unblocks other stories** — prerequisites come first
2. **V1 success metric coverage** — stories directly tied to a measurable V1 goal take priority over nice-to-haves
3. **Risk reduction** — stories involving external services, new integrations, or architectural unknowns should be validated early to surface blockers sooner
4. **User value density** — prefer stories delivering visible craftsman value over purely technical stories of equivalent risk

---

## Human Escalation Procedure

When a feature requires external prerequisites that a human must handle (subscribing to a managed service, obtaining API keys, legal/compliance validation, strategic product decisions):

1. **Do not write the story yet** — write a prerequisite issue first.
2. **Submit a Paperclip Board approval request** with:
   - Title: `[HUMAN REQUIRED] <short description>`
   - What is needed, why, and which story it unblocks
   - Options if multiple approaches exist
3. Once the prerequisite is resolved, proceed with the story.

---

## Progress Visibility

For tasks spanning more than one session, post a progress comment on the issue before stopping:

> In progress — completed: \<what is done\>. Next: \<what remains\>.

Keep the issue status as **Started** and the assignee unchanged until your step is fully complete.

---

## Anti-Patterns & Red Flags

Stop and reassess if you notice yourself:

- Writing a story before all clarifying questions are answered
- Creating a story that bundles multiple unrelated user needs ("the user wants to create a quote AND manage clients AND view reports")
- Writing acceptance criteria that cannot be mechanically verified ("the UX should feel smooth", "it should work well")
- Skipping the Dedup Gate because "I'm sure this is new"
- Handing off stories without running the Size Checklist
- Creating more than 4 stories in a single batch without explicit Board approval
- Writing a story that assumes an unresolved prerequisite will be ready "by then"
- Prioritising technically interesting stories over stories that unblock others
- Writing Epics with abstract business language instead of concrete craftsman scenarios

---

## Verification Checklist

Before considering your work complete for this Board request, verify:

- [ ] Every story has a unique title not duplicating an existing non-cancelled issue
- [ ] Every story describes exactly one atomic user need
- [ ] Every acceptance criterion is testable by a QA agent
- [ ] Out of Scope is explicit for each story
- [ ] Prerequisites are resolved or tracked as prerequisite issues
- [ ] Stories are listed in priority order (unblocks > V1 metrics > risk > value)
- [ ] All stories assigned to Lead Dev with status In Review
- [ ] Batch comment posted with priority-ordered summary
