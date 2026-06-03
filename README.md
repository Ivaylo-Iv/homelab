# Homelab

This repository contains a GitOps-driven homelab configuration for managing applications, infrastructure controllers, and monitoring via Flux and Kustomize.

## Overview

- Purpose: a small, reproducible homelab environment for testing automation, networking, and platform tooling.
- GitOps: the cluster(s) are managed via Flux (GitRepository + Kustomization + HelmRelease manifests under `clusters/`).

## Quickstart

Prerequisites

- A Kubernetes cluster (k3s/kind/k3d/minikube) and `kubectl` configured for it.
- Flux CLI installed and bootstrapped to the target cluster (see Flux docs).
- `kustomize` when rendering or validating manifests locally (optional).

Deploying (high level)

- This repo is intended to be applied by Flux. The Flux `GitRepository` and `Kustomization` manifests under `clusters/` point Flux at this repo and synchronize the directories:
  - The cluster-level kustomizations live in `clusters/homelab`.
  - App manifests are under `apps/homelab` and are referenced by the cluster `Kustomization` at `clusters/homelab/apps.yaml`.
- To validate or apply locally (not recommended for long-term state):

```bash
# render all the app manifests
kustomize build apps/homelab | kubectl apply -f -

# or apply a single app kustomization
kubectl apply -k apps/homelab/homelab
```

## Repository layout

- `apps/` — Application-level kustomizations.
  - `apps/homelab` aggregates sub-apps for this homelab.
    - `homelab/` — Simple static-site app (Deployment, Service, Namespace).
      - image: `ghcr.io/ivaylo-iv/static-site-generator:<commit>` (serves on port 80)
    - `inbrowser-beatmaker/` — Small web app (Deployment, Service, Namespace).
      - image: `ghcr.io/ivaylo-iv/inbrowser-beatmaker:<commit>` (serves on port 80)

- `infrastructure/` — Cluster controllers and infrastructure components managed as kustomize overlays.
  - `cloudflared-connector/` — `cloudflared` tunnel Deployment using a token secret.
  - `longhorn/`, `reflector/` — Storage and helper controllers (kustomize manifests present).
  - `renovate/` — Renovate CronJob and config (automated dependency updates via Renovate).

- `clusters/` — Cluster-specific bootstrap and Flux kustomizations.
  - `clusters/homelab/` contains the Flux `Kustomization` definitions that point to `apps/` and `infrastructure/` paths.

- `monitoring/` — Monitoring stack manifests consumed as HelmRelease(s).
  - `kube-prometheus-stack` — a `HelmRelease` configured to use an existing Grafana secret.

## Notable components & configuration

- Flux: `clusters/homelab/apps.yaml` is a Flux `Kustomization` that syncs `./apps/homelab` with a 1m interval and SOPS decryption configured via `sops-age` secret.
- Renovate: a `CronJob` under `infrastructure/controllers/homelab/renovate` runs `renovate/renovate:latest` against the repo. Note the schedule is currently `* * * * *` (every minute) — adjust as needed.
- Cloudflared: the `cloudflared` Deployment uses a token stored in `cloudflared-token` secret and runs the official cloudflared image.
- Monitoring: `monitoring/controllers/homelab/kube-prometheus-stack/release.yaml` installs the kube-prometheus-stack Helm chart via Flux HelmRelease; Grafana credentials are expected in `grafana-monitoring-secret`.

## Security & secrets

- Secrets are referenced by name (e.g., `cloudflared-token`, `renovate-token`, `sops-age`). Do not store sensitive values in plain YAML in this repo — use SOPS or your cluster's secret management.
- Flux `Kustomization` is configured to use `sops` for decryption (see `clusters/homelab/apps.yaml` for `decryption.provider`).

## Contributing

- Add or update applications under `apps/` using Kustomize bases/overlays.
- Add cluster-level controllers under `infrastructure/controllers/homelab` and include them in the top-level kustomization.
- When changing deployment images, build and push the image to the registry (GHCR in this repo), then update the image tag in the deployment kustomization or use kustomize imageTransformer.

## Useful commands

- Render kustomize output:

```bash
kustomize build apps/homelab
```

- Apply a single kustomization locally:

```bash
kubectl apply -k apps/homelab/homelab
```

- If you manage images locally with `skaffold` or `kind`/`k3d`, push to `ghcr.io/ivaylo-iv/...` or adjust imagePullPolicy for testing.

## Notes & next steps

- Review the Renovate CronJob schedule; running every minute can be noisy.
- Add documentation for how to bootstrap Flux to a new cluster (if you want, I can add a `docs/flux-bootstrapping.md`).

## License

This repository is provided under the terms of the LICENSE file in this repo.
