# Jenkins checkout fails on root-owned build artifacts

## Symptom and cause

Checkout fails before the application build with errors such as:

```text
ERROR: Failed to clean the workspace
Unable to delete '/var/lib/jenkins/workspace/<workspace>'
java.nio.file.FileSystemException: .../dist/...: Operation not permitted
hudson.plugins.git.GitException: Failed to delete workspace
ERROR: Maximum checkout retry attempts reached, aborting
```

This is a recurring issue across repositories. Docker build stages running as
root can leave root-owned directories in the mounted Jenkins workspace. Jenkins
cannot remove their contents or change their permissions during the next checkout.
Possible affected directories include `dist`, `dist-cdn`, or other generated
output; use the paths in the actual error rather than assuming the directory name.

An ownership mismatch must be confirmed before applying this fix. Similar errors
can also come from immutable attributes, ACLs, or a read-only filesystem. Rebuilding
alone does not resolve a persistent ownership mismatch.

## Diagnose

1. Read [Jenkins infrastructure](../../inventory/jenkins-infrastructure.md).
   Identify the worker and exact workspace from the failed checkout stage.
   Different stages can execute on different workers. Use the configured SSH
   alias for the affected worker.
2. Confirm the host, Jenkins agent identity, and ownership of the workspace and
   error paths. Run these commands on that worker, replacing the example paths:

   ```bash
   hostname
   id jenkins
   sudo stat -c '%U:%G %u:%g %a %n' \
     /var/lib/jenkins/workspace/<workspace> \
     /var/lib/jenkins/workspace/<workspace>/<generated-directory>
   sudo find /var/lib/jenkins/workspace/<workspace>/<generated-directory> \
     -maxdepth 2 -printf '%u:%g %m %p\n'
   sudo lsattr -d /var/lib/jenkins/workspace/<workspace>/<generated-directory>
   ```

   Verify the actual agent user if it differs from `jenkins`. A root-owned
   directory with mode `755` prevents Jenkins from deleting entries inside it.
   Root-owned files alone do not necessarily prevent deletion: directory
   permissions matter.
3. Inspect the repository Jenkinsfile and the pipeline loaded from
   `/opt/Jenkinsfiles/<repository>`. Look for Docker `args '-u root:root ...'`
   and workspace mounts. This can override the Jenkins UID/GID shown earlier
   in the `docker run` command.
4. Confirm no build or container is currently writing to this workspace before
   changing ownership. Check Jenkins running builds/queue and worker containers:

   ```bash
   sudo docker ps --format '{{.ID}} {{.Image}} {{.Names}}'
   ```

   If needed, inspect a relevant container's mount paths without dumping its
   environment or credentials:

   ```bash
   sudo docker inspect --format '{{json .Mounts}}' <container-id>
   ```

## Repair and rebuild

Restore ownership only on the confirmed generated directory. For an agent
running as `jenkins:jenkins`:

```bash
sudo chown -R --no-dereference jenkins:jenkins \
  /var/lib/jenkins/workspace/<workspace>/<generated-directory>
sudo stat -c '%U:%G %a %n' \
  /var/lib/jenkins/workspace/<workspace>/<generated-directory>
sudo -u jenkins test -w \
  /var/lib/jenkins/workspace/<workspace>/<generated-directory>
sudo -u jenkins test -x \
  /var/lib/jenkins/workspace/<workspace>/<generated-directory>
```

Use the verified agent owner/group if different. Ensure the target is a real
directory inside the affected workspace, not a symlink or a shared cache mount.
Do not recursively change ownership of all Jenkins workspaces, shared dependency
caches, SSH directories, or credential files. Do not use `chmod 777`.

Check whether a later build has already recovered before rebuilding. Confirm the
branch, current source revision, and publish/deployment targets; follow repository
approval rules for production changes. Trigger one build and monitor it through
completion.

Verify:

- Checkout succeeds on the previously affected worker.
- Build and intended package publishing/deployment finish successfully.
- The application or published asset works, where applicable. A green build
  alone does not establish application health.
- Newly generated directories remain removable by the agent. If Docker still
  produces root-owned artifacts, restore ownership after the build has stopped
  writing, and report that this recovery is temporary.

Do not delete the workspace as a first response. Repository instructions require
explicit confirmation for file/directory deletion. An agent or controller restart
does not repair filesystem ownership.

## Prevent recurrence

Prefer running build commands with the agent UID/GID when the image and tooling
support it. Review global package installation, cache permissions, and required
mounts before changing the container user.

If root is required, add an ownership-restoration step inside the root container
after artifact creation. It should run on both success and failure, use the
verified host agent UID/GID, and affect only generated output in that workspace.
Keep shared cache and credential mounts outside the ownership change. Validate
that the cleanup executes before the container exits and that Jenkins can clean
the outputs on the next checkout.

Update the pipeline source in `sysadmin-utils` and distribute it through
`sysadmin-utils-pull` using the documented workflow. Compare node checkouts:
the pipeline-loading worker may differ from the build worker. Do not leave a
worker-only pipeline edit as the permanent fix. Observe production approval
requirements if the change affects production builds.

## Confirmed incident: 2026-10-09

- [rf-components development-3 build 17](https://cij.rfservs.com/job/Renderforest/job/rf-components/job/development-3/17/)
  failed during checkout on `jnk-node-4` (`ssh jnk-4`). Builds 14–17 failed.
- Workspace:
  `/var/lib/jenkins/workspace/rest_rf-components_development-3`.
  The workspace belonged to `jenkins:jenkins`, but `dist/` belonged to
  `root:root` with mode `755`, and its generated files belonged to root.
- The pipeline explicitly used `-u root:root`. Restoring ownership of `dist/`
  allowed checkout to complete.
- [Build 18](https://cij.rfservs.com/job/Renderforest/job/rf-components/job/development-3/18/)
  succeeded with the same application commit, `1074df712ec40be79311d3ab61d925545f2a5c2a`.
  It published `@renderforest_internal/rf-components@1.0.3-development` and
  uploaded the CDN bundle to the `website-front-end-3` static directory.
- `http://www-3.local.renderforest.com/static/rf-components/index.js` returned
  HTTP 200 with JavaScript content after publication.
- The build recreated root-owned `dist/` artifacts; their ownership was restored
  again after completion. No pipeline prevention change was applied.
