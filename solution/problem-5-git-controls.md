# Problem 5 — Git controls

## Cause and risk

Direct pushes to `main` skip review and required checks, and coupling every push to production means an unreviewed change can become an outage. A small team can add lightweight controls without adding another CI/CD platform.

## Proposed workflow

1. Protect `main`; developers create short-lived `feature/<change>` or `fix/<change>` branches and open pull requests back to `main`. No routine direct pushes or force pushes to `main`.
2. Require Azure Repos branch policies on `main`: **Build validation** for the PR validation pipeline, at least **one reviewer** (not the author), **comment resolution**, and a limited merge strategy such as squash. Reset reviewer votes when new changes are pushed. Use branch security to restrict direct contribution, force-push, and policy-bypass permissions to a small, audited maintainer group while allowing PR completion.
3. PR validation runs formatting/lint, unit and integration tests, migration compatibility checks, and any fast security checks. Require the validation policy to pass before merge; cancel or supersede stale builds when new commits arrive.
4. After merge, the main-branch pipeline builds and publishes the image once, records its immutable digest, and deploys that digest only after the configured **production approval**. Limit production deployment permissions and approval authority to named maintainers; record the approver and release artifact.
5. Keep the release process and checks the same for urgent fixes: branch from `main`, make the smallest fix, run required validation, get one available reviewer, merge, then obtain production authorization. For a genuine emergency, the designated maintainer can expedite reviewer response and run the same required checks; do not make a direct production push or silently bypass policy. If a documented policy bypass is unavoidable due to an exceptional platform outage, restrict it to authorized maintainers, record the reason, and open a review/audit PR with validation immediately afterward.

## Verification

- Attempt a direct push to `main` as a normal developer and confirm it is rejected.
- Open a PR with a failing validation build and unresolved comment; confirm merge is blocked until checks pass and comments are resolved.
- Confirm only authorized approvers can authorize production, and that production receives the exact digest produced by the validated main-branch build.
- Exercise an urgent-fix tabletop: use the same branch, PR, check, review, and approval sequence with expedited response, and verify bypasses are logged if exceptional recovery is necessary.

These policies and tests are proposed; no Azure DevOps Server project was configured for this submission.
