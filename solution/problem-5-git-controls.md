# Problem 5 — Git controls

## Cause and risk

Direct pushes skip review/checks; deploying every push can ship an unreviewed defect. Lightweight Azure Repos and pipeline controls address this without another CI/CD system.

## Proposed workflow

1. Developers use short-lived `feature/*` or `fix/*` branches and PR into protected `main`; disallow routine direct and force pushes.
2. Set Azure Repos **Build validation**, **minimum one reviewer** (not author), and **comment resolution** policies. Reset votes on new changes; restrict bypass/direct-push permissions to audited maintainers.
3. PR CI runs lint, tests and migration checks; require success before merge. Main builds/publishes once; production deploy requires named maintainer approval and uses the recorded image digest.
4. Urgent fixes use the same branch, PR, check, review and production approval path—expedite review, not safeguards. Only an exceptional platform outage justifies restricted, logged bypass; follow it with a validated audit PR.

## Verification

- Confirm direct push is blocked and a PR with failing checks/unresolved comments cannot merge.
- Confirm only authorized approvers can release and production uses the validated digest; tabletop the urgent-fix path.

These policies and tests are proposed; no Azure DevOps Server project was configured for this submission.
