# Slicer Hub (Nesomic)

Multi-user [3D Slicer](https://www.slicer.org/) sessions in the browser: the 3D Slicer counterpart of
JupyterHub. A user signs in, the hub starts one 3D Slicer desktop for them (KasmVNC, from the
[docker-slicer](https://github.com/NESOMIC/docker-slicer) image) and publishes it under
`https://<hub host>/<session uuid>/` through its own nginx sidecar.

This branch (`nesomic`) is the adaptation to the Nesomic platform. It shares its code with
[t4-hub](https://github.com/NESOMIC/t4-hub) (the same hub, evolved for Transformer4), from which the
Kubernetes orchestrator, the Keycloak sign-in, the session guards and the Nesomic header were brought
back. The original Docker Compose + LDAP deployment is in the `main` and `develop` branches.

## How it works

- **Sign-in**: "Continue with Google", through the Nesomic Keycloak realm (`USE_KEYCLOAK=true`,
  client `nesomic-hub`) or Google OAuth directly.
- **Authorization**: the hub asks the platform for the user's roles
  (`GET ${NEXTGENDEM_BASE_URL}/api/user_roles?email=...`). `APP_ROLE` (default `3dslicer`) or
  `sys-admin` may open a session; `APP_ADMIN_ROLE` (default `3dslicer-admin`) or `sys-admin` also
  manage other people's sessions.
- **Sessions**: one Deployment per user, rendered from `app-deployment-template2.yaml` (or the file
  named by `APP_MANIFEST_TEMPLATE`) and applied with `kubectl`. Pods carry the label
  `app=${APP_LABEL}` (default `slicer`). Sessions without CPU activity for `INACTIVITY_TIME_SEC`
  are removed; documents live in `/home/kasm-user/Documents`.
- **Sharing**: a session can be shared read-only or interactive from its management page.

## Configuration

Read from the environment, or from the file named by `ENV_FILE` (`app_hub_local.env` is the
template, with comments). Variables specific to this hub:

| Variable | Default | Meaning |
|---|---|---|
| `APP_ROLE` / `APP_ADMIN_ROLE` | `3dslicer` / `3dslicer-admin` | platform roles for users / session admins |
| `APP_LABEL` | `slicer` | `app` label of session pods |
| `APP_IMAGE_NAME` / `APP_IMAGE_TAG` | `slicer` / `latest` | session image |
| `SESSION_CPU_REQUESTED` / `SESSION_CPU_LIMIT` | `1` / `4` | per-session CPU |
| `SESSION_MEM_REQUESTED` / `SESSION_MEM_LIMIT` | `2Gi` / `8Gi` | per-session memory |
| `SESSION_GPU` | `false` | request one `nvidia.com/gpu` per session |
| `MAX_SESSIONS`, `INACTIVITY_TIME_SEC` | | capacity and idle culling |

### GPU

3D Slicer uses the GPU for rendering, not for computation. Without one it renders in software
(Mesa llvmpipe): slice views are fine, volume rendering is slow. With `SESSION_GPU=true` each
session requests `nvidia.com/gpu: 1` and tolerates the `nvidia.com/gpu` taint. That unit does not
have to be a whole card: with the NVIDIA device plugin configured for time-slicing, one small GPU
(for example a T4) is advertised as several `nvidia.com/gpu` and shared by as many sessions.

## Build

```bash
docker build -t slicer-hub .
```

The session image is built from https://github.com/NESOMIC/docker-slicer.

## Deployment

The Kubernetes manifests (hub Deployment with the nginx sidecar, RBAC, Ingress, session
NetworkPolicy, per-environment overlays) are in the platform's infrastructure repository
(`nsm-sys`, `kustomize/base/slicer-hub`).

## License, authors and funding

BSD 3-Clause License, the same as the rest of the NESOMIC platform: see `LICENSE`. Copyright
belongs to the NESOMIC authors listed in `AUTHORS`.

Developed within the NEXTGENDEM project (https://www.nextgendem.eu), funded by the Interreg MAC
2014-2020 Cooperation Programme (project code MAC2/4.6d/236), and continued under a commission
from the Cabildo de Gran Canaria.
