# Jenkins Infrastructure

## Purpose

This document describes the current Jenkins architecture used by Renderforest and provides infrastructure context for the AiDevOps agent.

The goal is to help the agent understand where Jenkins pipeline logic is stored, how it is distributed to worker nodes, how repositories are connected to Jenkins, and what infrastructure checks should be performed when diagnosing Jenkins failures.

---

## Jenkins Controller

The Jenkins controller is hosted in the cloud.

- URL: `https://cij.rfservs.com`
- Public IP: `138.201.158.8`
- SSH hostname in config file: `salt`

To connect to jenkins master node, use `ssh salt` command, as the .ssh/config file contains connection info.
The ssh user are password free for sudo command.

The controller manages the Jenkins organization, jobs, pipelines, credentials, and worker nodes.

---

## Jenkins Service Credentials

Credentials for accessing the Jenkins service are stored at:

```text
/home/aiagent/credsandmore/.config/secrets/jenkins.env
```

The AiDevOps agent may source this file as environment variables and use them to authenticate to Jenkins when troubleshooting jobs. Run the following in a Bash shell on the host where the file resides:

```bash
set +x
set -a
source /home/aiagent/credsandmore/.config/secrets/jenkins.env
set +a
```

Use the variables defined by the file for authenticated Jenkins requests in the same shell session. Keep shell tracing disabled while handling credentials. Never print the file contents or credential values, include them in logs, or copy them into documentation or Git.

---

## Jenkins Worker Nodes

There are five Jenkins worker nodes in the local office network.

| Jenkins Node | IP Address |
|---|---|
| `jnk-node-5` | `192.168.0.32` |
| `jnk-node-4` | `192.168.0.33` |
| `jnk-node-3` | `192.168.0.34` |
| `jnk-node-2` | `192.168.0.35` |
| `jnk-node` / `jnk` | `192.168.0.44` |

The name `jnk` may also refer to the Jenkins node named `jnk-node`.

---

## Jenkins Organization

Jenkins contains an organization named:

```text
Renderforest
```

The organization is connected to the company GitHub account/repositories.

Each Renderforest repository has its own Jenkinsfile.

The Jenkinsfile stored inside each application repository is intentionally small and loads the real pipeline definition from the Jenkins node filesystem.

Example:

```groovy
node {
    load '/opt/Jenkinsfiles/current_repo_name'
}
```

This means the application repository does not necessarily contain the complete Jenkins pipeline implementation.

When troubleshooting pipeline behavior, always check the pipeline file loaded from `/opt/Jenkinsfiles/` as well as the Jenkinsfile stored in the application repository.

---

## Central Jenkins Pipeline Repository

The shared Jenkins pipeline definitions are stored in the Git repository:

```text
sysadmin-utils
```

This repository is cloned on every Jenkins worker node.

Repository location:

```text
/var/lib/jenkins/sysadmin-utils
```

The repository contains a directory named:

```text
Jenkinsfiles
```

This directory contains the real Jenkins pipeline definitions used by the Renderforest repositories.

---

## `/opt/Jenkinsfiles` Symlink

The Jenkins pipeline directory from `sysadmin-utils` is made available through `/opt/Jenkinsfiles`.

The intended relationship is:

```text
/opt/Jenkinsfiles -> /var/lib/jenkins/sysadmin-utils/Jenkinsfiles
```

This allows application repositories to load their centralized pipeline definition using paths such as:

```groovy
load '/opt/Jenkinsfiles/current_repo_name'
```

Useful verification commands on a Jenkins node:

```bash
ls -ld /opt/Jenkinsfiles
readlink -f /opt/Jenkinsfiles
ls -la /var/lib/jenkins/sysadmin-utils/Jenkinsfiles
```

Expected resolved path:

```text
/var/lib/jenkins/sysadmin-utils/Jenkinsfiles
```

---

## Updating Jenkins Pipeline Definitions

When Jenkins pipeline code is changed in the `sysadmin-utils` Git repository, every Jenkins node must receive the updated repository contents.

A Jenkins job named:

```text
sysadmin-utils-pull
```

is used for this purpose.

The job performs the equivalent of a Git pull/update on the Jenkins nodes so that the local copy of `sysadmin-utils` contains the latest pipeline definitions.

Therefore, after changing a Jenkins pipeline in `sysadmin-utils`, make sure that `sysadmin-utils-pull` has completed successfully for all relevant nodes.

---

## SSH Access to Jenkins Nodes

SSH access to Jenkins nodes currently uses the personal SSH credential:

```text
gevorg.hakobyan.personal
```

SSH username:

```text
rfuser
```

Typical connection pattern:

```bash
ssh rfuser@192.168.0.X
```

The corresponding private SSH key is managed through Jenkins credentials and should not be copied into documentation or logs.

Never print, expose, or store private key contents in troubleshooting output.

---

## Troubleshooting Model

When a Jenkins pipeline fails, do not assume that the problem is only inside the application repository.

The execution path is approximately:

