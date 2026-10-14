# TTS Workers Infrastructure

## Scope and verification

Last inspected: **2026-10-14**, via sequential SSH connections to all five workers as `rfuser`. Inspection covered host resources, networking, installed runtime packages, systemd, Compose configuration, all 13 containers, application paths/source revisions, selected log signatures, DNS, and credential-free HTTP checks. No server configuration changes, restarts, installs, image pulls, or synthesis jobs were performed. Credentials and raw environment/configuration/log contents are excluded.

AWS access through WireGuard and production database port registration were supplied by the service owner. Tunnel endpoints, AWS routes, production database records, external CI/CD, and backup policy remain unverified. No WireGuard interface, service, or installed WireGuard package was observed on the workers; do not assume a worker terminates the tunnel.

## Servers and SSH access

Use the existing SSH aliases; all resolve to LAN addresses, user `rfuser`, port `22`. `rfuser` has Docker access. Passwordless host sudo was unavailable during inspection; protected host logs/configuration were not exhaustively inspected.

| SSH command | LAN address | DNS hostname | OS hostname | Zabbix hostname | LAN interface |
|---|---|---|---|---|---|
| `ssh ttsw-1` | 192.168.0.2 | ttsw-1.rfservs.com | `farm` | `tts-farm` | `eno0np0` |
| `ssh ttsw-2` | 192.168.0.30 | ttsw-2.rfservs.com | `ttsw-2` | `ttsw-2` | `eno1` |
| `ssh ttsw-3` | 192.168.0.29 | ttsw-3.rfservs.com | `ttsw-3` | `ttsw-3` | `eno1` |
| `ssh ttsw-4` | 192.168.0.28 | ttsw-4.rfservs.com | `ttsw-4` | `ttsw-4` | `enp1s0` |
| `ssh ttsw-5` | 192.168.0.27 | ttsw-5.rfservs.com | `ttsw-5` | `ttsw-5` | `enp3s0` |

All workers use a `/22` LAN prefix, gateway `192.168.0.1`, and DNS servers `192.168.0.3` and `192.168.0.1`. The DNS names above resolved to their listed addresses from every worker. Interfaces are configured with DHCP in `/etc/network/interfaces`, managed through enabled `networking.service`; DHCP reservations were not inspected. DNS settings are in `/etc/resolv.conf`.

## Hardware and storage differences

Memory figures below are usable RAM reported by the OS; CPU counts describe exposed logical CPUs. Disk capacities are approximate binary sizes.

| Worker | CPU | Logical CPUs | Usable RAM | GPUs | Root storage | Swap |
|---|---|---|---|---|---|---|
| ttsw-1 | 2 × Xeon Gold 6130, 16 cores/socket | 64 | 172 GiB | 5 × RTX 3080 Ti, 12 GiB each | 954 GiB NVMe; `/dev/nvme0n1p2`, 904 GiB partition | 49.2 GiB partition |
| ttsw-2 | Xeon W-2155, 10 cores | 20 | 62 GiB | GPU 0: RTX 3080, **10 GiB**; GPU 1: RTX 3080 Ti, 12 GiB | 954 GiB NVMe; `/dev/nvme0n1p2`, 904 GiB partition | 49.2 GiB partition + 16 GiB `/opt/swap` |
| ttsw-3 | Xeon W-2155, 10 cores | 20 | 62 GiB | 2 × RTX 3080 Ti, 12 GiB each | 3.5 TiB SATA SSD; `/dev/sda2`, 3.4 TiB partition | 63.7 GiB partition |
| ttsw-4 | Core i9-7920X, 12 cores, 1 thread/core exposed | 12 | 62 GiB | 2 × RTX 3080 Ti, 12 GiB each | 954 GiB SATA SSD; `/dev/sdb2`, 904 GiB partition | 49.2 GiB partition |
| ttsw-5 | Core i9-7920X, 12 cores, 2 threads/core | 24 | 62 GiB | 2 × RTX 3080 Ti, 12 GiB each | 477 GiB NVMe; `/dev/nvme0n1p2`, 451 GiB partition | 24.6 GiB partition + **128 GiB `/opt/swap`** |

All root filesystems are ext4 with a separate EFI partition. Swap files on workers 2 and 5 are configured in `/etc/fstab`. Worker 4 additionally has a 1.8 TiB SATA SSD, `/dev/sda1`, formatted XFS but not mounted or listed in fstab at inspection. Its intended use is unknown. Worker 5's large swap file shares the smaller root disk with Docker and must be considered in capacity planning.

## Host software and services

All hosts run Debian 13 (trixie), x86_64. Versions are inspection baselines, not update targets.

