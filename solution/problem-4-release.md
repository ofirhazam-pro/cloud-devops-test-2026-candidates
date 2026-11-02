# Problem 4 — Failed release

## Cause and risk

The pipeline publishes every build as `latest`, so the deployment cannot reliably identify or reproduce the image that failed or the image that was previously serving. A migration is run as part of deployment, but the release flow does not describe validation, production authorization, or what to do if the application fails after the schema has changed. Redeploying an old image does not reverse database changes.

## Proposed Azure DevOps Server flow

Keep Azure DevOps Server as the Git and CI/CD system. Use a pull request into `main` to trigger CI: lint/type checks, unit and integration tests, migration checks, and a container build. Build once; tag the image with the source commit and pipeline run, push it to Artifact Registry, and promote that exact immutable digest through validation and production rather than rebuilding.

The following is illustrative Azure Pipelines YAML. For Azure Repos Git on Azure DevOps Server, configure a **Build validation branch policy** on `main` to run the validation pipeline for PRs; do not rely on a YAML `pr:` trigger. `PSEUDOCODE` steps stand for scripts or templates the team would implement for its environment. Configure the `production` environment with an Azure DevOps Server approval/check and restrict who can approve it; confirm the installed Server version supports the environment/check syntax. If it does not, use a protected deployment pipeline/stage with an explicit manual approval gate.

```yaml
trigger:
  branches:
    include:
      - main

variables:
  imageRepository: survey-api
  registry: REGION-docker.pkg.dev/PROJECT/REPOSITORY
  imageTag: $(Build.SourceVersion)-$(Build.BuildId)

stages:
  - stage: Validate
    jobs:
      - job: BuildAndTest
        steps:
          - script: |
              # PSEUDOCODE: run lint, unit/integration tests,
              # migration compatibility checks, and build the container locally.
              ./ci/validate-and-build.sh

          - script: |
              # PSEUDOCODE: authenticate using the agent's attached identity,
              # tag and push the image already built/tested above, resolve its
              # registry digest, and write release-manifest.txt.
              ./ci/push-image-and-record-digest.sh \
                "$(registry)/$(imageRepository):$(imageTag)"
            condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
            displayName: Push tested image and record digest
          - publish: $(Build.ArtifactStagingDirectory)/release-manifest.txt
            artifact: release-manifest
            condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))

  - stage: Production
    dependsOn: Validate
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    # Configure an approval/check on the "production" environment in Azure DevOps.
    jobs:
      - deployment: DeployProduction
        environment: production
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: release-manifest
                - script: |
                    # PSEUDOCODE: read the recorded digest; run a backward-compatible
                    # expand migration; deploy API using image@sha256:digest;
                    # run smoke checks; persist the deployed digest and pipeline run
                    # in a release record (and as a Kubernetes annotation).
                    ./ci/deploy-production.sh \
                      "$(Pipeline.Workspace)/release-manifest/release-manifest.txt"
                  displayName: Migrate, deploy exact digest, verify and record
```

`release-manifest.txt` should contain at least the full image reference (`...@sha256:...`), source commit, pipeline run/build ID, and migration version. The push step must resolve the digest returned by Artifact Registry after upload; a commit/build tag is a convenient lookup tag, not a substitute for deploying by digest. Do not overwrite or reuse release tags. Keep the prior production digest and release record so the on-call engineer can identify the exact rollback candidate.

The deployment sequence assumes an **expand/contract** database strategy: first make additive, backward-compatible schema changes; deploy code that works with both old and new schema; later, in a separately authorized release after the rollback window, remove obsolete schema. For example, add a nullable column before code starts using it; backfill safely; only later remove an old column once no deployed version needs it. Make migrations observable, bounded, and safe to retry where possible. Do not combine irreversible destructive schema changes with a release that might need application rollback.

## Failed application release and rollback

If API errors start after the migration succeeds:

1. Stop further rollout and pause jobs or writes if they risk corrupting data. Check deployment health, logs, metrics, migration outcome, and the recorded image digest/schema version.
2. If the migration is backward-compatible, redeploy the previously recorded production image **by digest**, not `latest`. Verify the old version can operate with the migrated schema, run smoke/integration checks, and monitor error rates. Keep the schema change in place until the cause and safe cleanup are understood.
3. If the migration is incompatible with the previous application, **do not blindly redeploy the old image**. Keep the release paused and assess the exact schema and data changes. Prefer a forward fix or a new compatibility release that restores service while retaining the migrated data. If a database rollback/restore is unavoidable, first assess data loss and consistency, preserve the current database, and follow the tested database recovery procedure; coordinate file/object state where relevant. Reopen traffic only after validation.
4. Record the incident, the exact old and new image digests, migration version, actions, and verification results. Add a regression test and update the migration/rollback procedure.

An application rollback changes code, not database state. A reverse migration is not automatically safe: it can fail or discard data written by the new version. Schema rollback or point-in-time restore is a separate, explicitly assessed recovery operation and can lose legitimate writes after its recovery point.

## Verification

- In a non-production environment, prove a PR cannot merge without its required CI checks; verify the production stage cannot deploy without its configured approval.
- Confirm one build produces a manifest with the immutable registry digest, commit, run ID, and migration version. Deploy that same digest to validation and production, then confirm the live Deployment image and release record match it.
- Test a controlled application failure after a backward-compatible migration. Roll back by the recorded digest and verify the prior application works with the expanded schema.
- Test an incompatible-migration failure path as a tabletop or isolated restore drill: prove the runbook prevents blind old-image redeployment and identifies forward-fix versus data-restore decision owners.
- These are proposed controls and tests; no Azure DevOps Server pipeline or production release was run for this submission.
