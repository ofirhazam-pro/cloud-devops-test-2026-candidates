# Problem 4 — Failed release

## Cause and risk

The pipeline publishes every build as `latest`, so the deployment cannot reliably identify or reproduce the image that failed or the image that was previously serving. A migration is run as part of deployment, but the release flow does not describe validation, production authorization, or what to do if the application fails after the schema has changed. Redeploying an old image does not reverse database changes.

## Proposed Azure DevOps Server flow

Keep Azure DevOps Server for Git and CI/CD. PR validation runs lint, tests, migration checks and a container build. Build once; publish a commit/run-tagged image to Artifact Registry and promote its recorded digest through validation and production.

Illustrative Azure Pipelines YAML; scripts are explicitly pseudocode. For Azure Repos, configure a **Build validation branch policy** on `main` for PR CI; do not rely on YAML `pr:` for Azure Repos. Configure a production environment approval/check and restrict approvers. Azure DevOps Server 2022 supports deployment approvals; verify configuration in the installed version.

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
              # push the image built/tested above, resolve its registry digest,
              # and write release-manifest.txt (digest, commit, run, migration).
              ./ci/push-image-and-record-digest.sh \
                "$(registry)/$(imageRepository):$(imageTag)"
            condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
            displayName: Push tested image and record digest
          - publish: $(Build.ArtifactStagingDirectory)/release-manifest.txt
            artifact: release-manifest
            condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))

  - stage: ValidationDeploy
    dependsOn: Validate
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployValidation
        environment: validation
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: release-manifest
                - script: |
                    # PSEUDOCODE: deploy recorded image@sha256:digest to validation;
                    # run smoke checks and fail the stage on errors.
                    ./ci/deploy-and-smoke-test.sh \
                      "$(Pipeline.Workspace)/release-manifest/release-manifest.txt"
                  displayName: Deploy exact digest and smoke test

  - stage: Production
    dependsOn: ValidationDeploy
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    # Configure a production environment approval/check in Azure DevOps.
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
                    # PSEUDOCODE: launch a one-off backward-compatible migration
                    # Job; deploy the same recorded digest; smoke-test; record it.
                    ./ci/migrate-deploy-and-record.sh \
                      "$(Pipeline.Workspace)/release-manifest/release-manifest.txt"
                  displayName: Migrate and deploy validated digest
```

The manifest records image digest, commit, build ID and migration version. Resolve the digest after push; deploy by `image@sha256:...`, not tag. Enable immutable tags and retain the prior production digest for rollback.

Use **expand/contract** migrations: make additive backward-compatible changes first, deploy code compatible with old/new schema, then remove obsolete schema in a later release after the rollback window. Make migrations observable and safe to retry where possible.

## Failed application release and rollback

If API errors start after the migration succeeds:

1. Stop rollout; inspect health, logs, migration result and recorded digest/schema version. Pause risky writes/jobs.
2. If schema change is backward-compatible, redeploy the prior image by recorded digest, smoke-test, and monitor; leave schema unchanged until reviewed.
3. If migration is incompatible, do **not** blindly restore the old image. Prefer a forward fix/compatibility release. If database restore is unavoidable, preserve current data, assess data loss and file consistency, and follow the tested recovery runbook before reopening.
4. Record digests, migration version, actions and checks; add regression tests.

Application rollback changes code, not data. Reverse migration may fail or discard writes; database restore is separate and can lose later changes.

## Verification

- Verify PR checks and production approval block unauthorized merge/deploy. Confirm one digest is deployed and tested in validation and then production, and matches the release record.
- Simulate failure after a compatible migration and test rollback by digest; tabletop the incompatible-migration forward-fix/restore decision.
- Proposed only; no Azure DevOps Server pipeline or production release was run.
