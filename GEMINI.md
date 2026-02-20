# Gemini Context: k3s GitOps Repository

This repository hosts the GitOps configuration for a Kubernetes (k3s) cluster, managed via Argo CD using the "App of Apps" pattern.

## Project Overview

*   **Cluster:** k3s.
*   **GitOps Controller:** Argo CD (Self-managed).
*   **Secret Management:** 1Password Operator.
*   **Identity Provider:** Authentik.
*   **Chat:** Matrix (Synapse + Sliding Sync) with S3 offload.
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
    *   `actual-budget-app.yaml`: Manages the `actual-budget/` workload.
    *   `alex-bot-app.yaml`: Manages the `alexbot/` workload.
    *   `authentik-app.yaml`: Manages the `authentik/` workload.
    *   `cert-manager.yaml`: Managed cert-manager operator.
    *   `cloudflare.yaml`: Cloudflare Tunnel deployment.
    *   `cnpg.yaml`: Managed CloudNativePG operator.
    *   `matrix-app.yaml`: Manages the `matrix/` workload.
    *   `monitoring.yaml`: Managed Prometheus/Grafana stack.
*   **`actual-budget/`**: Manifests for Actual Budget, using static PV binding.
*   **`alexbot/`**: Consolidated manifests (`app.yaml`, `database.yaml`, `secrets.yaml`).
*   **`authentik/`**: Identity provider configuration (`app.yaml` with Helm values, `database.yaml`).
*   **`matrix/`**: Synapse homeserver, Sliding Sync Proxy, and S3 configuration.
*   **`tailscale/`**: Operator configuration and `ProxyClass` definitions.

## Patterns & Templates

### Storage & Longhorn Patterns

*   **Disabling Default Backups:** To prevent Longhorn from applying the default recurring backup jobs (e.g., for databases managed by CNPG/Barman), you must assign the volume to a specific recurring job group that *excludes* the backup jobs.
    *   *Mechanism:* Create a recurring job (e.g., `weekly-trim`) and assign it to a group (e.g., `disabled`).
    *   *StorageClass:* Set `recurringJobSelector: '[{"name":"disabled", "isGroup":true}]'` in the StorageClass parameters.
    *   *Why:* Simply setting `recurringJobs: '[]'` is often overridden by Longhorn's global default group setting.
*   **Filesystem Trim:** A `weekly-trim` recurring job (task: `filesystem-trim`) is configured for both `default` and `disabled` groups to ensure disk space is reclaimed regularly.
*   **StorageClass Updates:** StorageClass parameters are **immutable**. If you need to change them (e.g., updating `recurringJobSelector`), you must **delete the StorageClass** manually and let Argo CD recreate it. Argo CD sync will fail with "Forbidden: updates to parameters" otherwise.

### Static PersistentVolume Binding (Data Persistence)

To ensure critical application data persists across namespace deletions or "nuke and pave" operations, use the **Static PV** pattern (as seen in `actual-budget/`):

1.  **Define a PersistentVolume (PV):** Explicitly create a `PersistentVolume` resource.
    *   Set `volumeHandle` to the exact name of the existing Longhorn volume.
    *   Set `claimRef` (optional but good practice) or rely on `volumeName` binding.
2.  **Define a PersistentVolumeClaim (PVC):**
    *   Set `volumeName` to match the static PV name.
    *   This forces the PVC to bind to your specific, pre-existing Longhorn volume instead of provisioning a new one.

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

### CloudNativePG (CNPG) & Barman
*   **"Expected empty archive" Error:** If `barman-cloud-wal-archive` fails with this error, it means the destination S3 bucket path is not empty or has conflicting metadata.
    *   **Fix:** Increment the destination path suffix (e.g., from `.../db-v2` to `.../db-v3`) in the `ObjectStore` or `Cluster` configuration to force a fresh archive start.

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

*   **Field Mapping:** Do NOT attempt to use `spec.data` in `OnePasswordItem` or complex sync hooks for field remapping. The operator version/CRD in this cluster does not support it reliably.
*   **Best Practice:** If an application requires specific secret keys (e.g., `AWS_ACCESS_KEY_ID`), **instruct the user to rename the fields directly in the 1Password item** to match the required keys. This is the only supported way to achieve custom keys.
*   **Longhorn S3 Backups:** Require `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_ENDPOINTS`, and `AWS_REGION` keys in the secret. The `backupTarget` URL should also follow the format `s3://bucket@region/path/`.
*   **Longhorn Replicas:** Since this is a single-node cluster, all volumes MUST be configured with `numberOfReplicas: 1`. The global `defaultReplicaCount` is also set to `1`.

### Leantime Deployment
*   **OIDC Configuration:**
    *   **Slug:** `lean-time` (Required for correct Issuer matching in Authentik).
    *   **Env Vars:** Requires explicit mapping for `LEAN_OIDC_CREATE_USER: "true"` and `LEAN_OIDC_DEFAULT_ROLE: "40"`.
    *   **Field Mapping:** Explicitly map email with `LEAN_OIDC_FIELD_EMAIL: "email"`.
    *   **Trusted Proxies:** Leantime defaults to trusting `127.0.0.1`. The "Not a trusted proxy" (403) error is often a red herring for other configuration or health check failures.
*   **Database:**
    *   **Stable Secrets:** Must use an existing Secret (via 1Password) for `database-root`, `database-user`, `database-password`. Relying on the chart's auto-generated secrets causes password rotation on every Argo CD sync, breaking the connection to the persisted volume.
*   **RWO Volume Locking (Important):**
    *   Since Leantime uses a ReadWriteOnce (RWO) volume for the database on a single-node cluster, **RollingUpdate strategies fail** because the old pod holds the volume lock while the new pod tries to start.
    *   **Fix:** Use `strategy: { type: Recreate }` in the Deployment/Helm values to ensure the old pod terminates fully before the new one starts.

## Tool Usage & Safety

### kubectl
*   **NEVER use the `-w` (watch) flag:** Executing `kubectl get ... -w` causes the agent to hang indefinitely as the stream never closes. Always use standard `get` commands or `sleep` loops for status checks.
