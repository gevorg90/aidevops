# Build and deploy a new TTS worker image

## Scope

Use after updating `sysadmin-utils/ansible/playbook/tts-worker/DockerImage`,
for example changing image setup or adding files to `ref-sounds/`.
Build and push once, then pull and recreate the affected services on each worker.

Infrastructure, SSH aliases, GPU assignments, and ports are documented in
[TTS workers inventory](../../inventory/tts-workers.md).
This procedure was written on 2026-10-14; no build or deployment was executed.

## Before starting

- Confirm the intended source changes and build host. The builder host is not
  documented; use a host with Docker, sufficient disk space, and existing access
  to `rfhub.rfservs.com:8443`.
- Check the current Dockerfile and `.dockerignore`. The inventory's inspected
  Dockerfile copies `ref-sounds/` but does **not** execute `setup.sh`. Editing
  `setup.sh` alone therefore does not change the image's installed software
  unless the Dockerfile is changed to use it. Confirm the build uses your changes.
- Obtain explicit approval before deploying to production, following
  [repository instructions](../../AGENTS.md). Preparing this runbook is not
  authorization to execute a deployment.
- Arrange a maintenance window or drain the selected service through the
  application's established procedure before recreation. Traffic routing and
  draining commands are not documented here. Recreation interrupts active jobs;
  a longer stop timeout alone does not guarantee graceful completion because
  ComfyUI runs in the background with `sleep` as PID 1.
- Preserve any required container-local inputs, outputs, user database, caches,
  or manual changes before replacement. The inspected containers have no bind
  mounts or volumes. Their writable filesystem is lost on recreation, and startup
  also clears outputs. Keeping the old image does not back up this state.

## 1. Build and push

Run in the `sysadmin-utils` checkout on the build host. If already inside that
repository, omit the leading `sysadmin-utils/` from the `cd` command.

```bash
cd sysadmin-utils/ansible/playbook/tts-worker/DockerImage
git status --short
git rev-parse HEAD
docker image build -t rfhub.rfservs.com:8443/tts-worker:latest .
```

Stop if the build fails. After a successful build:

```bash
docker push rfhub.rfservs.com:8443/tts-worker:latest
docker image inspect --format '{{.Id}} {{json .RepoDigests}}' rfhub.rfservs.com:8443/tts-worker:latest
```

Omitting `:latest` in build/push commands uses the same tag. Record the source
commit, any uncommitted build inputs, build image ID, and the registry digest
reported by the push. Keep `latest` unchanged during the rollout so every host
receives the same image. For stronger release tracking, consider unique release
tags or digest-pinned deployment in a separately approved configuration change.

## 2. Prepare and pull on each worker

Connect sequentially using `ssh ttsw-1` through `ssh ttsw-5`. Use each host's
existing `/home/rfuser/ttsw.yaml`, project `rfuser`; do not replace it with the
repository Compose template, which has documented GPU/memory differences.

Commands below run in Bash on the worker:

```bash
cd /home/rfuser
hostname
docker compose -p rfuser -f /home/rfuser/ttsw.yaml config --quiet
docker compose -p rfuser -f /home/rfuser/ttsw.yaml config --services
docker compose -p rfuser -f /home/rfuser/ttsw.yaml config --images
docker compose -p rfuser -f /home/rfuser/ttsw.yaml ps
df -h /var/lib/docker
```

Confirm the host, service names, and configured image match the intended target.
Before pulling, record every service's running image ID and retain each distinct
old image with a local rollback tag. Example for `tts-0` on `ttsw-1`:

```bash
old_image_id=$(docker inspect --format '{{.Image}}' tts-0)
docker image tag "$old_image_id" tts-worker-rollback:before-update-tts-0
```

Use a fresh rollback tag per deployment so an earlier recovery image is not
overwritten. Record the service-to-tag mapping. Images can differ across workers
and services; do not assume one previous image covers the fleet.

Pull the updated image using either command:

```bash
docker compose -p rfuser -f /home/rfuser/ttsw.yaml pull
# Alternative when all target services use this same image:
docker image pull rfhub.rfservs.com:8443/tts-worker:latest
```

Check the pulled image before recreating anything:

```bash
docker image inspect --format '{{.Id}} {{json .RepoDigests}}' rfhub.rfservs.com:8443/tts-worker:latest
```

