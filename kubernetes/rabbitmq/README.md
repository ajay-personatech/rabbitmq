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


## Building and Using a Custom RabbitMQ Docker Image

This project includes a `Dockerfile` at `../../docker/rabbitmq/Dockerfile` (relative to this READMEs location, or adjust path as needed) to build a custom RabbitMQ image. While the official RabbitMQ images are secure and well-maintained (and used as a base), building your own allows for:
- Pinning to specific, vetted LTS versions.
- Baking in custom plugins or configurations if needed in the future.
- Standardizing your image supply chain and storing images in your own Artifact Registry.

The provided `Dockerfile` uses `rabbitmq:3.12-management-alpine` as a base.

### 1. Build the Docker Image

Navigate to the directory containing the `Dockerfile` for RabbitMQ (e.g., `docker/rabbitmq/` from the project root) and run the build command:

\`\`\`bash
# From the project root directory:
cd docker/rabbitmq

# Replace with your desired image name and tag
export IMAGE_NAME="asia-south1-docker.pkg.dev/phoenix-development-351409/custom-rabbitmq:latest" # Or your specific image name and tag like v1.0.0

docker build -t ${IMAGE_NAME} .

# (Optional) To return to the project root:
# cd ../..
\`\`\`

### 2. Push the Image to Artifact Registry

Ensure you have authenticated Docker to your GCP Artifact Registry.

\`\`\`bash
# This command assumes IMAGE_NAME is still set from the build step
docker push ${IMAGE_NAME}
\`\`\`
If you havent configured Docker for Artifact Registry, you might need to run:
\`\`\`bash
# gcloud auth configure-docker asia-south1-docker.pkg.dev
\`\`\`

### 3. Update Bindings to Use the Custom Image

After building and pushing your custom image, you **MUST** update the `bindings.yaml` files for each of your environments (`config/deploy/<env>/bindings.yaml`) to point to this new image:

-   **`rabbitmq_image_repository`**: Set this to the path of your image in Artifact Registry, without the tag.
    (e.g., `asia-south1-docker.pkg.dev/phoenix-development-351409/custom-rabbitmq`)
-   **`rabbitmq_image_tag`**: Set this to the tag you used during the build (e.g., `latest`, `v1.0.0`).

**Example update in a `bindings.yaml` file:**

\`\`\`yaml
# ... other bindings ...
rabbitmq_image_repository: "asia-south1-docker.pkg.dev/phoenix-development-351409/custom-rabbitmq"
rabbitmq_image_tag: "latest" # Or your specific build tag
# ... other bindings ...
\`\`\`

The ERB templates for the StatefulSet (`kubernetes/rabbitmq/templates/rabbitmq-statefulset.yaml.erb`) will then use these binding values to pull your custom image.

### Alternative: Baking `rabbitmq.conf` into the Docker Image

While the default setup uses a Kubernetes ConfigMap to supply `rabbitmq.conf`, `definitions.json`, and `enabled_plugins`, you also have the option to bake your primary `rabbitmq.conf` directly into your custom Docker Image. This approach has been partially prepared in this repository.

**How it Works if Fully Implemented:**

1.  **Custom Configuration File**:
    -   A file named `my-custom-rabbitmq.conf` is provided in the `docker/rabbitmq/` directory. You would populate this with your desired base RabbitMQ configurations.
    *   Example content in `docker/rabbitmq/my-custom-rabbitmq.conf`:
        ```ini
        # docker/rabbitmq/my-custom-rabbitmq.conf
        default_vhost = /customvhost
        consumer_timeout = 30000
        ```

2.  **Dockerfile Modification**:
    -   The `docker/rabbitmq/Dockerfile` has been updated to include a `COPY` instruction:
        ```dockerfile
        COPY --chown=rabbitmq:rabbitmq my-custom-rabbitmq.conf /etc/rabbitmq/rabbitmq.conf
        ```
    -   When you build your custom Docker image, this `my-custom-rabbitmq.conf` becomes `/etc/rabbitmq/rabbitmq.conf` inside the image.

3.  **Kubernetes ConfigMap (`rabbitmq-configmap.yaml.erb`) Changes**:
    -   The `data` section for `rabbitmq.conf` has been removed from this ConfigMap template (`kubernetes/rabbitmq/templates/rabbitmq-configmap.yaml.erb`).
    -   The ConfigMap now primarily serves `definitions.json` and `enabled_plugins`.

4.  **Kubernetes StatefulSet (`rabbitmq-statefulset.yaml.erb`) Changes**:
    -   The `config-volume` definition (sourced from the `rabbitmq-config` ConfigMap) in the StatefulSet template (`kubernetes/rabbitmq/templates/rabbitmq-statefulset.yaml.erb`) has been modified to no longer project the `rabbitmq.conf` key.
    -   This means the pods will use the `/etc/rabbitmq/rabbitmq.conf` from the Docker image, not from the ConfigMap.

**Pros of Baking `rabbitmq.conf` into the Image:**

-   **Immutability**: The core `rabbitmq.conf` becomes an immutable part of your specific image version.
-   **Consistency**: Ensures all containers from that image version share the exact same base configuration.
-   **Simplified ConfigMap**: The ConfigMap becomes simpler, focusing only on dynamic definitions or plugin enablement.

**Cons of Baking `rabbitmq.conf` into the Image:**

-   **Less Flexibility**: Changes to `rabbitmq.conf` require rebuilding and redeploying the Docker image, which is a slower process than updating a ConfigMap and performing a rolling restart of pods.
-   **Environment-Specific Core Configs**: If your base `rabbitmq.conf` needs to differ significantly per environment (beyond what ENV variable overrides can achieve), you might need different Docker images or a more complex Dockerfile, whereas ConfigMaps can be more easily templated per environment.

**When to Choose:**

-   **Baked-in `rabbitmq.conf`**: Good for very stable, universal base configurations that define the fundamental behavior of your RabbitMQ nodes and are unlikely to change frequently.
-   **ConfigMap for `rabbitmq.conf` (Previous Default)**: Generally more flexible for most use cases, especially if you anticipate needing to tweak configurations or have environment-specific variations that are not easily handled by environment variable overrides alone.

**Current State in this Repository:**
The repository is now set up to demonstrate the **baked-in `rabbitmq.conf`** approach. If you prefer to manage `rabbitmq.conf` via ConfigMap for more flexibility, you would need to:
1. Revert the changes made to `docker/rabbitmq/Dockerfile` (remove the COPY line for `my-custom-rabbitmq.conf`).
2. Revert the changes to `kubernetes/rabbitmq/templates/rabbitmq-configmap.yaml.erb` (re-add the `rabbitmq.conf` data section).
3. Revert the changes to `kubernetes/rabbitmq/templates/rabbitmq-statefulset.yaml.erb` (re-add `rabbitmq.conf` to the `items` in `config-volume`).

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

### Environment Configuration via `bindings.yaml`

To manage environment-specific values for the ERB templates, this project now includes a recommended structure using `bindings.yaml` files located in `config/deploy/<environment_name>/`.

**Example Structure:**

```
config/
└── deploy/
    ├── qa/
    │   └── bindings.yaml       # Bindings for the QA environment
    ├── staging/
    │   └── bindings.yaml    # Bindings for the Staging environment
    ├── feature/
    │   └── bindings.yaml    # Bindings for a Feature environment
    └── demo/
        └── bindings.yaml       # Bindings for a Demo environment
    # You can add more environments like 'testing' following this pattern.
