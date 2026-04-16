---
name: technical-specification
description: Formalizes the procedure for receiving a PM story, analyzing the codebase, writing a complete technical specification, and handing off to the PO for ticket decomposition.
---

# Technical Specification — Shared Skill

> This skill formalizes the procedure for receiving a PM story, analyzing the codebase, writing a complete technical specification, and handing off to the PO for ticket decomposition.

**Loaded by:** Lead Dev agent.

---

## When This Skill Applies

When a story issue is assigned to you by the PM with status **In Review**, and the story has acceptance criteria and context but no technical specification yet.

---

## Iron Laws

1. **NO SPEC WITHOUT CODEBASE ANALYSIS** — always query GitNexus and read existing code patterns before writing. Never spec from assumptions or memory alone. Every file path, class name, and pattern reference must be verified against the current codebase state.

2. **NO SPEC WITHOUT DECISION CLOSURE** — the PO must be able to decompose into tickets without making any architectural choice. If the spec leaves a design decision open ("the PO can decide whether to use X or Y"), it is incomplete. The spec decides.

3. **NO SPEC WITHOUT CONCRETE REFERENCES** — never write "TBD", "to be determined", "add appropriate validation", "implement as needed", or any vague instruction. Every file path must be a real path. Every pattern reference must match what exists. Every edge case must be explicit.

4. **NO SPEC WITHOUT ATOMIC SCOPE** — PM stories are atomic. If a spec requires significant exploration that forces splitting, comment on the issue explaining why and ask the PM to split the story first. Do not write a partial spec.

5. **NO HANDOFF WITHOUT VERIFICATION CHECKLIST** — the Verification Checklist must pass before assigning the issue to the PO. No exceptions.

---

## The Process

### Phase 1: Receive & Analyze

Read the story in full: context, user story, acceptance criteria, NFRs, prerequisites, out of scope.

**Actions:**

1. Read the story issue thoroughly — understand the user need, not just the technical surface.
2. Query GitNexus MCP to analyze:
   - Call chains touching the affected domain entities.
   - Cross-package dependencies relevant to this story.
   - Existing patterns in the affected layers (resolvers, services, activities, workflows).
3. Read the actual source files for every module you plan to reference in the spec.
4. Identify shared types in `@crafto/types` that will need to be extended or created.
5. Check for existing Temporal workflows/activities that interact with the same domain.

#### Gate: Receive & Analyze

- [ ] Story fully understood — can articulate the user need in one sentence
- [ ] Codebase analyzed — can name every file that will be touched
- [ ] Existing patterns identified — know which conventions to follow
- [ ] Shared types reviewed — know what exists vs what needs to be created

If any gate fails: continue analysis. Do NOT proceed to drafting.
If all pass: proceed to Phase 2.

---

### Phase 2: Identify Stack Layers

List every stack layer impacted by this story.

**Available layers:**

| Layer | Description |
|---|---|
| `internal-api` | Data models, business logic, internal endpoints |
| `craftman-gateway` | Craftsman-facing gateway endpoints, DTOs, validation |
| `admin-gateway` | Admin-facing gateway endpoints, DTOs, validation |
| `temporal-activities:<package-name>` | Individual activity implementations |
| `temporal-workflow:<name>` | Workflow orchestration |
| `craftman-mobile-app` | Mobile UI (React Native + Expo) |
| `craftman-interface` | Craftsman-facing web UI (React) |
| `admin-interface` | Admin-facing web UI (React) |

**Actions:**

1. Cross-reference each acceptance criterion against the layers it requires.
2. Verify no layer is missing — a common miss is forgetting `temporal-activities` when a workflow change is needed, or forgetting the gateway when a new `internal-api` endpoint is added.

#### Gate: Identify Layers

- [ ] Every acceptance criterion maps to at least one layer
- [ ] No layer is missing (cross-referenced against acceptance criteria)
- [ ] Layer list matches what you found in codebase analysis

If any gate fails: revisit analysis.
If all pass: proceed to Phase 3.

---

### Phase 3: Draft Spec

Write the technical specification as a comment on the story issue, using the Spec Format below.

**Actions:**

For each impacted layer:

1. List files to modify with real paths and what changes are needed.
2. List files to create with real paths and what they contain.
3. Write numbered implementation steps — specific enough that a developer can follow them without making design decisions.
4. Document edge cases: security concerns, performance considerations, error handling, offline scenarios.
5. Specify tests required: name the test file, describe each test case.

**Rules:**

- Reference actual class names, decorator patterns, resolver patterns from the codebase.
- If a pattern does not exist yet, describe the new pattern completely — do not say "follow the existing pattern for X" when X does not exist.
- For MongoDB changes: specify the collection, the document shape changes, any indexes needed.
- For GraphQL changes: specify the type/input/query/mutation changes, including the decorator usage (type-graphql 2 style).
- For Temporal changes: specify whether it is a new activity, a new workflow, or a modification, and which worker runs it.