| Worker | Debian point release | Running kernel | Docker CE | Compose plugin | containerd.io | NVIDIA driver | NVIDIA Container Toolkit | Docker storage driver |
|---|---|---|---|---|---|---|---|---|
| ttsw-1 | 13.5 | 6.12.90+deb13-amd64 | 29.5.2 | 5.1.4 | 2.2.4 | 595.71.05 | 1.19.1 | overlayfs |
| ttsw-2 | 13.6 | 6.12.107+deb13-amd64 | 29.8.0 | 5.5.1 | 2.3.4 | 595.99.02 | 1.20.0 | overlayfs |
| ttsw-3 | 13.6 | 6.12.101+deb13-amd64 | 29.7.2 | 5.5.0 | 2.3.3 | 595.91.07 | 1.19.1 | overlayfs |
| ttsw-4 | 13.7 | 6.12.111+deb13-amd64 | 29.9.0 | 5.6.0 | 2.4.1 | 595.104.02 | 1.20.1 | overlay2 |
| ttsw-5 | 13.6 | 6.12.107+deb13-amd64 | 29.7.2 | 5.5.0 | 2.3.4 | 595.99.02 | 1.20.0 | overlayfs |

- Enabled host services: `docker.service`, `containerd.service`, `ssh.service`, `zabbix-agent2.service`, `networking.service`. Unit definitions for the first four are in `/usr/lib/systemd/system/`. No dedicated host TTS systemd unit was found; TTS processes run inside Docker.
- Common supporting services include cron, journald, and systemd-timesyncd. No TTS timer or `rfuser` crontab was found.
- Host Python is 3.13.5; Git is 2.47.3; OpenSSH server is 10.0p1. Docker Buildx and NVIDIA Container Toolkit packages are installed.
- Docker data root: `/var/lib/docker`. `/etc/docker/daemon.json` and `/etc/docker/certs.d` were absent on all workers.
- NVIDIA runtime configuration: `/etc/nvidia-container-runtime/config.toml`; CDI refresh environment file: `/etc/nvidia-container-toolkit/nvidia-cdi-refresh.env`.
- NVIDIA `.run` installers are present in `/opt` on workers 1, 2, 4, and 5. Installer presence alone does not establish the historical installation procedure.

## Compose deployment and container mapping

On **every worker**, Compose configuration is `/home/rfuser/ttsw.yaml`, project `rfuser`, working directory `/home/rfuser`. Live Compose labels, port bindings, GPU assignments, and memory limits matched this file during inspection. Use the Compose plugin syntax `docker compose -f /home/rfuser/ttsw.yaml ...`.

The configured image is `rfhub.rfservs.com:8443/tts-worker:latest`, from the owner-identified local Nexus registry. `rfhub.rfservs.com` resolves to `192.168.0.44` on all workers. No local image-build or external deployment pipeline was identified; the authoritative build/release process remains unknown.

Each container has one explicit NVIDIA GPU reservation (`driver: nvidia`, capability `gpu`, `device_ids` below), restart policy `unless-stopped`, 64 MiB shared memory, and no configured init or Docker health check. Runtime is `runc` with NVIDIA device requests. Containers run as root and have **no bind mounts or Docker volumes**.

| Worker | Container / Compose service | Host GPU index | Host SSH → container 22/tcp | Host HTTP → container 8011/tcp | RAM limit |
|---|---|---|---|---|---|
| ttsw-1 | `tts-0` | 0 | 2220 | 8220 | 32 GiB |
| ttsw-1 | `tts-1` | 1 | 2221 | 8221 | 32 GiB |
| ttsw-1 | `tts-2` | 2 | 2222 | 8222 | 32 GiB |
| ttsw-1 | `tts-3` | 3 | 2223 | 8223 | 32 GiB |
| ttsw-1 | `tts-4` | 4 | 2224 | 8224 | 32 GiB |
| ttsw-2 | `tts-6` | 0 | 2226 | 8226 | 26 GiB |
| ttsw-2 | `tts-7` | 1 | 2227 | 8227 | 26 GiB |
| ttsw-3 | `tts-8` | 0 | 2228 | 8228 | 26 GiB |
| ttsw-3 | `tts-9` | 1 | 2229 | 8229 | 26 GiB |
| ttsw-4 | `tts-10` | 0 | 2230 | 8230 | 26 GiB |
| ttsw-4 | `tts-11` | 1 | 2231 | 8231 | 26 GiB |
| ttsw-5 | `tts-12` | 0 | 2232 | 8232 | 26 GiB |
| ttsw-5 | `tts-13` | 1 | 2233 | 8233 | 26 GiB |

There are **13 TTS containers and 13 GPUs across five hosts**; `tts-5` is absent from both live containers and inspected Compose files. The reason for the numbering gap is unknown. Container number is a fleet identifier, not necessarily its host GPU index.

