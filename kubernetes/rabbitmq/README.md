# RabbitMQ Cluster on GKE with GCP Secret Manager Integration

This directory contains Kubernetes manifests to deploy a fault-tolerant and secure RabbitMQ cluster on Google Kubernetes Engine (GKE), leveraging GCP Secret Manager for sensitive data like Erlang cookies and passwords, and Workload Identity for secure access to GCP resources.

## Features

- **Clustered RabbitMQ**: Deploys a 3-node RabbitMQ cluster using a StatefulSet.
- **High Availability**:
    - Queue mirroring policy (`ha-all`) enabled by default for all queues.
    - PodDisruptionBudget (`rabbitmq-pdb`) to ensure service availability during voluntary disruptions (e.g., node maintenance).
- **Horizontal Pod Autoscaler (HPA)**: Scales pods from 3 to 10 based on CPU/memory.
- **Templating**: Uses ERB templates for dynamic configuration per environment, suitable for Krane deployments.
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

- **Namespace**: Defined via ERB template variable `<%= bindings["namespace"] %>` (formerly `../namespace.yaml`).
- `templates/rabbitmq-configmap.yaml.erb`: RabbitMQ configuration, including clustering, plugins, and initial definitions (users, policies).
- `templates/rabbitmq-secret-provider-class.yaml.erb`: Configures the Secrets Store CSI Driver to fetch secrets from GCP Secret Manager.
- `templates/rabbitmq-statefulset.yaml.erb`: Deploys RabbitMQ nodes, integrates with Secret Manager via CSI driver, and configured for non-root execution.
- `templates/rabbitmq-headless-svc.yaml.erb`: Headless service for DNS-based peer discovery.
- `templates/rabbitmq-client-svc.yaml.erb`: LoadBalancer service to expose AMQP and Management UI.
- `templates/rabbitmq-pdb.yaml.erb`: PodDisruptionBudget to ensure high availability.
- `templates/rabbitmq-hpa.yaml.erb`: HorizontalPodAutoscaler for automatic scaling.
- `templates/rabbitmq-networkpolicy.yaml.erb`: Network policies for restricting traffic.
- `templates/rabbitmq-secret.yaml.erb`: (Legacy placeholder, Erlang cookie now via CSI) ERB template for placeholder Kubernetes Secret. Useful if not using CSI for all secrets or for other manually managed secrets.

## Setup and Deployment with Krane

These manifests are designed to be deployed using Krane, leveraging ERB templates for environment-specific configurations.

### 1. Prerequisites for Krane Deployment

- **Krane CLI**: Ensure you have Krane installed and configured.
- **Ruby Environment**: Krane uses Ruby and ERB.
- **GCP Configuration**: Ensure your environment where Krane runs has access to the target GKE cluster and any necessary GCP services (like Secret Manager if Krane needs to interact with it, though typically secrets are fetched by the CSI driver in-cluster).
- **GKE Cluster**: A running GKE cluster with the Secrets Store CSI Driver and its GCP provider installed.
  ```bash
  # Enable Secrets Store CSI Driver on your GKE cluster (if not already enabled)
  # gcloud container clusters update YOUR_CLUSTER_NAME --update-addons ConfigConnector=ENABLED,GcpManagedSecrets=ENABLED --region YOUR_REGION
  # The GcpManagedSecrets addon includes the CSI driver and GCP provider.
  ```
- **IAM Permissions**:
    - Your user/principal needs permissions to manage GKE, IAM, and Secret Manager for initial setup of secrets and service accounts.
    - The GSA used by Workload Identity (linked to `rabbitmq_ksa_name`) needs `roles/secretmanager.secretAccessor` for the specified secrets.
    - (Optional) If using private images from GCR/GAR, the GSA also needs `roles/artifactregistry.reader`.

### 2. Environment Configuration (Bindings)

Krane uses a mechanism (often a `bindings.yaml` file or environment variables) to provide values to the ERB templates. You will need to define the following bindings for each target environment (e.g., qa, staging, demo, testing):

**Required Bindings:**

