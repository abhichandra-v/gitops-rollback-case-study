# From manual deploys to GitOps with a rehearsed rollback — blast radius 100% → 10%

**The work:** replaced hand-run `kubectl` deploys with an ArgoCD GitOps pipeline
and an Argo Rollouts canary, so a bad release reaches at most 10% of traffic and
rolls back on its own — turning deploys from scary events into non-events.

---

## The problem

Releases were a person running commands against the live cluster. The cluster
*was* the source of truth, so it quietly drifted from anything in Git; a deploy
replaced every pod at once, so a bad build took 100% of traffic immediately; and
"rollback" meant someone remembering the previous image tag and re-running
`kubectl set image` under incident pressure. No audit trail, no rehearsal.

| | Before |
| --- | --- |
| Source of truth | the live cluster (drifts from Git) |
| Rollout | all-at-once, no traffic gating |
| Blast radius of a bad deploy | **100% of traffic** |
| Rollback | manual, ad-hoc, ~5–15 min under pressure |
| Audit trail | none |

## What I changed

1. **GitOps with ArgoCD.** Git is now the source of truth. An `Application`
   object watches the repo and continuously reconciles the cluster to match it,
   with `selfHeal` (reverts manual `kubectl` edits) and `prune` on. A deploy is a
   commit; a rollback is a revert; everything is in the git history.
2. **Canary rollouts with Argo Rollouts.** A new version takes 10% of traffic,
   pauses, then ramps 25% → 50% → 100% — instead of all-at-once.
3. **An automated gate.** At each pause an `AnalysisTemplate` checks the canary's
   success rate against Prometheus; below 95% and the rollout **auto-aborts and
   shifts 100% back to stable with no human involved.**
4. **A rehearsed rollback.** Three recovery paths (automatic, one-command, and
   `git revert`), documented in [`rollback-drill.md`](./rollback-drill.md) and
   run in staging monthly so the recovery time is known, not hoped for.

## The result

| | Before | After |
| --- | --- | --- |
| Source of truth | live cluster (drifts) | **Git** (reconciled, self-healing) |
| Blast radius of a bad deploy | 100% | **≤ 10%** (canary first) |
| Rollback | manual, ~5–15 min | **automatic**, or one command / a git revert |
| Who has to be awake for a bad deploy | an engineer | **no one** (auto-abort) |
| Audit trail | none | **full git history** of every change |
| Manual steps per release | ~4 | **1** (git push) |

**Why it matters to the business:** a broken release can no longer take the whole
service down — it's capped at 10% of traffic and reverts itself — and recovery is
a rehearsed, measured procedure rather than a scramble. Deploys stop being events
teams schedule around.

---

*Stack: ArgoCD (GitOps), Argo Rollouts (canary + analysis), Kustomize,
Prometheus, Kubernetes. Manifests validated with `kustomize build`; the full
before/after is in the repo.*
