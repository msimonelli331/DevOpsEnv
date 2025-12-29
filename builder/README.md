# Builder

Local pipeline builder image. This will be built with the private registries ca cert and credentials so that it can seemlessly build and push images.

## Setup

1. Populate ca.crt with the one you created when setting up the private container registry

2. Update env vars in the `.gitea/workflows/main.yml` to match the services

   - **DEPRECATED** `BUILDKIT_DAEMON` is the kubernetes service name
   - `REGISTRY` is the kubernetes ingress name

3. Add secrets for `NEXUS_USERNAME` and `NEXUS_PASSWORD` in this git repos Settings > Actions > Secrets

   - The secrets should match the username and password you used when setting up the private container registry
