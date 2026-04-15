---
name: cluster-monitoring
description: Proactive monitoring heartbeat procedure covering cluster health checks, issue identification, root cause investigation, and escalation.
---

# Cluster Monitoring — Shared Skill

Proactive monitoring heartbeat: check cluster health, identify issues, investigate root causes, escalate when needed.

**Loaded by:** DevOps/SRE agent.

---

## When This Skill Applies

On every heartbeat (every 1 hour). This is a proactive monitoring skill — it runs even when no ticket is assigned.

**Cluster scope:** staging only (production monitoring to be added later).
**Kubeconfig:** `/etc/paperclip/kubeconfigs/staging`
**Reports to:** CTO agent (never directly to Board unless human action is required).

---

## Iron Laws

1. **NO ACTION WITHOUT INVESTIGATION** — never restart, scale, or modify resources before understanding the root cause. Logs first, events second, resource state third, action last.
2. **NO MULTITASKING WITHOUT RESOLUTION** — stop at the first issue found in the check sequence. Investigate it fully and resolve or escalate before moving to the next check.
3. **NO FIX WITHOUT DOCUMENTATION** — if a check reveals an issue, always create or update a Paperclip issue. Never silently fix something without documenting it.
4. **NO HEARTBEAT WITHOUT REPORT** — every heartbeat must end with either an issue report or an all-clear message. Never end a heartbeat without posting a status.

---

## The Process

Five checks, executed sequentially. Stop at the first failure, resolve or escalate, then continue.

### Check 1: ArgoCD Sync Status

```bash
kubectl --kubeconfig /etc/paperclip/kubeconfigs/staging get applications -n argocd
```

- If any app is `OutOfSync` or `Degraded`: investigate the app diff, identify root cause, fix or escalate to CTO.
- If any app is stuck in `Progressing` for more than 10 minutes: investigate sync logs and resource events.

**Gate:** All apps `Synced` + `Healthy`, or issue created/escalated.

### Check 2: Pod Health

```bash
kubectl --kubeconfig /etc/paperclip/kubeconfigs/staging get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
```

- For any non-Running / non-Succeeded pod: check logs and events, identify root cause.
- `OOMKilled` → check memory limits and recent usage trends.
- `CrashLoopBackOff` → read logs before any other action.

**Gate:** All pods healthy, or issue created/escalated.

### Check 3: Sentry Errors

- Check Sentry dashboard for new or spiking errors in the last hour.
- If a new error cluster appeared: investigate, correlate with recent deployments, create a Paperclip issue.

**Gate:** No new critical errors, or issue created.

### Check 4: Certificate Expiry

```bash
kubectl --kubeconfig /etc/paperclip/kubeconfigs/staging get certificates -A
```

- Flag any certificate expiring in less than 14 days.
- If expiring soon: investigate cert-manager logs, check ClusterIssuer health.

**Gate:** All certs valid for >14 days, or issue created.

### Check 5: Report

If all checks passed with no issues:

> ✅ Heartbeat — all systems nominal. `<timestamp>`

If any issue was found and handled:

> ⚠️ Heartbeat — issue found: `<brief description>`. Status: `<resolved/escalated/monitoring>`. See issue `#<id>`.

---

## Escalation Protocol

When an issue requires action beyond your scope:

1. **Create a Paperclip issue** with:
   - Title: `[INFRA] <short description>`
   - Severity: `Critical` / `Warning` / `Info`
   - Root cause analysis (what you found)
   - Impact assessment (what is affected)
   - Recommended action

2. **For technical decisions** (architecture, scaling strategy): assign to **CTO**.
3. **For human-required actions** (OVH account, DNS, external services): assign to **Board** with `[HUMAN REQUIRED]` prefix.
4. **For application bugs** (code issue causing infra symptoms): assign to **Lead Dev**.

---

## Anti-Patterns

- Restarting pods without reading logs first.
- Skipping checks because "everything was fine last time."
- Silently fixing an issue without creating a Paperclip issue.
- Ending a heartbeat without reporting status.
- Investigating multiple issues simultaneously instead of one at a time.
- Escalating without a root cause analysis ("something is wrong with the pods").
- Ignoring Sentry errors because they are "not infra."

---

## Verification Checklist

Run through this list at the end of every heartbeat.

- [ ] ArgoCD sync status checked
- [ ] Pod health checked
- [ ] Sentry errors checked
- [ ] Certificate expiry checked
- [ ] Status reported (all-clear or issue report)
- [ ] Any issues found are documented as Paperclip issues
- [ ] Escalations include root cause and recommended action
