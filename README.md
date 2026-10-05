# checkout — GitOps + safe-rollback case study

A real before/after: hand-run `kubectl` deploys (all-at-once, 100% blast radius,
manual rollback) → **ArgoCD GitOps + an Argo Rollouts canary** that caps a bad
release at 10% of traffic and rolls back on its own.

👉 **Read the write-up: [CASE-STUDY.md](./CASE-STUDY.md)**
👉 **The rollback playbook: [rollback-drill.md](./rollback-drill.md)**

## Result

| | Before | After |
| --- | --- | --- |
| Blast radius of a bad deploy | 100% | **≤ 10%** |
| Rollback | manual, ~5–15 min | **automatic** / one command / git revert |
| Source of truth | live cluster | **Git** (self-healing) |

## What's here

| Path | What it is |
| --- | --- |
| `before/deploy-notes.md` | ❌ the manual `kubectl` release ritual (the baseline) |
| `gitops/apps/checkout.yaml` | ✅ the ArgoCD Application (Git → cluster) |
| `gitops/base/rollout.yaml` | ✅ the Argo Rollouts canary (10→25→50→100%) |
| `gitops/base/analysistemplate.yaml` | ✅ the automated success-rate gate |
| `gitops/base/services.yaml` | stable + canary services |
| `rollback-drill.md` | the three recovery paths + the monthly drill |

## Validate the manifests

```bash
kustomize build gitops/base      # renders all 4 resources
```

## How it works

A commit to this repo changes `gitops/base`; ArgoCD reconciles the cluster to
match. Argo Rollouts shifts traffic to the new version in steps, checking the
success rate at each pause. A failed check aborts and returns to stable
automatically — the bad version never gets more than 10% of traffic.

---

Built by [Abhichandra](https://www.abhichandra.online) — SRE / Platform Engineer.
Available for scoped platform & reliability work.
