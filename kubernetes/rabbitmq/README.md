# RabbitMQ Cluster on GKE with GCP Secret Manager Integration

This directory contains Kubernetes manifests to deploy a fault-tolerant and secure RabbitMQ cluster on Google Kubernetes Engine (GKE), leveraging GCP Secret Manager for sensitive data like Erlang cookies and passwords, and Workload Identity for secure access to GCP resources.

## Features

- **Clustered RabbitMQ**: Deploys a 3-node RabbitMQ cluster using a StatefulSet.
- **High Availability**:
    - Queue mirroring policy (`ha-all`) enabled by default for all queues.
    - PodDisruptionBudget (`rabbitmq-pdb`) to ensure service availability during voluntary disruptions (e.g., node maintenance).
- **Secure Credential Management**:
    - Erlang cookie fetched from GCP Secret Manager using the Secrets Store CSI Driver.
    - Prepared for Workload Identity to allow GKE service accounts to securely access GCP resources (like Secret Manager and Artifact Registry).
- **Security Best Practices**:
    - Runs RabbitMQ containers as a non-root user (UID/GID 999).
    - NetworkPolicies to restrict traffic to and from RabbitMQ pods.
    - Placeholder for admin password hash in ConfigMap, encouraging secure generation.
- **Persistent Storage**: Uses PersistentVolumeClaims for message data.
- **Configuration**: Leverages ConfigMaps for RabbitMQ configurations and enabled plugins.
- **Services**: Exposes AMQP and Management UI via LoadBalancer service.

## Manifests Overview

- `../namespace.yaml`: Defines the `rabbitmq-qa` namespace.
- `rabbitmq-configmap.yaml`: RabbitMQ configuration, including clustering, plugins, and initial definitions (users, policies).
- `rabbitmq-secret-provider-class.yaml`: Configures the Secrets Store CSI Driver to fetch secrets from GCP Secret Manager.
- `rabbitmq-statefulset.yaml`: Deploys RabbitMQ nodes, integrates with Secret Manager via CSI driver, and configured for non-root execution.
- `rabbitmq-headless-svc.yaml`: Headless service for DNS-based peer discovery.
- `rabbitmq-client-svc.yaml`: LoadBalancer service to expose AMQP and Management UI.
- `rabbitmq-pdb.yaml`: PodDisruptionBudget to ensure high availability.
- `rabbitmq-networkpolicy.yaml`: Network policies for restricting traffic.
- `rabbitmq-secret.yaml`: (Legacy for initial setup, Erlang cookie now via CSI) Contains base64 encoded placeholders. Useful if not using CSI for all secrets immediately or for other secrets.

## Prerequisites

1.  **GKE Cluster**: A running GKE cluster with the Secrets Store CSI Driver and its GCP provider installed.
    ```bash
    # Enable Secrets Store CSI Driver on your GKE cluster (if not already enabled)
    # gcloud container clusters update YOUR_CLUSTER_NAME --update-addons ConfigConnector=ENABLED,GcpManagedSecrets=ENABLED --region YOUR_REGION
    # The GcpManagedSecrets addon includes the CSI driver and GCP provider.
    ```
2.  **`kubectl`**: Configured to connect to your GKE cluster.
3.  **GCP Project**: A GCP project with billing enabled.
4.  **IAM Permissions**:
    -   Your user/principal needs permissions to manage GKE, IAM, and Secret Manager.
    -   A Google Service Account (GSA) will be created for RabbitMQ with permissions to:
        -   Read secrets from GCP Secret Manager (`roles/secretmanager.secretAccessor`).
        -   (Optional) Read images from Artifact Registry/GCR if using private images (`roles/artifactregistry.reader`).

## Setup and Deployment

### 1. Clone the Repository (if applicable)

If these files are in a Git repository, clone it.

### 2. Review and Customize Manifests

-   **`kubernetes/rabbitmq/rabbitmq-secret-provider-class.yaml`**:
    -   Replace `YOUR_GCP_PROJECT_ID` with your actual GCP Project ID.
    -   Ensure the secret names (`rabbitmq-erlang-cookie`, `rabbitmq-admin-password`) match the names you will create in Secret Manager.