#### Gate: Draft Spec

- [ ] Every layer has: files to modify, files to create, key steps, edge cases, tests
- [ ] Every file path is a real path verified against the codebase (or a new file with explicit location rationale)
- [ ] No placeholders, no TBD, no vague instructions (Iron Law 3)
- [ ] Implementation steps are specific enough for a developer to execute without design decisions (Iron Law 2)

If any gate fails: revise.
If all pass: proceed to Phase 4.

---

### Phase 4: Verify

Run the Verification Checklist (see below).

#### Gate: Verify

- [ ] All checklist items pass

If any gate fails: fix before proceeding.
If all pass: proceed to Phase 5.

---

### Phase 5: Handoff — END OF RUN

**Actions:**

1. Post the spec as a comment on the story issue.
2. Set the issue status to **`todo`**.
3. Assign the issue to the PO for ticket decomposition.

#### Gate: Handoff

- [ ] Spec posted as comment on the story issue
- [ ] Issue status set to `todo`
- [ ] Issue assigned to PO

**Handoff complete — this run is complete.** Do not process additional stories from your inbox. Do not look for more work. End your response immediately so Paperclip can terminate this run cleanly and start the next queued run.

---

## Spec Format

Use this exact structure when writing the specification comment on the story issue:

```markdown
## Technical Specification

### Stack Layers Impacted

<List each layer affected: internal-api, craftman-gateway, admin-gateway,
temporal-activities:<package>, temporal-workflow:<name>, craftman-mobile-app,
craftman-interface, admin-interface>

### Architecture Rules (relevant to this story)

<List the patterns, conventions, and constraints from the codebase that apply.
Be specific: reference file paths, module names, class names, decorators, etc.>

### Implementation Breakdown

For each impacted layer:

#### <Layer name>

- **Files to modify**: `<path>` — <what to change and why>
- **Files to create**: `<path>` — <what it contains and why>
- **Key steps**: <numbered list of implementation steps>
- **Edge cases**: <security, performance, tricky logic>
- **Tests required**:
  - Unit: <what to test, which test file>
  - Integration: <what to test, which test file>
```

---

## Codebase Analysis Guide

When analyzing the codebase for a spec, follow this sequence:

1. **Domain entities** — query GitNexus for the entity/entities referenced in the story. Understand the current data model, where it is used, and what depends on it.
2. **Call chains** — trace the request path from gateway to internal-api to database (or from workflow to activity to external service). Identify every touchpoint.
3. **Existing patterns** — read 2-3 existing implementations of similar scope in the same layer to understand conventions (resolver structure, service patterns, activity patterns, test structure).
4. **Shared types** — check `@crafto/types` for existing type definitions that will be extended. Check if codegen will need to run.
5. **Cross-package impacts** — if the change affects a shared package, identify all consumers.

---

## Progress Visibility

For specs spanning more than one session, post a progress comment on the issue before stopping:

> In progress — completed: \<what is done\>. Next: \<what remains\>.

Keep the issue status as **`in_progress`** and the assignee unchanged until the spec is fully written.

---

## Anti-Patterns & Red Flags

Stop and reassess if you notice yourself doing any of the following:

- **Speccing from memory** — writing a spec without having read the codebase first ("I know how this works").
- **Delegating decisions** — leaving architecture decisions to the PO ("the PO can decide whether to use a new collection or extend the existing one").
- **Using placeholder paths** — writing placeholder file paths ("add the appropriate resolver in the relevant file").
- **Vague test specs** — specifying tests without naming the test file and describing the test cases.
- **Missing domain edge cases** — forgetting edge cases for the specific domain (voice input failures, offline scenarios, partial data, concurrent mutations).
- **Forgetting a stack layer** — common misses:
  - Forgetting `temporal-activities` when specifying a workflow change.
  - Forgetting the gateway when adding a new `internal-api` endpoint that clients need.
  - Forgetting codegen when adding GraphQL types.
- **Leaving design decisions open** — writing a spec that requires the developer to make non-trivial design decisions.
- **Pattern mismatch** — specifying a pattern that does not match what exists in the codebase.
- **Assuming types exist** — assuming a shared type exists without checking `@crafto/types`.

---

## Verification Checklist

Before handing off to the PO, verify every item:

- [ ] Every acceptance criterion from the story maps to at least one implementation step
- [ ] Every impacted stack layer is listed and has a complete breakdown
- [ ] File paths are real paths verified against the codebase (not guessed)
- [ ] Architecture patterns referenced match what currently exists in the codebase
- [ ] Edge cases are documented for each layer (not just "handle errors")
- [ ] Tests required section names specific test files and describes test cases
- [ ] No placeholders, no TBD, no vague instructions
- [ ] The PO can create tickets from this spec without asking architectural questions
- [ ] Spec posted as comment on the story issue
- [ ] Issue status set to `todo`, issue assigned to PO
