# The "before" — manual kubectl deploys, no safe rollback

This is how releases happened before GitOps. Every deploy was a person running
commands against the live cluster, with nothing recording what *should* be
running and no rehearsed way back.

## The release ritual

```bash
# build + push (tag is almost always :latest)
docker build -t registry.example.com/checkout:latest .
docker push registry.example.com/checkout:latest

# deploy: edit live state by hand
kubectl set image deployment/checkout checkout=registry.example.com/checkout:latest
# ...or kubectl apply -f a manifest that lives on someone's laptop

# then watch logs and hope
kubectl logs -f deployment/checkout
```

## Why it hurt

- **No source of truth.** The cluster *is* the state. If someone ran a
  `kubectl edit` at 2am, nobody else knows. Git and reality drift apart.
- **All-at-once rollout.** `set image` replaces pods with no traffic gating — a
  bad build takes 100% of traffic immediately. Blast radius = everyone.
- **No rehearsed rollback.** "Roll back" means someone remembering the previous
  image tag and manually re-running `set image` — under pressure, during an
  incident. Mean time to recovery is however long that takes.
- **No audit trail.** Who deployed what, when, and why? Not captured.

**Baseline numbers (typical for this setup):**

| Metric | Before |
| --- | --- |
| Source of truth | the live cluster (drifts) |
| Rollout strategy | all-at-once, no traffic gating |
| Blast radius of a bad deploy | 100% of traffic |
| Rollback | manual, ad-hoc, ~5–15 min under pressure |
| Audit trail | none |