Published ports listen on all host IPv4 and IPv6 addresses. Host SSH uses 22/tcp; Zabbix uses 10050/tcp. No reverse proxy was observed on these hosts. Firewall restrictions and AWS end-to-end reachability were not verified.

Each host has Compose bridge network `rfuser_default`, subnet `172.18.0.0/16`, gateway `172.18.0.1`, plus default Docker bridge `172.17.0.0/16`. These are separate per-host networks; do not use container bridge addresses for AWS routing. Correlate the table with production database registrations before diagnosing a specific worker; database schema and entries remain undocumented.

## Application layout and startup

The TTS HTTP application is **ComfyUI 0.37.4** with custom TTS/audio nodes, not a separate host daemon. The following layout was confirmed across all 13 containers:

| Purpose | Path inside container |
|---|---|
| Startup script | `/workspace/start/start.sh` |
| Workspace / Docker working directory | `/workspace/comfyui` |
| ComfyUI source, `main.py`, `requirements.txt`, `pyproject.toml` | `/workspace/comfyui/ComfyUI` |
| Primary Python environment | `/workspace/comfyui/venv` |
| Separate Seed-VC Python environment | `/workspace/seedvc-venv`; Python at `bin/python` |
| Custom nodes | `/workspace/comfyui/ComfyUI/custom_nodes` |
| Models | `/workspace/comfyui/ComfyUI/models` |
| OmniVoice models | `/workspace/comfyui/ComfyUI/models/omnivoice/OmniVoice` |
| Audio separator models | `/workspace/comfyui/ComfyUI/models/audio-separator` |
| Seed-VC Hugging Face cache (`SEEDVC_HF_CACHE`) | `/root/checkpoints` |
| Inputs / generated outputs | `/workspace/comfyui/ComfyUI/input`, `/workspace/comfyui/ComfyUI/output` |
| User state / local SQLite database | `/workspace/comfyui/ComfyUI/user`; `comfyui.db` |
| Comfy CLI configuration | `/root/.config/comfy-cli/config.ini`; recent workspace is `/workspace/comfyui/ComfyUI` |
| Optional model-path configuration | `/workspace/comfyui/ComfyUI/extra_model_paths.yaml.example` exists; active `extra_model_paths.yaml` was absent |

Container launch configuration references `start.sh`. The script starts SSH with `service ssh start`, cleans the ComfyUI output path, and launches `comfy launch` in the background on `0.0.0.0:8011`, redirecting logs to `comfy.log`. PID 1 is `sleep`; the active server uses `/workspace/comfyui/venv/bin/python`. No supervisor was found. Startup script and inspected main/requirements files had identical hashes across all containers.

Application source, models, caches, logs, and local database live in the container filesystem. Changes survive a stop/start of the same container but are not preserved by mounts when it is replaced. The startup output cleanup also means outputs should not be assumed to survive restart. External persistence/backup arrangements are unverified.

## Source repositories and application differences

Repositories below have Git metadata inside each container. Revisions are the 2026-10-14 inspection baseline.

| Component under ComfyUI | Repository | Workers 1, 2, 3, 5 | Worker 4 |
|---|---|---|---|---|
| ComfyUI core | `github.com/comfyanonymous/ComfyUI` | `8ff6dc384ba5` | Same |
| `custom_nodes/ComfyUI-OmniVoice-TTS` | `github.com/renderforest/ComfyUI-OmniVoice-TTS` | `26e08fbde791` | Same |
| `custom_nodes/comfyui-audio-nodes` | `github.com/renderforest/comfyui-audio-nodes` | `82bc3cac611e` | Same |
| `custom_nodes/ComfyUI-VideoHelperSuite` | `github.com/kosinkadink/ComfyUI-VideoHelperSuite` | `4d907bee61e9` | Same |
| `custom_nodes/comfyui-manager` | `github.com/ltdrdata/ComfyUI-Manager` | `9c29dc68a488` | `45c3fb68373e` |

Only the tracked output placeholder `output/_output_images_will_be_put_here` differed from the core checkout; no tracked changes were found in the custom-node repositories. Untracked application content was not exhaustively compared.

The audio-node repository contains `requirements.txt`, `requirements-seedvc.txt`, `seedvc_convert.py`, and `seedvc_server.py`, useful starting points for Seed-VC troubleshooting. OmniVoice has its own `requirements.txt`, `pyproject.toml`, and compatibility matrix. FFmpeg is installed at `/usr/bin/ffmpeg` inside containers.