-   **`kubernetes/rabbitmq/rabbitmq-configmap.yaml`**:
    -   In `definitions.json`, the `admin` user's `password_hash` is a placeholder: `"PLEASE_REPLACE_WITH_SECURE_HASH"`.
        Generate a hash for your desired admin password:
        ```bash
        # Run this in a temporary RabbitMQ Docker container or use an online generator (less secure for prod passwords)
        # docker run -it --rm rabbitmq:3.12 rabbitmqctl hash_password "YourStrongPasswordHere"
        ```
        Replace the placeholder with the generated hash (e.g., `{sha256_base64,LONG_HASH_STRING}`).
-   **(Optional) `kubernetes/rabbitmq/rabbitmq-client-svc.yaml`**:
    -   Change `type: LoadBalancer` if you need a different service type (e.g., `NodePort`, or `ClusterIP` for use with an Ingress).
    -   For GKE internal load balancers, uncomment and configure annotations.

### 3. Create Secrets in GCP Secret Manager

Create the following secrets in GCP Secret Manager (ensure they are in the same project specified in `SecretProviderClass`):

-   **`rabbitmq-erlang-cookie`**: A long, random string for inter-node communication.
    ```bash
    # Example:
    openssl rand -base64 48 | gcloud secrets create rabbitmq-erlang-cookie --data-file=- --project=YOUR_GCP_PROJECT_ID --replication-policy=automatic
    ```
-   **`rabbitmq-admin-password`**: The plain-text password for the `admin` user whose hash you placed in `rabbitmq-configmap.yaml`. This is stored for reference or if you build a system to update users via API. The primary source of truth for the password at deployment is the hash in the ConfigMap.

### 4. Set up Workload Identity

This allows the RabbitMQ pods (via a Kubernetes Service Account) to securely access GCP Secret Manager.

