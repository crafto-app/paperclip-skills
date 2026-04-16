---
name: infra-ticket
description: Formalizes the procedure for receiving an infrastructure ticket, implementing Kustomize manifest changes, validating builds, committing to main, and verifying ArgoCD sync.
---

# Infra Ticket — Shared Skill

> This skill formalizes the procedure for receiving an infrastructure ticket from the Lead Dev, implementing Kustomize manifest changes, validating builds, committing to main, and verifying ArgoCD sync.

**Loaded by:** DevOps/SRE agent.

---

## When This Skill Applies

When an infra ticket is assigned to you by the Lead Dev (or CTO) with an implementation plan describing infrastructure changes needed (new service, new deployment, config changes, namespace creation, secret wiring, ArgoCD app registration, etc.).

---

## Iron Laws

### 1. NO COMMIT WITHOUT KUSTOMIZE BUILD

Run `kustomize build` on every modified overlay before committing. If it fails, fix before committing. Never push broken manifests to main.

### 2. KUSTOMIZE ONLY

Never generate or use Helm charts. All manifests use Kustomize with `base/` -> `overlays/` structure. No exceptions.

### 3. NO SECRETS IN PLAIN TEXT

All secrets go through Infisical. Never commit secret values to git. Use InfisicalSecret CRDs with `creationPolicy: Orphan`. Never add `argocd.argoproj.io/instance` label in InfisicalSecret managed secret templates.

### 4. INVESTIGATE BEFORE RESTART

If something breaks after your change, investigate logs, events, and resource state before restarting any service. Restarting is never the first action.

### 5. VERIFY ARGOCD SYNC

After pushing to main, verify ArgoCD picks up the change and syncs successfully. If sync fails, investigate immediately. Never walk away from a failed sync.

### 6. NO COMMIT WITHOUT UNDERSTANDING

Read the ticket and plan fully. Understand what infrastructure change is needed and why before touching any manifests. If the plan is ambiguous, ask the Lead Dev for clarification.

---

## The Process

### Phase 1: Receive & Understand

- Read the ticket: what infra change is needed, which overlays are affected, any dependencies.
- Read the implementation plan from the Lead Dev's spec.
- Identify which files in `base/` and `overlays/` need to change.
- Check if new namespaces, secrets, or ArgoCD apps are needed.

**Gate:** Can articulate what will change and why.

### Phase 2: Analyze Current State

- Read the existing manifests that will be modified.
- Check the current state of the relevant resources on the cluster:
  ```bash
  kubectl get <resource> -n <namespace>
  ```
- Verify ArgoCD is healthy before making changes:
  ```bash
  kubectl get applications -n argocd
  ```

**Gate:** Current state understood, no pre-existing issues that would mask the change.

### Phase 3: Implement

- Modify manifests following the `base/` -> `overlays/` structure.
- For new services: create in `base/`, patch in relevant overlays.
- For secrets: create InfisicalSecret CRD (never plain text). Always set `creationPolicy: Orphan`.
- For namespaces: add to `base/system/namespaces/system-namespaces.yaml` (never create namespace files elsewhere, never duplicate).
- For ArgoCD apps: use ApplicationSet pattern in `argocd/clusters/staging/appsets/`.
- Remember: ArgoCD ApplicationSets use `prune: false`.

**Gate:** Manifests modified according to plan.

### Phase 4: Validate

- Run `kustomize build` on every modified overlay:
  ```bash
  kustomize build overlays/staging
  kustomize build overlays/production  # only if modified
  ```
- Both must succeed with zero errors.
- Review the rendered output to verify it matches your intent.

**Gate:** `kustomize build` passes for all modified overlays.

### Phase 5: Commit & Push

- Commit with Commitizen format: `<type>(<scope>): <description>`
  - Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `ci`
  - Scope: the affected component (e.g., `staging`, `argocd`, `internal-api`)
- Push directly to `main` (DevOps workflow — no branches, no PRs).

**Gate:** Commit pushed to main.

### Phase 6: Verify Deployment

- Wait for ArgoCD to detect the change (usually < 2 minutes).
- Check sync status:
  ```bash
  kubectl get applications -n argocd | grep <app-name>
  ```
- If **Synced + Healthy**: deployment successful.
- If **OutOfSync** or **Degraded**: investigate immediately (logs, events, describe).
- Verify the actual resources are running correctly:
  ```bash
  kubectl get pods -n <namespace>
  kubectl logs <pod> -n <namespace> --tail=50
  ```

**Gate:** ArgoCD synced, resources healthy.

---

## Progress Visibility

For tickets spanning more than one session, post a progress comment on the issue:

> :arrows_counterclockwise: In progress — completed: \<what is done\>. Next: \<what remains\>.

---

## Handoff

Once the infra change is deployed and verified:

1. Post a comment on the ticket confirming deployment:
   > :white_check_mark: Infrastructure change deployed. ArgoCD synced. Resources healthy.
2. **Assess application impact**: does this infra change affect the runtime of application services? Examples: new environment variable, new secret, new service endpoint, changed resource limits, new namespace, modified network policy.
   - **If no app impact**: set the issue status to `done`. Do not reassign — you remain the last assignee as the agent who completed the work.
   - **If app impact**: set the issue status to `done`. Additionally, **create a new Paperclip ticket** describing the application-side changes required (e.g., "update config to use new env var X", "add SDK client for new service Y"). Assign this new ticket to the Lead Dev so it enters the normal specification → decomposition → implementation pipeline.

3. **Stop — this run is complete.** Do not process additional tickets or start monitoring. End your response immediately so Paperclip can terminate this run cleanly and start the next queued run.

---

## Anti-Patterns & Red Flags

- Committing without running `kustomize build` first.
- Creating Helm charts or templated YAML instead of Kustomize.
- Writing secret values directly in manifest files.
- Restarting pods/deployments as a first response to issues.
- Pushing to main without verifying the rendered output.
- Creating namespaces outside the central namespace file (`base/system/namespaces/system-namespaces.yaml`).
- Modifying production overlay without explicit instruction to do so.
- Not verifying ArgoCD sync after pushing.
- Adding `argocd.argoproj.io/instance` label in InfisicalSecret managed secret templates.
- Completing an infra change that impacts app runtime without creating a follow-up ticket for the application team.

---

## Verification Checklist

- [ ] Ticket and plan fully read and understood
- [ ] Manifests follow `base/` -> `overlays/` structure
- [ ] No secrets in plain text (InfisicalSecret CRDs used)
- [ ] `kustomize build` passes for all modified overlays
- [ ] Commit message follows Commitizen format
- [ ] Pushed to main
- [ ] ArgoCD synced successfully
- [ ] Resources are healthy on cluster
- [ ] Confirmation comment posted on ticket
- [ ] If change impacts app runtime: follow-up ticket created and assigned to Lead Dev