- `namespace`: (String) The Kubernetes namespace to deploy to (e.g., `"rabbitmq-qa"`, `"rabbitmq-staging"`). All resources will be deployed into this namespace.
- `gcp_project_id`: (String) Your Google Cloud Project ID where secrets are stored (used by `rabbitmq-secret-provider-class.yaml.erb`).
- `erlang_cookie_secret_name`: (String, Default: `"rabbitmq-erlang-cookie"`) Name of the secret in GCP Secret Manager for the Erlang cookie.
- `admin_password_secret_name`: (String, Default: `"rabbitmq-admin-password"`) Name of the secret in GCP Secret Manager for the admin password (used by SecretProviderClass).

**Application Behavior Bindings:**

- `rabbitmq_base_replicas`: (Integer, Default: `3`) Initial number of RabbitMQ pods for the StatefulSet.
- `hpa_min_replicas`: (Integer, Default: `3`) Minimum replicas for HPA.
- `hpa_max_replicas`: (Integer, Default: `10`) Maximum replicas for HPA.
- `rabbitmq_image_repository`: (String, Default: `"rabbitmq"`) The container image repository.
- `rabbitmq_image_tag`: (String, Default: `"3.12-management"`) The container image tag.
- `rabbitmq_admin_password_hash`: (String, Default: `"PLEASE_REPLACE_WITH_SECURE_HASH"`) The hashed admin password for `definitions.json` in the ConfigMap.
  Generate using: `docker run -it --rm rabbitmq:3.12 rabbitmqctl hash_password "YourStrongPasswordHere"`
  This hash should be for the password whose plain text is stored in the GCP secret referenced by `admin_password_secret_name` if you intend for them to match. The primary mechanism for RabbitMQ login will be this hash.

**Resource Bindings (with defaults):**

- `rabbitmq_cpu_request`: (String, Default: `"500m"`) CPU request per RabbitMQ pod.
- `rabbitmq_memory_request`: (String, Default: `"1Gi"`) Memory request per RabbitMQ pod.
- `rabbitmq_cpu_limit`: (String, Default: `"1"`) CPU limit per RabbitMQ pod.
- `rabbitmq_memory_limit`: (String, Default: `"2Gi"`) Memory limit per RabbitMQ pod.

**Optional Bindings (with defaults):**

- `rabbitmq_ksa_name`: (String, Default: `"rabbitmq-ksa"`) Kubernetes Service Account name to use for the pods. This KSA should be configured for Workload Identity by binding it to a Google Service Account (GSA) that has permissions to access the specified secrets in GCP Secret Manager.
  **Manual Setup for Workload Identity (one-time or per new KSA/GSA):**
    1.  Create KSA: `kubectl create serviceaccount <%= bindings["rabbitmq_ksa_name"] || "rabbitmq-ksa" %> --namespace <%= bindings["namespace"] %>`
    2.  Create GSA (e.g., `rabbitmq-gsa@<%= bindings["gcp_project_id"] %>.iam.gserviceaccount.com`).
    3.  Grant GSA access to secrets (see IAM Permissions above).
    4.  Bind KSA to GSA:
        ```bash
        gcloud iam service-accounts add-iam-policy-binding \
          --role roles/iam.workloadIdentityUser \
          --member "serviceAccount:<%= bindings["gcp_project_id"] %>.svc.id.goog[<%= bindings["namespace"] %>/<%= bindings["rabbitmq_ksa_name"] || "rabbitmq-ksa" %>]" \
          rabbitmq-gsa@<%= bindings["gcp_project_id"] %>.iam.gserviceaccount.com # Replace with your GSA email
        kubectl annotate serviceaccount <%= bindings["rabbitmq_ksa_name"] || "rabbitmq-ksa" %> \
          --namespace <%= bindings["namespace"] %> \
          iam.gke.io/gcp-service-account=rabbitmq-gsa@<%= bindings["gcp_project_id"] %>.iam.gserviceaccount.com # Replace with your GSA email
        ```
- `app_env`: (String) Environment name (e.g., `"qa"`, `"staging"`). Can be used for conditional logic in templates or naming conventions. (Example usage for cluster name or annotations is commented in templates).
- `rabbitmq_erlang_cookie_base64`: (String, Default: `"PLACEHOLDER_ERLANG_COOKIE"`) Base64 encoded Erlang cookie for `templates/rabbitmq-secret.yaml.erb` (if used).
- `rabbitmq_admin_password_base64`: (String, Default: `"PLACEHOLDER_ADMIN_PASSWORD_BASE64"`) Base64 encoded admin password for `templates/rabbitmq-secret.yaml.erb` (if used).