```

- Example `bindings.yaml` files have been provided for `qa` (`config/deploy/qa/bindings.yaml`) and `staging` (`config/deploy/staging/bindings.yaml`).
- Example files for `feature` (`config/deploy/feature/bindings.yaml`) and `demo` (`config/deploy/demo/bindings.yaml`) have also been added, adapting the QA/Staging examples.
- **These files are illustrative examples.** You **MUST** review and update them with your actual environment-specific values, especially for:
    - `gcp_project_id` (examples use `"phoenix-development-351409"` but verify for each environment)
    - Secret names (`erlang_cookie_secret_name`, `admin_password_secret_name`)
    - `rabbitmq_admin_password_hash`
    - Image repository if using a private registry for RabbitMQ images.
    - Resource requests and limits.
- You can create similar `bindings.yaml` files for other environments like `demo` and `testing` by copying one of the examples and adjusting the values accordingly.
- Your Krane deployment process will need to be configured to load the appropriate `bindings.yaml` file for the target environment to make these values available to the ERB templates. How Krane ingests these bindings (e.g., via a specific command-line flag, by convention, or loaded into environment variables that ERB can access) depends on your Krane setup. The ERB templates currently expect these values to be accessible via a hash named `bindings` (e.g., `<%= bindings['namespace'] %>`).

**List of Expected Bindings (to be defined in your `bindings.yaml`):**

(This list is largely the same as before, but now it's contextualized with the `bindings.yaml` files)
- `namespace`: (String) The Kubernetes namespace (e.g., `"qa-rabbitmq-new"`, `"staging-rabbitmq-new"`).
- `gcp_project_id`: (String) GCP Project ID.
- `erlang_cookie_secret_name`: (String) Name of the Erlang cookie secret in GCP Secret Manager.
- `admin_password_secret_name`: (String) Name of the admin password secret in GCP Secret Manager.
- `rabbitmq_base_replicas`: (Integer) Initial RabbitMQ pods.
- `hpa_min_replicas`: (Integer) HPA minimum replicas.
- `hpa_max_replicas`: (Integer) HPA maximum replicas.
- `rabbitmq_admin_password_hash`: (String) Hashed admin password for `definitions.json`.
- `rabbitmq_image_repository`: (String) The full container image repository path (e.g., `"asia-south1-docker.pkg.dev/phoenix-development-351409/rabbitmq-qa"`). This is constructed using your artifact repository location, GCP project ID, and image name.
- `rabbitmq_image_tag`: (String) Container image tag.
- `rabbitmq_cpu_request`: (String) CPU request.
- `rabbitmq_memory_request`: (String) Memory request.
- `rabbitmq_cpu_limit`: (String) CPU limit.
- `rabbitmq_memory_limit`: (String) Memory limit.
- `rabbitmq_ksa_name`: (String, Optional) Kubernetes Service Account name.
- `app_env`: (String, Optional) Environment name.
- `erlang_cookie_placeholder`: (String, Optional) Placeholder for `templates/rabbitmq-secret.yaml.erb`.
- `admin_password_placeholder`: (String, Optional) Placeholder for `templates/rabbitmq-secret.yaml.erb`.
# Add any other custom bindings your ERB templates might expect here.

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
