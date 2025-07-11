# Initial Setup Specification

This document outlines the steps for the initial setup of the system, including the deployment of the One Password operator, Argo CD, and the activation of the `root-app.yaml`.

## 1. Deploy One Password Operator

The One Password operator will be deployed using its Helm chart. This operator is crucial for securely managing secrets within the Kubernetes cluster.

### Instructions:

1.  **Add the One Password Helm repository:**
    ```bash
    helm repo add onepassword https://1password.github.io/onepassword-operator
    helm repo update
    ```
2.  **Install the One Password operator:**
    ```bash
    helm install onepassword-operator onepassword/onepassword-operator -n onepassword-operator --create-namespace
    ```

## 2. Deploy Argo CD

Argo CD will be deployed as the GitOps continuous delivery tool. It will be responsible for automatically synchronizing the desired state of the applications from Git repositories to the Kubernetes cluster.

### Instructions:

1.  **Create the `argocd` namespace:**
    ```bash
    kubectl create namespace argocd
    ```
2.  **Apply the Argo CD manifest:**
    ```bash
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```
3.  **Wait for Argo CD pods to be ready:**
    ```bash
    kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
    ```

## 3. Activate `root-app.yaml`

The `root-app.yaml` file will define the initial set of applications and their configurations that Argo CD will manage. This typically includes core infrastructure components and foundational applications.

### Instructions:

1.  **Ensure `root-app.yaml` is accessible:** This file should be present in a Git repository that Argo CD has access to. For this initial setup, assume it's in the same repository where Argo CD will be configured to sync from.
2.  **Create an Argo CD Application pointing to `root-app.yaml`:**
    ```bash
    kubectl apply -n argocd -f <path-to-your-root-app.yaml-definition-for-argocd>
    ```
    *   **Note:** The `<path-to-your-root-app.yaml-definition-for-argocd>` here refers to an Argo CD Application manifest that points to your actual `root-app.yaml` within a Git repository. An example of such a manifest might look like:

        ```yaml
        apiVersion: argoproj.io/v1alpha1
        kind: Application
        metadata:
          name: root-applications
          namespace: argocd
        spec:
          destination:
            namespace: default # Or your desired target namespace for root apps
            server: https://kubernetes.default.svc
          project: default
          source:
            path: path/to/your/root-app.yaml # Path within your Git repository
            repoURL: https://github.com/your-org/your-repo.git
            targetRevision: HEAD
          syncPolicy:
            automated:
              prune: true
              selfHeal: true
        ```

3.  **Verify synchronization in Argo CD UI:** After applying the Argo CD Application, you can access the Argo CD UI (typically by port-forwarding `argocd-server` service) to observe the `root-applications` syncing and deploying the defined resources.