**Example `bindings.yaml` snippet (conceptual):**
```yaml
# For QA environment
namespace: "rabbitmq-qa"
gcp_project_id: "my-gcp-project-qa"
erlang_cookie_secret_name: "qa-rabbitmq-erlang-cookie"
admin_password_secret_name: "qa-rabbitmq-admin-password"
rabbitmq_base_replicas: 3
# ... other qa specific values
```

### 3. Create Secrets in GCP Secret Manager (one-time)

Ensure the secrets referenced by `erlang_cookie_secret_name` and `admin_password_secret_name` exist in GCP Secret Manager in the `gcp_project_id`.

-   **Erlang Cookie**: Store a long, random string.
    ```bash
    # Example:
    openssl rand -base64 48 | gcloud secrets create YOUR_ERLANG_COOKIE_SECRET_NAME --data-file=- --project=YOUR_GCP_PROJECT_ID --replication-policy=automatic
    ```
-   **Admin Password**: Store the plain-text password for the `admin` user. The hash of this password should be used for the `rabbitmq_admin_password_hash` binding.

### 4. Krane Deployment Steps

1.  **Navigate to your deployment directory** where your Krane configuration and these templates reside.
2.  **Ensure your Krane bindings** for the target environment are correctly set up (e.g., via environment variables or a bindings file that Krane loads).
3.  **Run `krane deploy`**:
    ```bash
    krane deploy <your_target_environment_or_cluster> --filenames kubernetes/rabbitmq/templates/
    ```
    (Adjust the command based on your specific Krane setup, especially how you specify the target namespace/environment and template paths).

### 5. Verify Deployment

Verification steps are similar to traditional `kubectl` based ones, but you would target the namespace specified in your Krane deployment:

- `kubectl get all -n <%= bindings["namespace"] %>`
- `kubectl get pvc -n <%= bindings["namespace"] %>`
- `kubectl logs rabbitmq-0 -n <%= bindings["namespace"] %> -c rabbitmq` (use the correct pod name if different)
- Check secrets mounted via CSI: `kubectl exec -it rabbitmq-0 -n <%= bindings["namespace"] %> -- ls -l /mnt/secrets-store/` (use the correct pod name)

## Horizontal Pod Autoscaling (HPA)

To automatically scale the RabbitMQ cluster based on load, a `HorizontalPodAutoscaler` (`rabbitmq-hpa.yaml`) is configured with the following parameters:

- **Target**: `rabbitmq` StatefulSet
- **Minimum Replicas**: 3
- **Maximum Replicas**: 10
- **Metrics for Scaling**:
    - CPU Utilization: Scales up if average CPU utilization across pods exceeds 60%.
    - Memory Utilization: Scales up if average memory utilization across pods exceeds 60%.
- **Scaling Behavior**: Includes defined policies for more controlled scale-up (quick) and scale-down (stabilized) operations.

### HPA Prerequisites

- **Metrics Server**: The Kubernetes Metrics Server must be installed and running in your cluster. GKE clusters usually have this pre-installed or available as an addon. Your Krane deployment should target a cluster where Metrics Server is active for HPA to function.
  If needed, you can typically install it with:
  ```bash
  kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
  ```
  Verify its installation with `kubectl get deployment metrics-server -n kube-system`.

### Autoscaling Implications for RabbitMQ

- **Node Discovery**: The RabbitMQ Kubernetes peer discovery plugin is designed to handle nodes joining and leaving the cluster, which is essential for autoscaling.
- **Queue Data During Scale-Down**:
    - **Classic Mirrored Queues (Current Setup)**: When the cluster scales down, RabbitMQ nodes are removed. If classic mirrored queues (`ha-mode: all` or `ha-mode: exactly`) are used, RabbitMQ will attempt to re-synchronize mirrors from the departing node to remaining nodes. This process can take time and consume resources. For graceful shutdown, ensuring queues are fully synced before a node is terminated is ideal, though HPA-driven scale-down is typically abrupt. The configured `ha-sync-mode: automatic` helps.
    - **Quorum Queues**: For environments with frequent scaling, Quorum Queues (available in RabbitMQ 3.8+) are generally recommended over classic mirrored queues. They offer better data safety and are designed with modern distributed systems principles in mind, handling node additions/removals more robustly. Migrating to Quorum Queues would involve changing the HA policies in `templates/rabbitmq-configmap.yaml.erb`.
