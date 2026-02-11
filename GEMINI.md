# Gemini Context: k3s GitOps Repository

This repository hosts the GitOps configuration for a Kubernetes (k3s) cluster, managed via Argo CD using the "App of Apps" pattern.

## Project Overview

*   **Cluster:** k3s.
*   **GitOps Controller:** Argo CD (Self-managed).
*   **Secret Management:** 1Password Operator.
*   **Network/Ingress:** Tailscale Kubernetes Operator.

## Architecture & Workflow

1.  **Argo CD Root:** The bootstrap entry point is `root-app.yaml`, which watches the `apps/` directory.
2.  **Self-Management:** Argo CD manages itself via `apps/argo-cd.yaml`. It uses the official Argo CD Helm chart.
3.  **Tailscale Ingress:** 
    *   Applications are exposed using Tailscale `LoadBalancer` services or `Ingress` objects with the `tailscale` ingress class.
    *   Argo CD is reachable at `https://argo.moose-amberjack.ts.net`.
    *   **TLS Termination:** Handled by Tailscale. Argo CD server must run in `--insecure` mode to prevent redirect loops.

## Directory Structure

*   **`root-app.yaml`**: The "App of Apps" bootstrap manifest.
*   **`apps/`**: The control plane for all cluster workloads.
    *   `argo-cd.yaml`: Self-management for Argo CD (v7.8.2+).
    *   `alex-bot-app.yaml`: Manages the `alexbot/` workload.
    *   `cnpg.yaml`: Managed CloudNativePG operator.
    *   `monitoring.yaml`: Managed Prometheus/Grafana stack.
*   **`alexbot/`**: Consolidated manifests (`app.yaml`, `database.yaml`, `secrets.yaml`).
*   **`tailscale/`**: Operator configuration and `ProxyClass` definitions.

## Patterns & Templates

### Adding a New Application with Tailscale Ingress

To add a new workload and expose it at `custom-name.moose-amberjack.ts.net`:

1.  **Create Ingress Manifest**:
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: my-app
      namespace: my-namespace
      annotations:
        tailscale.com/hostname: "custom-name"
        tailscale.com/https: "true"
        tailscale.com/tags: "tag:deployment"
        tailscale.com/proxy-class: "https-proxy"
    spec:
      ingressClassName: tailscale
      tls:
      - hosts:
        - custom-name.moose-amberjack.ts.net
      rules:
      - host: custom-name.moose-amberjack.ts.net
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-service
                port:
                  number: 80
    ```

## Troubleshooting & Learnings

### Naming Collisions (Tailscale)
If a new node shows up as `name-1` instead of `name`, it means an old machine record exists in the Tailscale Admin Console. 
*   **Fix:** Manually delete the old machine from the Tailscale Admin Console, then delete the proxy pod in K8s to trigger a fresh registration.

### Redirect Loops (PR_CONNECT_RESET_ERROR)
If you get redirect errors when using Tailscale HTTPS termination:
*   Ensure the backend application is running in "insecure" or "http" mode (e.g., Argo CD `--insecure` flag).
*   Verify the `ProxyClass` is correctly forcing the `TS_HTTPS_PORT: "443"` environment variable.

### Disaster Recovery ("Nuke and Pave")
If Argo CD has immutable field conflicts (e.g., `spec.selector` changes in Helm charts):
1.  Manually bootstrap a minimal Argo CD:
    ```bash
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side --force-conflicts
    ```
2.  Delete the conflicting deployments/statefulsets manually.
3.  Let Argo CD recreate them via the `root` application.

## Development Conventions

*   **Consolidation:** Group related resources (Deployment + Service) into single files (e.g., `app.yaml`) to reduce clutter.
*   **Server-Side Apply:** Use `ServerSideApply=true` in Argo CD for large resources like Prometheus CRDs.

## Important: 1Password Operator (Legacy)

The 1Password operator in this cluster (v1.11.0+) creates Kubernetes Secrets where the **Keys** match the **Field Labels** in 1Password.

*   **Field Mapping:** Do NOT attempt to use `spec.data` in `OnePasswordItem` or complex sync hooks for field remapping. The operator version/CRD might not support it reliably.
*   **Best Practice:** If an application requires specific secret keys (e.g., `AWS_ACCESS_KEY_ID`), ask the user to rename the fields directly in the 1Password item to match the required keys.
*   **Longhorn S3 Backups:** Require `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_ENDPOINTS`, and `AWS_REGION` keys in the secret. The `backupTarget` URL should also follow the format `s3://bucket@region/path/`.
