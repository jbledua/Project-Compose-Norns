# Project-Compose-Norns

Deploys ComfyUI with an NVIDIA GPU–enabled container and a Tailscale sidecar for private, secure access. Includes persistent user data and custom nodes, and supports a shared models directory (local or NAS-mounted) plus optional shared input/output folders to avoid duplicating weights.

## Local-Only Test (No Tailscale)
This repo is currently configured to run ComfyUI locally without Tailscale for quick testing. The service binds to `127.0.0.1:8188` by default.

Quick start:

```bash
# From the repo root
mkdir -p comfyui/user comfyui/custom_nodes models input output

# Optional: change port or bind address
# export COMFYUI_PORT=8288
# export COMFYUI_HOST_BIND=0.0.0.0   # expose on LAN (be cautious)

docker compose pull
docker compose up -d norns-app
```

Open http://localhost:8188 in your browser.

Mounts used with safe defaults:
- `./comfyui/user -> /comfyui/user`
- `./comfyui/custom_nodes -> /comfyui/custom_nodes`
- `./models -> /comfyui/models`
- `./input -> /comfyui/input`
- `./output -> /comfyui/output`

To re-enable Tailscale later, restore the sidecar service and `network_mode: service:norns-ts` in `docker-compose.yml`.

## Why “Norns”?
After The Norns (nornir) — three powerful, fate‑weaving goddess‑like beings in Norse mythology. The name nods to the feeling that AI can seem like magic: threads of inputs and models woven into outcomes.

## Overview
This compose setup runs two services:
- `norns-ts`: a Tailscale sidecar that provides private networking for the app.
- `norns-app`: ComfyUI (GPU-enabled) sharing the network stack with the sidecar for secure, tailnet-only access.

By using `network_mode: service:norns-ts`, ComfyUI is reachable on your tailnet without exposing ports to the public internet.

## Prerequisites
- Docker Engine and Docker Compose v2 on Linux
- NVIDIA GPU + drivers installed on the host
- NVIDIA Container Toolkit installed (required for `gpus: all`)
- A Tailscale account and an auth key (or use interactive login)

## Quick Start
1. Create (or confirm) required host directories for bind mounts:
   - A host directory for models (e.g. `/data/models`)
   - Optional: input/output host directories
2. Copy `.env.example` to `.env` and edit values, or create a `.env` file next to `docker-compose.yml`:
   ```env
   TS_HOSTNAME=norns
   TS_AUTHKEY=tskey-xxxxxxxxxxxxxxxxxxxxxxxx
   STACK_NAME=norns
   MODELS_DIR=/data/models
   INPUT_DIR=/data/comfyui-input
   OUTPUT_DIR=/data/comfyui-output
   # Optional: publish ComfyUI on the LAN
   COMFYUI_HOST_BIND=0.0.0.0
   COMFYUI_PORT=8188
   ```
3. Start the stack:
   ```bash
   docker compose up -d
   ```
4. Access ComfyUI over your tailnet (default port 8188):
   - Visit `http://<TS_HOSTNAME>:8188` or use the device’s Tailscale IP/MagicDNS name.
   - If LAN publishing is enabled, also reachable at `http://<host-ip>:${COMFYUI_PORT}`.

## Data & Volumes
- ComfyUI user data: named volume `${STACK_NAME}-user -> /comfyui/user`
- Custom nodes: named volume `${STACK_NAME}-custom-nodes -> /comfyui/custom_nodes`
- Shared models: `${MODELS_DIR} -> /comfyui/models`
- Optional input/output: `${INPUT_DIR} -> /comfyui/input`, `${OUTPUT_DIR} -> /comfyui/output`
- Tailscale state: named volume `${STACK_NAME}-ts-state -> /var/lib/tailscale`

These mounts keep your settings, nodes, and models persistent across updates.

## Environment Variables
Defined via `.env` and used by `docker-compose.yml` (see `.env.example` for a template):
- `TS_HOSTNAME`: Hostname for the Tailscale node (MagicDNS friendly)
- `TS_AUTHKEY`: Auth key to auto‑enroll into your tailnet
- `STACK_NAME`: Prefix used to name persistent volumes for multi‑instance deployments
- `MODELS_DIR`: Absolute path on host to shared models
- `INPUT_DIR`, `OUTPUT_DIR`: Optional absolute paths for shared I/O directories
 - `COMFYUI_HOST_BIND`: Host bind IP for published port (`0.0.0.0` for LAN, `127.0.0.1` for local-only)
 - `COMFYUI_PORT`: Host port to publish ComfyUI (container listens on 8188)

## Updating
Pull the latest images and recreate:
```bash
docker compose pull
docker compose up -d
```

## Troubleshooting
- GPU not detected in container:
  - Ensure host `nvidia-smi` works.
  - Confirm NVIDIA Container Toolkit is installed and Docker recognizes GPUs.
- Can’t access ComfyUI:
  - Check Tailscale enrollment: `docker logs norns-ts`.
  - Verify the service is up: `docker compose ps` and `docker compose logs -f norns-app`.
- Permissions issues on mounted folders:
  - Ensure host directories exist and are readable/writable by Docker.

## Security Notes
- Traffic is limited to your tailnet via Tailscale; no public ports are exposed.
- If you need external access, consider Tailscale Funnel or a reverse proxy and review security implications carefully.
 - If you enable LAN publishing via `COMFYUI_HOST_BIND`/`COMFYUI_PORT`, the UI is accessible on your local network. Use firewall rules and strong network hygiene.

## Compose File
See `docker-compose.yml` for the full service configuration and volume mappings. The key settings:
- `network_mode: service:norns-ts` to share the sidecar’s network
- `gpus: all` for GPU acceleration
- Named volumes to persist user data, custom nodes, and Tailscale state, parameterized by `${STACK_NAME}`

## Multiple Instances
Run multiple isolated stacks on the same host by setting a unique `STACK_NAME` per deployment. Each stack gets its own volumes (e.g., `my-stack-user`, `my-stack-custom-nodes`, `my-stack-ts-state`) while sharing the host‑level models/input/output directories if desired.

Example:
```bash
STACK_NAME=norns-a docker compose up -d
STACK_NAME=norns-b docker compose up -d
```
Inspect created volumes:
```bash
docker volume ls | grep norns-
```

## License
No license is specified. Use and modify responsibly.