-   **Create a Kubernetes Service Account (KSA)**:
    ```bash
    kubectl create serviceaccount rabbitmq-ksa --namespace rabbitmq-qa
    ```
    (The `rabbitmq-statefulset.yaml` is already configured to use `serviceAccountName: rabbitmq-ksa` if uncommented. For CSI driver access to secrets, the node's GSA is often used by default if the CSI driver is installed with GKE default config. If you need pod-specific GSA for CSI, further annotation on KSA for CSI driver might be needed or ensure `rabbitmq-ksa` is used by CSI driver.)

    For the CSI driver to use a specific KSA (`rabbitmq-ksa`) to impersonate a GSA when accessing secrets, you usually need to annotate the KSA that the *CSI driver's provider pod* runs as, or ensure the nodes themselves have the necessary scope.
    However, for Workload Identity to be used by applications *within the pod* (e.g. if RabbitMQ itself needed to talk to GCP services), you'd set it on `rabbitmq-ksa`.

    The GKE managed addon for Secrets Store CSI Driver (`GcpManagedSecrets`) often uses the GKE node pool's service account. To ensure least privilege for RabbitMQ accessing secrets:
    1. Create a dedicated Google Service Account (GSA) for RabbitMQ.
    ```bash
    GSA_NAME="rabbitmq-gsa"
    GSA_PROJECT="YOUR_GCP_PROJECT_ID"
    gcloud iam service-accounts create ${GSA_NAME} --project=${GSA_PROJECT} --display-name="RabbitMQ GSA"
    GSA_EMAIL="${GSA_NAME}@${GSA_PROJECT}.iam.gserviceaccount.com"
    ```
    2. Grant the GSA access to the secrets:
    ```bash
    gcloud secrets add-iam-policy-binding rabbitmq-erlang-cookie --project=${GSA_PROJECT} --member="serviceAccount:${GSA_EMAIL}" --role="roles/secretmanager.secretAccessor"
    gcloud secrets add-iam-policy-binding rabbitmq-admin-password --project=${GSA_PROJECT} --member="serviceAccount:${GSA_EMAIL}" --role="roles/secretmanager.secretAccessor"
    ```
    3.  **Bind KSA to GSA for Workload Identity**: This allows the KSA to act as the GSA.
    ```bash
    kubectl annotate serviceaccount rabbitmq-ksa       --namespace rabbitmq-qa       iam.gke.io/gcp-service-account=${GSA_EMAIL}       --overwrite # Add --overwrite if KSA already exists and you're re-annotating

    gcloud iam service-accounts add-iam-policy-binding ${GSA_EMAIL}       --role roles/iam.workloadIdentityUser       --member "serviceAccount:${GSA_PROJECT}.svc.id.goog[rabbitmq-qa/rabbitmq-ksa]"       --project=${GSA_PROJECT}
    ```
    4. Ensure the `serviceAccountName: rabbitmq-ksa` is uncommented and set in `kubernetes/rabbitmq/rabbitmq-statefulset.yaml`.

### 5. Deploy to GKE

Apply the namespace first, then all other manifests.

```bash
# 1. Apply the namespace
kubectl apply -f namespace.yaml

# 2. Apply all RabbitMQ manifests to the 'rabbitmq-qa' namespace
kubectl apply -f kubernetes/rabbitmq/ -n rabbitmq-qa
# Alternatively, if you prefer kustomize, create a kustomization.yaml
```

### 6. Verify Deployment

Check the status of the pods, services, and persistent volume claims:

```bash
kubectl get all -n rabbitmq-qa
kubectl get pvc -n rabbitmq-qa
kubectl get pods -n rabbitmq-qa -w # Watch pods come up
```
Check logs of a RabbitMQ pod:
```bash
kubectl logs rabbitmq-0 -n rabbitmq-qa -c rabbitmq # Use -f to follow
```
Look for messages about successful clustering and server startup.
Check that secrets are mounted:
```bash
kubectl exec -it rabbitmq-0 -n rabbitmq-qa -- ls -l /mnt/secrets-store/
kubectl exec -it rabbitmq-0 -n rabbitmq-qa -- cat /mnt/secrets-store/erlang_cookie # (Don't print sensitive data like this in prod logs)
```

## Accessing RabbitMQ

-   **Management UI**:
    Find the external IP address of the `rabbitmq-client` service:
    ```bash
    kubectl get svc rabbitmq-client -n rabbitmq-qa
    ```
    Access the UI at `http://<EXTERNAL_IP>:15672`. Log in with `admin` and the password you set (whose hash is in the ConfigMap).

-   **Client Connections (AMQP)**:
    Use the same `<EXTERNAL_IP>` and port `5672`.
    Connection String: `amqp://admin:YourPassword@<EXTERNAL_IP>:5672`

## High Availability Explained

-   **Queue Policies**: The `ha-all` policy in `definitions.json` (inside `rabbitmq-configmap.yaml`) ensures that all queues are mirrored across all available nodes. `ha-sync-mode: automatic` ensures new mirrors sync automatically.
-   **PodDisruptionBudget (PDB)**: `rabbitmq-pdb.yaml` ensures that at least 2 out of 3 pods remain available during voluntary disruptions (like node upgrades), minimizing downtime.
-   **StatefulSet**: Provides stable pod hostnames and persistent storage, crucial for RabbitMQ clustering and data durability.

## Security Notes

-   **Non-Root User**: RabbitMQ runs as user `rabbitmq` (UID/GID 999).
-   **Network Policies**: Restrict ingress/egress traffic. Review and adjust them to your specific application needs (e.g., which pods can connect to RabbitMQ).
-   **Secret Management**: Erlang cookie is sourced from GCP Secret Manager. Admin password hash is in ConfigMap (ensure the actual password is not easily guessable).
-   **Workload Identity**: Provides a secure, keyless way for pods to access GCP resources.

## Scaling

-   To scale the cluster, change the `replicas` field in `kubernetes/rabbitmq/rabbitmq-statefulset.yaml` and re-apply. RabbitMQ clustering should handle the new nodes. Ensure your PVCs can be provisioned and that underlying resources (CPU, memory, disk IOPS) are sufficient.

## Persistence

-   Message data is stored on PersistentVolumes provisioned by `volumeClaimTemplates` in the StatefulSet. Ensure your default StorageClass or the specified `storageClassName` provides adequate performance and durability.

## Troubleshooting

-   **Pod Startup Issues**: Check pod logs (`kubectl logs ...`), events (`kubectl describe pod ...`), and status of PVCs.
-   **Clustering Problems**:
    -   Ensure the headless service (`rabbitmq-headless`) is correctly configured.
    -   Verify pods can resolve each other's FQDNs (e.g., `rabbitmq-0.rabbitmq-headless.rabbitmq-qa.svc.cluster.local`).
    -   Check the Erlang cookie is identical across all nodes (mounted correctly from Secret Manager).
    -   Review NetworkPolicies to ensure they don't block inter-node communication on ports 4369 (epmd) and 25672 (distribution).
-   **Secret Store CSI Driver Issues**:
    -   Check logs of the CSI driver pods in the `kube-system` namespace.
    -   Verify IAM permissions for the GSA used by the CSI driver or Workload Identity.
    -   Ensure `SecretProviderClass` is correctly configured.