| Runtime/package in main Python environment | Workers 1, 2, 3, 5 | Worker 4 |
|---|---|---|
| Python (also Seed-VC interpreter) | 3.13.15 | 3.13.16 |
| PyTorch reported by HTTP API | 2.14.0+cu130 | 2.14.1+cu130 |
| torchvision / torchaudio | 0.29.0 / 2.11.0 | 0.29.1 / 2.11.0 |
| comfy-cli | 1.21.0 | 1.22.0 |
| transformers / huggingface-hub | 5.17.0 / 1.33.0 | 5.19.0 / 2.2.0 |
| onnxruntime-gpu | 1.30.0 | 1.31.0 |
| audio-separator / soundfile / librosa | 0.47.0 / 0.14.0 / 1.0.0 | Same |

Despite identical `latest` references in Compose, running images differ:

- Workers 1, 2, 3, 5: image ID `sha256:1d376517eed1a45c10a0df14163cff0f3490c12c59863f8bc90a7cc028357374`. Workers 1–3 also record that digest under `rfhub.rfservs.com:8443/tts-worker`; worker 5 has no recorded `RepoDigests`.
- Worker 4: image ID `sha256:473fefc35ba5bde3504e9b329c337c72ae83449374e62a15b89dfcd602701a3c`; recorded repository digest `sha256:eaeef9ff07abc0a213cc8fa77fa21bce47f01587812d4311da69758d7e81ab4b`.

Compare running image IDs/digests and package versions when investigating behavior differences. A fresh pull of mutable `latest` is not guaranteed to reproduce the currently running application.

## Logs and monitoring

- Container launch/server output: `/workspace/comfyui/comfy.log`.
- ComfyUI per-port logs: `/workspace/comfyui/ComfyUI/user/comfyui_8011.log`, with `.prev.log` and `.prev2.log` siblings.
- Docker logging: `json-file`, with no configured rotation options on the inspected containers. Engine log files are under `/var/lib/docker/containers/<container-id>/<container-id>-json.log`; application file logs are separate.
- Host service logs: journald for `docker.service`, `containerd.service`, `ssh.service`, and `networking.service`.
- Monitoring: Zabbix Agent 2 **7.2.4** (Debian 12-labelled package on Debian 13), GPU plugin **7.4.14** (Debian 13-labelled package). Compatibility was not evaluated.
- Zabbix configuration: `/etc/zabbix/zabbix_agent2.conf`; includes `/etc/zabbix/zabbix_agent2.d/*.conf` and `plugins.d/*.conf`. Log path: `/var/log/zabbix/zabbix_agent2.log`; host logrotate rule: `/etc/logrotate.d/zabbix-agent2`.
- Passive server: `192.168.0.44`, agent port `10050`; active checks target `192.168.0.44:10051`. Host identities are in the access table.
- GPU check: `/etc/zabbix/zabbix_agent2.d/gpu.conf` defines `gpu.health`, using `/usr/local/bin/check_nvidia_gpu_health.sh`; the script references `nvidia-smi`. Plugin configuration lives in `/etc/zabbix/zabbix_agent2.d/plugins.d/`.

## Operational caveats and verification limits

- **Container liveness is insufficient:** background ComfyUI plus PID 1 `sleep` can leave Docker reporting running after the application exits. No Docker health check is configured. Use `GET /system_stats` on the published HTTP port and inspect processes/logs. Zombie `comfy`/SSH processes were observed, consistent with a need to review process reaping; causation and growth rate were not established.
- **Registry TLS client difference:** normal certificate-verifying curl requests to `https://rfhub.rfservs.com:8443/v2/` returned expected unauthenticated **401** on all workers. Host Python 3.13 HTTPS verification rejected the CA with `CA cert does not include key usage extension`. The local CA file is `/usr/local/share/ca-certificates/rfCA.crt` on every worker. Do not assume Python-based registry automation will work solely because curl works; Docker pull/authentication was not tested.
- **Fleet differences:** mixed GPU VRAM on worker 2, differing CPU thread counts on workers 4/5, host runtime version drift, a different application image on worker 4, and a large swap allocation on worker 5 affect capacity and reproducibility.
- All 13 `GET /system_stats` requests returned HTTP 200 and reported ComfyUI 0.37.4; no failed systemd units were reported at inspection. This verifies HTTP/API availability only, not synthesis quality, all model/dependency paths, or AWS/database integration.

## Read-only diagnostic entry points

Run on the affected worker; substitute the container and host HTTP port from the mapping table. Avoid dumping complete Docker inspect/config/environment output because it may contain credentials in future deployments.

```bash
docker compose -f /home/rfuser/ttsw.yaml config --services
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
docker inspect --format '{{.Image}} {{json .HostConfig.DeviceRequests}} {{json .HostConfig.PortBindings}}' tts-0
nvidia-smi --query-gpu=index,name,memory.total,driver_version --format=csv
curl --fail --silent --show-error --max-time 10 http://127.0.0.1:8220/system_stats
docker exec tts-0 sh -c 'ps -eo pid,ppid,stat,comm'
df -hT /
```

Review relevant logs with credential redaction before sharing them.