Require a successful pull and a repository digest matching the recorded push.
If they differ, stop and investigate. Registry authentication/TLS failures must
be resolved through the established configuration; do not disable verification.

## 3. Recreate and verify one service at a time

Start with one drained service as a canary. Example for `tts-0` on `ttsw-1`:

```bash
docker compose -p rfuser -f /home/rfuser/ttsw.yaml up -d --no-deps --force-recreate --pull never tts-0
```

`--pull never` uses the image already pulled and checked. `--no-deps` limits the
operation to the selected service. `up` normally recreates a container when its
image changes; `--force-recreate` makes the replacement explicit.

**Do not use `docker compose restart --force-recreate`.** `restart` does not
support that flag and restarts existing containers without applying a new image.

Verify the running image ID equals the pulled image ID:

```bash
docker image inspect --format '{{.Id}}' rfhub.rfservs.com:8443/tts-worker:latest
docker inspect --format '{{.Image}}' tts-0
docker compose -p rfuser -f /home/rfuser/ttsw.yaml ps tts-0
curl --fail --silent --show-error --max-time 10 http://127.0.0.1:8220/system_stats
```

Allow for model/application startup, then require a successful API response.
Check startup/application logs locally at `/workspace/comfyui/comfy.log` and
`/workspace/comfyui/ComfyUI/user/comfyui_8011.log`; redact credentials before
sharing. Docker reporting `running` is insufficient: no Docker health check is
configured and PID 1 can remain alive after ComfyUI exits.

Run a known application TTS workflow through the established client and verify
successful synthesis, audio quality, and any changed reference sounds or nodes.
Include voice conversion if affected; HTTP availability alone does not validate
Seed-VC or model inference. The production workflow/client commands are not
documented, so do not substitute an invented API request.

After the canary passes, return it to traffic and repeat drain, recreation, and
verification for each service, then proceed to the next host:

| Worker | Services | Corresponding host HTTP ports |
|---|---|---|
| `ttsw-1` | `tts-0`, `tts-1`, `tts-2`, `tts-3`, `tts-4` | 8220, 8221, 8222, 8223, 8224 |
| `ttsw-2` | `tts-6`, `tts-7` | 8226, 8227 |
| `ttsw-3` | `tts-8`, `tts-9` | 8228, 8229 |
| `ttsw-4` | `tts-10`, `tts-11` | 8230, 8231 |
| `ttsw-5` | `tts-12`, `tts-13` | 8232, 8233 |

If downtime for all services on one host is approved, the shorter host-wide
command after pulling is:

```bash
docker compose -p rfuser -f /home/rfuser/ttsw.yaml up -d --force-recreate --pull never
```

Validate every service afterward. Avoid `compose down` or image pruning during
the rollout; retain rollback images until the deployment is accepted.

## If verification fails

Stop the rollout and keep the failing service out of traffic. Inspect its image,
GPU access, API response, and application logs. If rollback is approved, create
a temporary Compose override on that host referencing its retained image:

```yaml
# /home/rfuser/ttsw-rollback.yaml; example for tts-0 only
services:
  tts-0:
    image: tts-worker-rollback:before-update-tts-0
    pull_policy: never
```

Validate the merged configuration, then recreate only that service:

```bash
docker compose -p rfuser -f /home/rfuser/ttsw.yaml -f /home/rfuser/ttsw-rollback.yaml config --quiet
docker compose -p rfuser -f /home/rfuser/ttsw.yaml -f /home/rfuser/ttsw-rollback.yaml up -d --no-deps --force-recreate --pull never tts-0
```

Verify its running image ID against the retained tag, repeat API and synthesis
checks, and restore traffic only after recovery. This restores the image, not
lost container-local state. Record the rollback override: later commands using
only `ttsw.yaml` can switch the service back to `latest`.

## Completion record

Record the source revision/build inputs, pushed digest, deployed image ID per
service, rollout time, API and synthesis results, and any rollback. Do not record
credentials. A completed rollout requires all intended services to run the
recorded new image and pass functional verification.

## Docker command references

- [Compose up: recreation, service selection, and pull options](https://docs.docker.com/reference/cli/docker/compose/up/)
- [Compose restart: supported options and restart behavior](https://docs.docker.com/reference/cli/docker/compose/restart/)
