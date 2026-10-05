# The rehearsed rollback

A rollback nobody has practiced is a wish, not a plan. With GitOps + Argo
Rollouts there are three recovery paths, fastest first. All three are safe to
rehearse in staging — and should be, on a schedule.

## Path 1 — automatic (no human) during a rollout

If the canary's success rate drops below 95%, the `AnalysisTemplate` fails, Argo
Rollouts **aborts the rollout and shifts 100% of traffic back to the stable
version by itself**. The bad version never got more than 10% of traffic. This is
the common case and needs no one awake.

## Path 2 — one command, mid-rollout

If you catch something the metric didn't:

```bash
# instantly abort and return to stable
kubectl argo rollouts abort checkout -n checkout

# or undo to the previous known-good revision
kubectl argo rollouts undo checkout -n checkout
```

Traffic returns to stable in seconds because the old ReplicaSet is still there
(`revisionHistoryLimit: 5`).

## Path 3 — git revert (the GitOps way, for an already-promoted bad version)

```bash
git revert <bad-commit> && git push
```

ArgoCD sees Git changed and reconciles the cluster back. Slower than Path 2
(bounded by the sync interval), but it keeps Git as the source of truth and
leaves a clean audit trail of what happened.

## The drill (run in staging monthly)

1. Deploy a deliberately-broken image (returns 500s).
2. Watch the canary analysis fail and auto-abort (Path 1). Record the time from
   deploy to traffic-restored.
3. Deploy another, and this time abort manually (Path 2). Record the time.
4. Promote a bad version fully, then `git revert` (Path 3). Record the time.
5. Log all three times. If any grew since last month, find out why.

**The point:** recovery is measured in seconds, it's been rehearsed, and the
times are written down — not "we'll figure it out when it breaks."
