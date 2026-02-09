# Gemini Context: k3s GitOps Repository

This repository hosts the GitOps configuration for a Kubernetes (k3s) cluster, managed via Argo CD using the "App of Apps" pattern.

## Project Overview

*   **Cluster:** k3s.
*   **GitOps Controller:** Argo CD (Self-managed).
*   **Secret Management:** 1Password Operator.
*   **Network/Ingress:** Tailscale Kubernetes Operator (exposing services directly to the tailnet).

## Architecture & Workflow

1.  **Argo CD Root:** The bootstrap entry point is `root-app.yaml`, which watches the `apps/` directory.
2.  **Self-Management:** Argo CD manages itself via `apps/argo-cd.yaml`. It uses the official Argo CD Helm chart.
3.  **Tailscale Ingress:** 
    *   Applications are exposed using Tailscale `LoadBalancer` services or `Ingress` objects with the `tailscale` ingress class.
    *   Argo CD is reachable at `https://argo.moose-amberjack.ts.net`.
    *   TLS is terminated by Tailscale; Argo CD server runs in `--insecure` mode to prevent redirect loops.
4.  **ACLs & Tagging:** 
    *   The cluster uses Tailscale tags for access control.
    *   Operator nodes and proxies are typically tagged with `tag:deployment`.
    *   The Tailscale Operator itself is tagged with `tag:k8s-operator`.

## Directory Structure

*   **`root-app.yaml`**: The "App of Apps" bootstrap manifest.
*   **`apps/`**: The control plane for all cluster workloads.
    *   `argo-cd.yaml`: Self-management for Argo CD (v7.8.2+).
    *   `alex-bot-app.yaml`: Manages the `alexbot/` workload.
    *   `traefik.yaml`: Deploys Traefik, also exposed via Tailscale.
*   **`alexbot/`**: Custom manifests for the Alex Bot application and its database (CloudNativePG).
*   **`tailscale/`**: Tailscale operator configuration and OAuth secrets (synced from 1Password).

## Key Operational Commands

### Deploying Changes
1.  Modify manifests in `apps/` or workload directories.
2.  Commit and Push to Git.
3.  Hard Refresh the `root` application if changes aren't picked up immediately:
    ```bash
    kubectl patch application root -n argocd --type merge -p='{"metadata": {"annotations": {"argocd.argoproj.io/refresh": "hard"}}}'
    ```

### Disaster Recovery ("Nuke and Pave")
If Argo CD becomes unstable or has immutable field conflicts:
1.  Manually bootstrap a minimal Argo CD:
    ```bash
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side
    ```
2.  Apply the root app to restore the GitOps state:
    ```bash
    kubectl apply -f root-app.yaml
    ```

## Patterns & Templates

### Adding a New Application with Tailscale Ingress

To add a new workload and expose it at `custom-name.moose-amberjack.ts.net`:

1.  **Create Workload Manifests** (e.g., in `my-app/`):
    ```yaml
    # Deployment
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-app
      namespace: my-namespace
    spec:
      # ... standard deployment spec ...
    ---
    # Service
    apiVersion: v1
    kind: Service
    metadata:
      name: my-app
      namespace: my-namespace
      annotations:
        "tailscale.com/expose": "true"
        "tailscale.com/https": "true"
        "tailscale.com/hostname": "custom-name" # The MagicDNS name
        "tailscale.com/tags": "tag:deployment"
    spec:
      type: LoadBalancer
      selector:
        app: my-app
      ports:
      - port: 443
        targetPort: 8080 # Your app's internal port
    ```

2.  **Create Argo CD Application** (in `apps/my-app.yaml`):
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: my-app
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://github.com/xela-zone/k3s.git
        path: my-app
        targetRevision: HEAD
      destination:
        server: https://kubernetes.default.svc
        namespace: my-namespace
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
    ```

3.  **Commit and Push:** Argo CD will automatically provision the proxy pod and the Tailscale node.

## Development Conventions

*   **Service Exposure:** Prefer `type: LoadBalancer` with Tailscale annotations for dedicated MagicDNS names.
*   **TLS:** Let Tailscale handle certificates (`tailscale.com/https: "true"`) whenever possible.
*   **Versions:** Always pin Helm chart `targetRevision` in Application manifests.