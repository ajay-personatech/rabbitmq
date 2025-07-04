# Use an official RabbitMQ image.
# Pinning to a specific version is recommended for production.
# RabbitMQ 3.12.x is a current release series. Alpine variant is smaller.
# Ensure this version aligns with your desired LTS and stability requirements.
FROM rabbitmq:3.12-management-alpine

# The official RabbitMQ image already:
# - Sets up a non-root user 'rabbitmq' (UID 999) and group 'rabbitmq' (GID 999)
# - Sets the default user to 'rabbitmq'
# - Exposes default ports (5672, 15672)

# Maintainer label (optional)
LABEL maintainer="infra@persoantech.com" # Replace with your email or team alias
LABEL description="Custom RabbitMQ image based on official Alpine version."

# Copy custom rabbitmq.conf if you have one
# This will replace the default rabbitmq.conf in the base image.
# Ensure my-custom-rabbitmq.conf is in the same directory as the Dockerfile (the build context).
COPY --chown=rabbitmq:rabbitmq my-custom-rabbitmq.conf /etc/rabbitmq/rabbitmq.conf

# Example: If you had custom plugins to add (uncomment and adjust)
# Ensure your plugins are compatible with the RabbitMQ version.
# COPY --chown=rabbitmq:rabbitmq plugins/ /opt/rabbitmq/plugins/

# Example: If you had a custom rabbitmq.conf to bake in (though using ConfigMap is often more flexible)
# COPY --chown=rabbitmq:rabbitmq custom_rabbitmq.conf /etc/rabbitmq/rabbitmq.conf

# Example: If you needed additional packages for health checks or other tools (less common for just RabbitMQ)
# RUN apk --no-cache add curl

# The base image already sets appropriate CMD and ENTRYPOINT.
# No further actions are strictly necessary if just using the base image features
# and ensuring non-root execution (which is handled by the base image).

# Add a simple healthcheck (optional, as Kubernetes probes are primary for K8s)
# This healthcheck is for Docker itself.
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s   CMD rabbitmq-diagnostics -q ping || exit 1

# Default command is already provided by the base image: `rabbitmq-server`
# Default user is already 'rabbitmq'
