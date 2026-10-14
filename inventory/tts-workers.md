# TTS Workers Infrastructure

## Overview

The TTS workers run on five servers in the local network. AWS communicates with these servers through a WireGuard tunnel.

Each worker uses NVIDIA GeForce RTX 3080 Ti GPU devices and runs TTS containers managed by Docker Compose. Container images are pulled from the local Nexus server.

## Servers and SSH access

The existing SSH configuration provides aliases in the form `ttsw-X`.

| Server | Local IP address | Other name | SSH command |
|---|---|---|---|
| ttsw-1 | 192.168.0.2 | tts-farm | `ssh ttsw-1` | DNS hostname: ttsw-1.rfservs.com |
| ttsw-2 | 192.168.0.30 | — | `ssh ttsw-2` | DNS hostname: ttsw-2.rfservs.com |
| ttsw-3 | 192.168.0.29 | — | `ssh ttsw-3` | DNS hostname: ttsw-3.rfservs.com |
| ttsw-4 | 192.168.0.28 | — | `ssh ttsw-4` | DNS hostname: ttsw-4.rfservs.com |
| ttsw-5 | 192.168.0.27 | — | `ssh ttsw-5` | DNS hostname: ttsw-5.rfservs.com |

## Container deployment

On each worker, the `rfuser` home directory contains the Docker Compose file `ttsw.yaml`. This file is used to start containers named `tts-1`, `tts-2`, and so on.

The number of containers and GPU devices per worker has not been specified.

## Connectivity and port mappings

Each container has a distinct SSH port and HTTP port on its worker. The production database stores the ports used to communicate with the containers.

```text
AWS
  |
  | WireGuard tunnel
  v
Local network: ttsw-1 through ttsw-5
  |
  | Per-container SSH and HTTP ports
  v
TTS containers: tts-1, tts-2, ...
```

When investigating connectivity, correlate the affected worker and container's published ports with the corresponding production database entry. Exact port mappings and database schema details have not yet been documented.

## Image source

Workers pull the TTS Docker image from the local Nexus server. The Nexus hostname, registry endpoint, image repository, and image tags have not yet been specified.

## Sources and scope

Infrastructure details were supplied by the service owner on 2026-10-14. Live servers, Compose files, database entries, and WireGuard configuration were not inspected. WireGuard peer locations, tunnel addresses, and routing details are not yet documented.
