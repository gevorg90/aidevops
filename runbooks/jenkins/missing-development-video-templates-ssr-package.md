# Missing video-templates-ssr development package in Nexus

## Symptom and cause

`website-front-end` or another development consumer fails during dependency
installation with an HTTP 404 for a tarball such as:

```text
https://rfhub.rfservs.com/repository/rf_npm/%40renderforest_internal/video-templates-ssr/-/video-templates-ssr-1.0.1-development.tgz
```

The operator sometimes deletes old packages from the internal development Nexus
repository. A previously successful producer build does not guarantee that its
package still exists. Rebuilding only the consumer cannot restore a deleted
dependency.

Build the matching `video-templates-ssr` development branch to republish the
package, then rebuild the failed consumer branch.

## Branch and package mapping

The development pipeline publishes `@renderforest_internal/video-templates-ssr`
to `https://rfhub.rfservs.com/repository/rf_npm/`.

| Producer and consumer branch | Local namespace | Package version |
| --- | --- | --- |
| `development` | `local` | `1.0.0-development` |
| `development-1` | `local-1` | `1.0.1-development` |
| `development-2` | `local-2` | `1.0.2-development` |
| `development-3` | `local-3` | `1.0.3-development` |
| `development-4` | `local-4` | `1.0.4-development` |
| `development-5` | `local-5` | `1.0.5-development` |
| `development-6` | `local-6` | `1.0.6-development` |

For a numbered branch `development-N`, the expected tarball is
`video-templates-ssr-1.0.N-development.tgz`. Confirm the current pipeline and
consumer dependency URL before acting if this mapping has changed.

## Recovery procedure

1. Read [Jenkins infrastructure](../../inventory/jenkins-infrastructure.md)
   and the relevant [local environment inventory](../../inventory/local-environment/local-nginx-structure.md).
   Inspect the exact failed dependency URL and branch. Distinguish a deleted
   package from missing registry authentication using the
   [private-package authentication runbook](pnpm-private-package-auth.md).
   Do not treat every private-registry 404 as proof of deletion.
2. Inspect the shared pipelines loaded from `/opt/Jenkinsfiles/video-templates-ssr`
   and `/opt/Jenkinsfiles/website-front-end`. The observed producer sets version
   `1.0.${ssrVersion}-development`, changes the scope to
   `@renderforest_internal`, builds/uploads assets, and publishes `./lib/`
   using Jenkins-managed `npmrc_nexus` configuration.
3. List existing Jenkins development branch jobs and their latest build results.
   Check for queued/running builds to avoid duplicate publishes. Never include
   `main`, `master`, or `stage` in a development recovery batch.
4. Build the required producer branch:

   ```text
   https://cij.rfservs.com/job/Renderforest/job/video-templates-ssr/job/development-N/
   ```

   For the base environment, use `job/development/`. For an authorized batch
   restoration, build existing development branches one by one, waiting for
   each to finish before starting the next. Inspect failures before continuing.
5. Verify producer success and the publish log's exact package name/version.
   Where possible, confirm the expected tarball is retrievable from Nexus using
   the same registry access as the consumer. Keep credentials out of logs and
   documentation. A successful asset upload alone does not prove npm publication.
6. After restoring the required packages, recheck consumers' latest results.
   Rebuild failed development consumer branches with matching producer versions.
   For a batch, finish producer restoration first, then rebuild failed consumers
   one at a time. Do not unnecessarily rebuild already healthy consumer branches.
7. Confirm that dependency installation, image build/push, and intended local
   deployment succeed. Verify Kubernetes rollout status and the application
   endpoint where available; a successful `rollout restart` request alone is
   insufficient to establish application health.

If the producer fails with workspace ownership errors or an agent disconnect,
follow the corresponding [ownership](workspace-root-owned-artifacts.md) or
[agent/checkout](skipped-stage-agent-checkout-failure.md) runbook, then resume
the same dependency-first sequence. Do not switch the consumer to another
environment's package version as a workaround.

## Operational notes

- Development packages use fixed per-environment versions. Rebuilding republishes
  the current branch contents; it does not recover the original deleted bytes.
- Protect development versions currently referenced by consumers when designing
  Nexus retention policies. Changes to retention or pipeline configuration are
  separate work from republishing the missing packages.
- This procedure does not require workspace deletion, credential changes, or
  production deployment. Follow current task authorization for development
  rebuilds; an explicit request to restore/build them authorizes the sequence.
- Authenticated Jenkins requests via SSH to `salt` and
  `http://localhost:8080` have worked when the public API returned HTTP 403.
  The credential location is documented in the inventory; never print its values.

## Confirmed example: 2026-10-09

[website-front-end development-1 build 61](https://cij.rfservs.com/job/Renderforest/job/website-front-end/job/development-1/61/)
failed during Docker `npm install` because Nexus returned HTTP 404 for
`video-templates-ssr-1.0.1-development.tgz`. Its pipeline explicitly rewrote the
dependency to that internal tarball URL. The operator confirmed that development
packages had been removed and requested sequential producer rebuilds followed
by failed consumer rebuilds.

Sequential producer restoration completed successfully:

| Branch | Jenkins build | Published version |
| --- | --- | --- |
| `development` | 15 | `1.0.0-development` |
| `development-1` | 6 | `1.0.1-development` |
| `development-2` | 1 | `1.0.2-development` |
| `development-3` | 1 | `1.0.3-development` |
| `development-4` | 4 | `1.0.4-development` |
| `development-5` | 1 | `1.0.5-development` |
| `development-6` | 1 | `1.0.6-development` |

Each producer log confirmed publication of the matching
`@renderforest_internal/video-templates-ssr` version. Website branch 2's earlier
failure also named its missing `1.0.2-development` tarball. Other failed branches
had different causes: branch 4 had an S3 timeout, and branch 6 lacked its
`website-front-end-6` deployment in namespace `local-6`. Inspect each consumer's
failure rather than assuming that all failed development jobs have missing
packages.

Consumer recovery results after all producer builds finished:

| Website branch | Jenkins build | Result and verification |
| --- | --- | --- |
| `development-1` | 62 | Success; rollout completed, 1/1 available, `/signin` HTTP 200 |
| `development-2` | 79 | Success; rollout completed, 1/1 available, `/signin` HTTP 200 |
| `development-4` | 15 | Success; rollout completed, 1/1 available, website root and direct application service HTTP 200 |
| `development-6` | 13, then 14 | Build 13 hit an S3 connection-acquisition timeout; retry 14 installed dependencies and built/pushed the image, then failed because deployment `website-front-end-6` was absent in `local-6` |

Rebuilding packages does not create missing Kubernetes deployments. Branch 6's
remaining infrastructure issue requires restoring its intended deployment using
the established environment configuration; repeating package publication or
consumer builds cannot resolve that absence. No deployment resources were
created during this package-restoration task.
