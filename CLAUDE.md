# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Kubernetes GitOps repository managed by **Flux CD v2**. It uses a multi-environment structure with declarative infrastructure and application deployments.

### Key Architecture Concepts

**Multi-Environment Structure:**
- Flux syncs from the `dev` branch to the dev environment
- `environments/dev/` contains environment-specific configuration
- `cluster/` contains shared cluster configurations used across environments
- The dev environment kustomization includes both `config` (env-specific) and `../../cluster/base` (shared)

**Reconciliation Order:**
1. **Config** (`cluster/base/config.yaml`) - Creates ConfigMap (`cluster-settings`) and Secret (`cluster-secrets`) from `cluster/config/`
2. **CRDs** (`cluster/base/crds.yaml`) - Installs Custom Resource Definitions from `cluster/crds/`
3. **Infrastructure** (`cluster/base/infrastructure.yaml`) - Deploys infrastructure components from `cluster/infra/`
4. **Apps** (`cluster/base/apps.yaml`) - Deploys applications from `cluster/apps/`

The `apps` Kustomization explicitly depends on `infrastructure` and `crds` via `dependsOn`.

**Secrets Management:**
- SOPS (with age encryption) is used for all secrets
- Encryption keys are defined in `.sops.yaml`
- Secrets follow the naming pattern `*.sops.yaml`
- Flux automatically decrypts secrets using the `sops-age` Secret in `flux-system` namespace
- Encrypted regex: `^(data|stringData)$` - only these fields are encrypted

**Variable Substitution:**
- Flux Kustomizations support variable substitution via `postBuild.substituteFrom`
- Variables come from:
  - `cluster-settings` ConfigMap (shared cluster config)
  - `cluster-secrets` Secret (shared sensitive config)
  - `cluster-env-secrets` Secret (environment-specific overrides, optional)
- Enable substitution by adding label: `substitution.flux.home.arpa/enabled: "true"` to Kustomizations
- The infrastructure and apps Kustomizations use patches to automatically inject substitution config into child Kustomizations with this label

## Working with Applications

### Adding a New Application

1. **Create directory structure:**
   ```
   cluster/apps/<category>/<app-name>/
   ├── ks.yaml              # Flux Kustomization (points to ./app)
   └── app/
       ├── kustomization.yaml
       ├── helmrelease.yaml  # or helm-release.yaml
       └── secret.sops.yaml  # if secrets needed
   ```

2. **Create the Flux Kustomization** (`ks.yaml`):
   ```yaml
   apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
   kind: Kustomization
   metadata:
     name: cluster-<category>-<app-name>
     namespace: flux-system
     labels:
       substitution.flux.home.arpa/enabled: "true"  # if using variables
   spec:
     path: ./cluster/apps/<category>/<app-name>/app
     prune: true
     sourceRef:
       kind: GitRepository
       name: flux-system
     healthChecks:
       - apiVersion: helm.toolkit.fluxcd.io/v2beta1
         kind: HelmRelease
         name: <app-name>
         namespace: <namespace>
     interval: 30m
     retryInterval: 1m
     timeout: 5m
   ```

3. **Add to parent kustomization:**
   Edit `cluster/apps/kustomization.yaml` and add:
   ```yaml
   - ./<category>/<app-name>/ks.yaml
   ```

### Helm Chart Sources

**Two patterns for OCIRepository sources:**

1. **Centralized in `flux-system` namespace** (referenced cross-namespace):
   - Define in `cluster/infra/sources/<name>.yaml`
   - Add to `cluster/infra/sources/kustomization.yaml`
   - Reference from HelmRelease: `namespace: flux-system`

2. **Co-located with HelmRelease** (same namespace):
   - Define OCIRepository in same file as HelmRelease
   - Both resources in app's namespace (e.g., `monitoring`)

For HelmRepository sources, always define centrally in `cluster/infra/sources/`.

### Secrets and Variables

**Creating encrypted secrets:**
```bash
# Ensure SOPS_AGE_KEY_FILE environment variable is set
sops cluster/apps/<category>/<app>/app/secret.sops.yaml
```

**Using substitution variables in manifests:**
```yaml
env:
  DOMAIN: ${SECRET_DEV_DOMAIN}  # From cluster-secrets or cluster-env-secrets
  TIMEZONE: ${TIMEZONE:=Europe/Berlin}  # From cluster-settings, with default
```

Remember to add the label `substitution.flux.home.arpa/enabled: "true"` to the Kustomization.

## Working with CRDs

CRDs are managed separately to avoid dependency issues:
- Located in `cluster/crds/<operator-name>/`
- CRD Kustomizations set `prune: false` to prevent accidental deletion
- CRDs can be sourced from:
  - External GitRepository (e.g., CloudNative-PG pulls directly from upstream)
  - Local files in the repo

## Validation and Testing

**Check YAML syntax:**
```bash
# Find all YAML files
find cluster environments -name '*.yaml' -type f

# Use Python/YAML validator if available
python3 -c "import yaml; yaml.safe_load(open('file.yaml'))"
```

**Preview encrypted secrets:**
```bash
sops -d cluster/apps/<path>/secret.sops.yaml
```

**Validate Flux Kustomizations:**
```bash
# If kustomize is installed
kustomize build cluster/apps/<app>/app

# Check for referenced resources
grep -r "kind: " cluster/apps/<app>/
```

## Common Patterns

**Namespace creation:**
Namespaces are centrally managed in `cluster/infra/namespaces/namespaces.yaml`

**Storage classes:**
Defined in `cluster/infra/storageclasses/`

**HelmRelease with health checks:**
```yaml
spec:
  install:
    remediation:
      retries: -1  # Infinite retries on install
  upgrade:
    cleanupOnFail: true
    remediation:
      retries: 3
```

**Using environment-specific config:**
Environment-specific secrets go in `environments/<env>/config/env-secrets.sops.yaml` and are available as `cluster-env-secrets` Secret for substitution.

## Repository Conventions

- Flux Kustomization files are named `ks.yaml`
- App-specific kustomizations are in `app/kustomization.yaml`
- Naming convention for Flux Kustomizations: `cluster-<category>-<app-name>`
- HelmRelease files can be either `helmrelease.yaml` or `helm-release.yaml` (both are used)
- Comments in YAML use `#` and indentation for commented resources maintains structure
