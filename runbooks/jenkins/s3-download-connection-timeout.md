# Jenkins S3 download connection timeout

## Symptom

A pipeline fails in `s3Download` with:

```text
java.util.concurrent.TimeoutException: Acquire operation took longer than 10000 milliseconds.
software.amazon.awssdk.core.exception.SdkClientException: Unable to execute HTTP request
```

The stack trace references `pipeline-aws`, the AWS SDK Netty connection pool,
and the worker executing the download.

This means the SDK could not acquire a connection within its configured timeout.
The message alone does not prove pool exhaustion, excessive traffic, a network
outage, or invalid credentials. Gather evidence before choosing a fix.

## First recovery step: rebuild

For this specific S3 connection acquisition timeout, first try building again.
Based on the operator's experience, a retry resolves the issue in approximately
99% of cases. This is an operational estimate.

Confirm the target environment and check whether a subsequent build has already
succeeded before triggering the retry. Follow repository approval rules for
production changes. Verify the retried build and intended deployment; if the
same timeout persists, continue with the diagnostics below.

## Troubleshooting

1. Read `inventory/jenkins-infrastructure.md`. Identify the failed step, worker,
   bucket, object, region, and source commit from the build log. Never print
   credentials or credential-bearing configuration.
2. Check subsequent builds before retrying or restarting anything. Compare the
   worker, source commit, S3 download result, and deployment result. A successful
   later build on a different commit shows recovery of the download path, but
   does not validate the original commit.
3. Connect using the configured SSH alias. For `jnk-node-2`, use `ssh jnk-2`.
   A direct IP connection may not select the configured identity file.
4. Inspect the repository Jenkinsfile and the pipeline it loads from
   `/opt/Jenkinsfiles/`. Verify the symlink and central repository state:

   ```bash
   readlink -f /opt/Jenkinsfiles
   sudo -u jenkins git -C /var/lib/jenkins/sysadmin-utils status --short
   sudo -u jenkins git -C /var/lib/jenkins/sysadmin-utils log -1 --oneline
   ```

   Run Git as the checkout owner; do not change global Git trust settings to
   bypass an ownership warning.
5. Test DNS and HTTPS from the affected worker using the actual bucket and
   region. For the observed backend download:

   ```bash
   getent ahostsv4 ssh-deployed-keys.s3.us-west-2.amazonaws.com
   curl -sS -o /dev/null --connect-timeout 5 --max-time 15 \
     -w 'S3 HTTP %{http_code} connect %{time_connect} TLS %{time_appconnect} total %{time_total}\n' \
     https://ssh-deployed-keys.s3.us-west-2.amazonaws.com/docdb-global-bundle.pem
   ```

   An unauthenticated HTTP 403 confirms DNS/TCP/TLS/HTTP reachability at the
   time of the check. It does not confirm authenticated object access or SDK
   connection-pool health. A successful Jenkins download is stronger evidence.
6. If failures recur, compare affected workers, concurrent AWS operations,
   agent logs, resource usage, and installed `pipeline-aws`/AWS SDK versions.
   Record timing and frequency before attributing the failure to a specific
   plugin or connection-pool defect.

## Recovery and verification

- If a subsequent build already downloads the object successfully, avoid an
  unnecessary agent/controller restart or duplicate deployment.
- Before triggering a rebuild, inspect its deployment stages and confirm the
  target environment. Follow repository approval rules for production changes.
- For recurring failures, consider a bounded retry with a delay around only
  the S3 download, after reviewing the shared pipeline and its environments.
  Validate the change and synchronize it using `sysadmin-utils-pull` according
  to the inventory. Do not blindly increase connection limits or timeouts.
- Verify the download, image build/push, and intended deployment independently.
  `kubectl rollout restart` confirms that a restart was requested; check rollout
  status and application health before reporting application recovery.

## Observed incident: 2026-10-08

- [Backend stage build 22](https://cij.rfservs.com/job/Renderforest/job/backend/job/stage/22/)
  failed on `jnk-node-2` downloading
  `s3://ssh-deployed-keys/docdb-global-bundle.pem` in `us-west-2`.
- The SDK connection acquisition timed out before Docker build commands ran;
  deployment stages were skipped.
- [Build 23](https://cij.rfservs.com/job/Renderforest/job/backend/job/stage/23/)
  succeeded on the same worker with a newer application commit and no repository
  Jenkinsfile changes. The S3 download completed and the stage backend deployment
  restart command succeeded.
- Subsequent unauthenticated HTTPS probing from the worker returned HTTP 403 in
  under one second. The shared pipeline checkout was clean and its symlink
  resolved to the documented location.
- Evidence was consistent with a transient SDK connection issue; the underlying
  cause was not established. No infrastructure or pipeline changes were made,
  and application health after deployment was not verified.
