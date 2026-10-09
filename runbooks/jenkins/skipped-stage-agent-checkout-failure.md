# Jenkins skipped stage fails during agent allocation or checkout

## Symptom

A branch-specific stage allocates an agent and checks out source even though its
`when` condition should skip the stage. An agent disconnect or workspace problem
during that checkout fails the entire pipeline before the skip is evaluated.

For example, a `main` build enters `Deploy to Development` and fails with:

```text
ERROR: Checkout failed
java.nio.channels.ClosedChannelException
hudson.remoting.RequestAbortedException
ERROR: Maximum checkout retry attempts reached, aborting
```

## Cause

By default, Declarative Pipeline evaluates a stage's `when` condition after
entering its agent. Agent setup can include automatic SCM checkout and Docker
setup. Thus a stage that will eventually be skipped can still fail during setup.

Add `beforeAgent true` directly inside the stage's `when` block to evaluate the
condition before allocating its agent:

```groovy
stage('Deploy to Development') {
    when {
        beforeAgent true
        branch 'development*'
    }
    agent {
        node {
            label 'node1'
        }
    }
    steps {
        // Existing deployment steps.
    }
}
```

For `allOf`, `anyOf`, or `not`, put the directive in the enclosing `when` block,
not inside the nested condition. Add it once per active `when` block.

Reference: [Jenkins evaluation of when before agent](https://www.jenkins.io/doc/book/pipeline/syntax/#evaluating-when-before-entering-agent-in-a-stage).

## Diagnose before cleaning the workspace

1. Read [Jenkins infrastructure](../../inventory/jenkins-infrastructure.md).
   Identify the branch, failed stage, worker, and exact exception.
2. Check whether that stage should run for this branch. Inspect both the
   application's Jenkinsfile and the loaded `/opt/Jenkinsfiles/<repository>`.
   The pipeline-loading worker may differ from the build/deployment worker.
3. For `ClosedChannelException`, correlate controller and affected agent logs
   with the build timestamps. Use an explicit UTC range or account for the
   host timezone. On the worker, where this service exists:

   ```bash
   sudo journalctl -u jenkins-agent \
     --since '<start UTC>' --until '<end UTC>' --no-pager
   ```

   On `salt`, inspect the corresponding `jenkins` service journal and
   `/var/lib/jenkins/logs/slaves/<node>/slave.log*`. Redact credentials before
   sharing logs. Check reconnection timing, process restarts, OOM events, and
   proxy/network evidence. A closed channel identifies a transport failure,
   but does not by itself establish why the connection closed.
4. Compare subsequent builds, source commits, and any cleanup timestamps.
   Success after cleanup does not prove cleanup was necessary if the agent
   reconnected first.
5. For `Operation not permitted` or confirmed root-owned output, follow
   [workspace ownership repair](workspace-root-owned-artifacts.md) instead.

## Implement and validate

For repository changes, work in `repos/sysadmin-utils`. Check the worktree and
switch to the requested test branch before editing. In the observed change,
`test-jenkinsfiles` was created from `master`.

1. Add `beforeAgent true` to each applicable active stage-level `when` block.
   Preserve branch/PR conditions and commented-out code. Expand inline blocks
   if needed for readability; avoid duplicate directives.
2. Review any conditions requiring workspace files, stage-specific environment,
   or agent steps before moving their evaluation earlier. Such conditions may
   need restructuring. The observed batch used branch/PR conditions.
3. Validate every changed file with the Jenkins Declarative Pipeline linter
   (`/pipeline-model-converter/validate`) and run `git diff --check`.
   If validation fails, compare the original file before attributing the error
   to this change. Do not execute deployment builds just to check syntax.
4. The Jenkins credentials location is documented in the inventory. Never print
   credential values. During the observed validation, authenticated requests
   through SSH to `salt` and `http://localhost:8080` worked when requests to the
   public URL returned HTTP 403.
5. After review and authorization for the affected environments, distribute
   the approved shared pipeline changes using `sysadmin-utils-pull`. Verify
   synchronization on pipeline-loading workers as well as build workers.

For the next authorized build, confirm that inapplicable stages skip without
allocating their stage agents or performing their stage checkouts. Applicable
stages must still run. Syntax validation alone does not verify runtime behavior.

## Transport recovery and limits

`beforeAgent true` removes unnecessary agent dependencies. It does not repair
WebSocket instability or protect applicable stages from disconnects.

For a transient disconnect, wait for the agent to reconnect and retry the
appropriate build or checkout. Consider a bounded
`retry(count: 2, conditions: [agent()])` around a node allocation and checkout
when restructuring the pipeline. Retrying only inside a dead node allocation
may be insufficient. Keep publish/deployment side effects outside retries
unless their repeat behavior has been reviewed.

Reference: [Jenkins agent-aware retry](https://www.jenkins.io/doc/pipeline/steps/workflow-basic-steps/#retry-retry-the-body-up-to-n-times).

The `cleanup_workplace` job invokes a privileged script that matches repository
names across workspace directories, potentially including multiple branches.
Its observed default node list also omitted `jnk-node-4`. Inspect the current
parameters and script before using it; do not assume it targets only the failed
workspace or includes the affected worker. Follow repository approval rules for
deletion and production changes, and avoid deleting active workspaces.

## Observed incident and validation: 2026-10-09

- [campaign-api main build 24](https://cij.rfservs.com/job/Renderforest/job/campaign-api/job/main/24/)
  built and pushed its image, then failed during checkout on `jnk-node-3` inside
  `Deploy to Development`, a stage that should skip on `main`.
- Yerevan time: the connection closed at 10:43:58, the agent reconnected at
  10:44:01, workspace cleanup ran at 10:45:27, and build 25 started at 10:45:32.
  Build 25 succeeded with the same source commit and issued the production
  deployment restart. Application health was not verified in this investigation.
- Other workers also disconnected that morning. The underlying transport cause
  was not established; no infrastructure changes were made during diagnosis.
- On `test-jenkinsfiles`, 423 active `when` blocks across 95 shared Jenkinsfiles
  received `beforeAgent true`. All 95 files passed Jenkins linter validation,
  and `git diff --check` passed. The edit preserved existing conditions and
  expanded three inline blocks. Deployment of that batch was not performed as
  part of the edit/validation task.