- **Monitoring**: Monitor HPA events (`kubectl describe hpa rabbitmq-hpa -n <%= bindings["namespace"] %>`) and RabbitMQ cluster status during scaling operations.

## Accessing RabbitMQ

-   **Management UI**:
    Find the external IP address of the `rabbitmq-client` service:
    ```bash
    kubectl get svc rabbitmq-client -n <%= bindings["namespace"] %>
    ```
    Access the UI at `http://<EXTERNAL_IP>:15672`. Log in with `admin` and the password whose hash is defined by the `rabbitmq_admin_password_hash` binding.

-   **Client Connections (AMQP)**:
    Use the same `<EXTERNAL_IP>` and port `5672`.
    Connection String: `amqp://admin:YourPassword@<EXTERNAL_IP>:5672` (replace `YourPassword` with the actual password).

## High Availability Explained

-   **Queue Policies**: The `ha-all` policy in `definitions.json` (inside `templates/rabbitmq-configmap.yaml.erb`) ensures that all queues are mirrored across all available nodes. `ha-sync-mode: automatic` ensures new mirrors sync automatically.
-   **PodDisruptionBudget (PDB)**: `templates/rabbitmq-pdb.yaml.erb` ensures that at least 2 out of 3 pods remain available during voluntary disruptions (like node upgrades), minimizing downtime.
-   **StatefulSet**: `templates/rabbitmq-statefulset.yaml.erb` provides stable pod hostnames and persistent storage, crucial for RabbitMQ clustering and data durability.

## Security Notes

-   **Non-Root User**: RabbitMQ runs as user `rabbitmq` (UID/GID 999).
-   **Network Policies**: Restrict ingress/egress traffic. Review and adjust them to your specific application needs (e.g., which pods can connect to RabbitMQ).
-   **Secret Management**: Erlang cookie is sourced from GCP Secret Manager. Admin password hash is in ConfigMap (ensure the actual password is not easily guessable).
-   **Workload Identity**: Provides a secure, keyless way for pods to access GCP resources.

## Scaling

-   To scale the cluster, adjust the `rabbitmq_base_replicas` binding for the StatefulSet (if not using HPA or to set a new base for HPA) or rely on the HPA settings (`hpa_min_replicas`, `hpa_max_replicas`) in `templates/rabbitmq-hpa.yaml.erb`. Krane will apply these changes. RabbitMQ clustering should handle the new nodes. Ensure your PVCs can be provisioned and that underlying resources (CPU, memory, disk IOPS) are sufficient.

## Persistence

-   Message data is stored on PersistentVolumes provisioned by `volumeClaimTemplates` in the StatefulSet. Ensure your default StorageClass or the specified `storageClassName` provides adequate performance and durability.

## Troubleshooting

-   **Pod Startup Issues**: Check pod logs (`kubectl logs ...`), events (`kubectl describe pod ...`), and status of PVCs.
-   **Clustering Problems**:
    -   Ensure the headless service (`templates/rabbitmq-headless-svc.yaml.erb`) is correctly configured.
    -   Verify pods can resolve each other's FQDNs (e.g., `rabbitmq-0.rabbitmq-headless.<%= bindings["namespace"] %>.svc.cluster.local`).
    -   Check the Erlang cookie is identical across all nodes (mounted correctly from Secret Manager).
    -   Review NetworkPolicies to ensure they don't block inter-node communication on ports 4369 (epmd) and 25672 (distribution).
-   **Secret Store CSI Driver Issues**:
    -   Check logs of the CSI driver pods in the `kube-system` namespace.
    -   Verify IAM permissions for the GSA used by the CSI driver or Workload Identity.
    -   Ensure `SecretProviderClass` is correctly configured.