```text
GitHub repository
    |
    v
Repository Jenkinsfile
    |
    v
load /opt/Jenkinsfiles/<pipeline>
    |
    v
/var/lib/jenkins/sysadmin-utils/Jenkinsfiles/<pipeline>
    |
    v
Pipeline executes on selected Jenkins worker node
```

Because of this architecture, failures can come from several layers.

### 1. Application repository

Check:

- repository source code
- package files
- lockfiles
- Dockerfiles
- environment configuration
- repository Jenkinsfile

### 2. Central pipeline definition

Check the actual pipeline loaded from:

```text
/opt/Jenkinsfiles/
```

and its source under:

```text
/var/lib/jenkins/sysadmin-utils/Jenkinsfiles/
```

### 3. `sysadmin-utils` synchronization

If a pipeline works on one node but behaves differently on another, check whether the affected node has an outdated checkout.

Useful commands:

```bash
cd /var/lib/jenkins/sysadmin-utils
git status
git branch --show-current
git log -1 --oneline
git remote -v
```

Compare the latest commit between nodes when necessary.

If the node is outdated, investigate the `sysadmin-utils-pull` job or perform the approved repository update procedure.

### 4. `/opt/Jenkinsfiles` symlink

Verify that the symlink exists and resolves to the expected directory:

```bash
ls -ld /opt/Jenkinsfiles
readlink -f /opt/Jenkinsfiles
```

A missing or incorrect symlink can cause Jenkins to load the wrong pipeline or fail to load a pipeline at all.

### 5. Node-specific configuration

If a failure occurs only on one Jenkins worker, compare that node with a working node.

Check differences such as:

- installed packages
- Node.js version
- npm/pnpm versions
- Docker version
- Docker images
- filesystem permissions
- Jenkins workspace state
- environment variables
- `/root/.npmrc`
- Jenkins-provided `.npmrc`
- credentials
- cached dependencies
- available disk space
- running services
- network access

### 6. Jenkins credentials

If the failure involves GitHub, npm, Docker registries, SSH, APIs, or another authenticated resource, verify that the expected Jenkins credential is actually injected into the pipeline.

Do not expose secrets while debugging.

Avoid commands such as:

```bash
cat ~/.npmrc
cat /root/.npmrc
cat ~/.ssh/id_rsa
printenv
```

when they could expose credentials.

Prefer checks that confirm access without printing the secret itself.

---

## Known Troubleshooting Example: Private npm Package

One observed Jenkins failure occurred during:

```bash
pnpm install
```

The error was similar to:

```text
ERR_PNPM_TARBALL_HTTP_STATUS
Tarball server returned HTTP 404
```

for a private package such as:

```text
@renderforest/models-definitions-main
```

In this environment, a private npm package returning HTTP 404 can indicate that npm authentication is missing from the Jenkins build environment.

A working troubleshooting direction is to verify that the correct managed npm configuration is injected into the pipeline.

Example:

```groovy
withNPM(npmrcConfig: 'npmrc_local_pnpm') {
    sh '''
        npm install -g pnpm && \
        export PATH="$(npm config get prefix)/bin:$PATH" && \
        pnpm install && \
        pnpm run swagger
    '''
}
```

When a similar private-package 404 occurs in the future, authentication and `.npmrc` handling should be among the first checks.

---

## Guidance for the AiDevOps Agent

When diagnosing Jenkins incidents in this environment:

1. Identify which Jenkins node executed the failed stage.
2. Read the exact Jenkins error before changing anything.
3. Determine whether the failure is application-specific, pipeline-specific, credential-related, or node-specific.
4. Remember that the real pipeline code usually comes from `/opt/Jenkinsfiles`, not only from the repository Jenkinsfile.
5. Verify the affected node has the latest `sysadmin-utils` version.
6. Compare a failing node with a working node when behavior differs between workers.
7. Check credentials and local configuration when failures involve authenticated services.
8. Prefer diagnostic commands before making changes.
9. Do not expose credentials, tokens, SSH private keys, or other secrets in Jenkins logs.
10. After identifying a confirmed fix, document the incident and solution so the same troubleshooting pattern can be reused later.

---

## Important Paths and Names

```text
Jenkins controller:
https://cij.rfservs.com
138.201.158.8

Jenkins organization:
Renderforest

Central pipeline repository:
/var/lib/jenkins/sysadmin-utils

Pipeline source directory:
/var/lib/jenkins/sysadmin-utils/Jenkinsfiles

Pipeline load path:
/opt/Jenkinsfiles

Pipeline update job:
sysadmin-utils-pull

SSH username:
rfuser

Jenkins SSH credential:
gevorg.hakobyan.personal
```

---

## Maintenance Rule

Update this file whenever the Jenkins architecture changes, including:

- adding or removing Jenkins nodes
- changing node IP addresses
- changing the Jenkins controller
- changing SSH users or Jenkins credential names
- changing the location of `sysadmin-utils`
- changing the pipeline synchronization method
- changing the `/opt/Jenkinsfiles` mapping
- introducing new shared Jenkins pipeline architecture

This file is infrastructure knowledge for the AiDevOps agent and should reflect the current production Jenkins environment.
