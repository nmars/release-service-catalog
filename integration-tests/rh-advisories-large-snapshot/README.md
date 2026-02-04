# rh-advisories-large-snapshot test

## Overview

This test validates the rh-advisories pipeline with a large snapshot (~200 components) to ensure it can handle production-scale workloads. The test uses pre-built container images to skip the build phase and focus on advisory creation and processing at scale.

### Infrastructure Architecture

**IMPORTANT: Understanding the Two-Cluster Setup**

This test follows the standard e2e test architecture with execution split across two clusters:

1. **PaC Execution Cluster**: Production (stone-prd-rh01)
   - Where: `rhtap-release-2-tenant` namespace
   - Purpose: PipelineRun execution controlled by PaC
   - Contains: The test orchestration and kubeconfig-secret

2. **Test Target Cluster**: Staging (stone-stg-rh01) 
   - Where: `dev-release-team-tenant` / `managed-release-team-tenant` namespaces
   - Purpose: Where actual test resources are created and release pipelines run
   - Uses: Staging Pyxis server and staging signing configuration
   - Connection: Via kubeconfig-secret from production cluster

#### Why PaC Runs on Production (But Test is Still Safe)

**The Constraint:**
The `release-service-catalog` repository and its PaC configuration live on the **production cluster** 
(stone-prd-rh01). This is a Konflux/RHTAP organizational decision - all catalog repositories are 
hosted on production infrastructure for stability and availability.

**Why It Must Run on Production:**
- PaC (Pipelines as Code) executes in the cluster where the Git repository is registered
- The repository cannot be moved to staging without significant infrastructure changes
- This affects ALL e2e tests in this repository, not just this one

**What Changed (This Test vs. Reality):**
- **What it looks like**: PaC runs on production → sounds scary! 🚨
- **What actually happens**: PaC on production is just the **orchestration layer**
  - The PipelineRun pod runs on production (minimal resource usage)
  - The test script inside that pod connects to **staging** via kubeconfig
  - All actual work (resources, pipelines, Pyxis calls) happens on **staging**

**Analogy:** Think of it like a remote control:
- Remote control (PaC): Lives on your production shelf
- The device being controlled (test execution): Runs in your staging room
- No production TV gets changed when you press buttons!

**Why This Separation?**
- The `release-service-catalog` repository lives on production cluster
- PaC triggers run where the repository is hosted (production)
- Tests safely execute against staging infrastructure via kubeconfig
- This prevents any accidental impact to production services

**Safety Guarantees:**
- ✅ No production Pyxis server interaction (uses stage Pyxis)
- ✅ No production signing operations (uses staging signing config)
- ✅ No production cluster resource creation (creates in staging)
- ✅ Safe to run repeatedly without external side effects

**Precautions & Expectations:**
- **Resource Cleanup**: Test automatically cleans up resources on completion/cancellation
  - Deletes GitHub test repositories
  - Removes Kubernetes resources from staging cluster
  - Cleanup runs even on test failure (via trap handlers)
- **Rate Limits**: Test creates ~200 components, respects staging cluster quotas
  - May consume significant staging cluster resources during 4-8 hour run
  - Does NOT impact production rate limits (staging Pyxis, staging signing)
- **Concurrent Runs**: Multiple instances can run simultaneously
  - Each test generates unique resource names (UUID-based)
  - No conflicts between concurrent test executions
- **Test Duration**: 4-8 hours is normal for ~200 components
  - Not a failure - this is expected for large snapshot processing
  - PipelineRun timeout configured to 8h0m0s
- **Production Cluster Impact**: Minimal (only PipelineRun orchestration pod)
  - No production services called
  - No production cluster resources created beyond the PipelineRun itself

### Environment Terminology Clarification

**IMPORTANT:** This test uses different terminology for different components:

| Component | Value | Configurable? | Meaning |
|-----------|-------|---------------|---------|
| **OpenShift Cluster** | `stone-stg-rh01` | ❌ Hardcoded | Physical cluster where test runs |
| **Pyxis Server** | `stage` | ❌ Hardcoded | Pyxis staging API endpoint |
| **Signing Config** | `staging-redhatbeta2` | ❌ Hardcoded | Staging signing keys/certificates |

**Key Points:**
- **All configuration hardcoded:** Cluster, Pyxis, and signing are fixed to staging
- **Consistent with other tests:** Matches the pattern of all e2e tests in this repository
- **Production not supported:** This test only works on staging infrastructure

## Setup

### Dependencies
* GitHub repo: https://github.com/hacbs-release-tests/e2e-base
* GitHub personal access token (classic) for above repo with **admin:repo_hook**, **delete_repo**, **repo** scopes.
* The password to the vault files. (Contact a member of the Release team should you want to run this
  test suite.)
* Access to the target cluster and tenant and managed namespaces
  * **Cluster:** stg-rh01 (staging cluster)
  * **PaC Runs:** Execute in `rhtap-release-2-tenant` (triggered by `/test-large-snapshot` comment)
  * **Local Runs:** Default to `dev-release-team-tenant` (can be overridden via `tenant_namespace` variable)
  * **Managed Namespace:** Both use `managed-release-team-tenant` for release pipelines

**IMPORTANT:** The namespace difference between PaC and local runs is intentional:
- PaC runs in `rhtap-release-2-tenant` where the PipelineRun is created
- Local test runs default to `dev-release-team-tenant` for isolation
- Both namespaces must have the required secrets configured

### Required Environment Variables
- GITHUB_TOKEN
  - The GitHub personal access token needed for repo operations
  - The repo in question can be located in [test.env](test.env)
- VAULT_PASSWORD_FILE
  - This is the path to a file that contains the ansible vault
    password needed to decrypt the secrets needed for testing.
- RELEASE_CATALOG_GIT_URL
  - The release service catalog URL to use in the RPA
  - This is provided when testing PRs
- RELEASE_CATALOG_GIT_REVISION
  - The release service catalog revision to use in the RPA
  - This is provided when testing PRs
### Optional Environment Variables

#### Additional Configuration

**Note:** This test uses hardcoded staging configuration:
- **Pyxis Server:** `stage` (staging Pyxis API)
- **Signing Config:** `staging-redhatbeta2` (staging signing keys)
- **Cluster:** `stg-rh01` (staging cluster)
- **Namespaces:** `dev-release-team-tenant` (local) / `rhtap-release-2-tenant` (PaC)

This is consistent with all other e2e tests in this repository.

- **KUBECONFIG**
  - The KUBECONFIG file to used to login to the target cluster
  - This is provided when testing PRs
  
- **CONSOLE_URL**
  - OpenShift console URL for generating PipelineRun links
  - **Auto-detected** from current cluster (recommended - always correct!)
  - Detection priority:
    1. `CONSOLE_URL` env var (explicit override)
    2. `oc whoami --show-console` (dynamic from current cluster)
    3. PaC ConfigMap `custom-console-url` (when running in PaC)
  - **Benefit**: Dynamic detection ensures generated links always point to the correct cluster console
  - **Consistent**: Same approach as all other e2e tests (no hardcoded URLs)

### Test Properties
#### [test.env](test.env)
- This file contains resource names and configuration values needed for testing.
- This test creates a large snapshot with pre-built components to test advisory creation at scale.
- The component count is configurable via `LARGE_SNAPSHOT_COMPONENT_COUNT` (default: 200).
#### [test.sh](test.sh)
- This file contains specific variables and functions needed for the test.
- Overrides standard functions to skip builds and use pre-built images.
### Test Functions
#### [lib/test-functions.sh](../lib/test-functions.sh)
- This file contains re-usable functions for tests
### Secrets
- Secrets needed for testing are stored in ansible vault files.
  - [vault/managed-secrets.yaml](vault/managed-secrets.yaml)
  - [vault/tenant-secrets.yaml](vault/tenant-secrets.yaml)
- The secrets required are contained in the files above.

### Running the test

#### Via Pull Request (Recommended)

This test can be triggered manually via PR comment:

```
/test-large-snapshot
```

Comment on any PR in the `release-service-catalog` repository. The PaC configuration 
([.tekton/rh-advisories-large-snapshot.yaml](../../.tekton/rh-advisories-large-snapshot.yaml)) 
will trigger the Tekton pipeline automatically.

**No Special Approval Required:** This test safely runs against staging infrastructure and does not 
require additional approvals or confirmations.

**Expected Duration:** 4-8 hours due to processing ~200 components.

**What Happens:**
1. PipelineRun starts in `rhtap-release-2-tenant` on production cluster
2. Test connects to staging cluster via kubeconfig-secret
3. Creates test resources on staging cluster
4. Executes release pipeline with staging Pyxis/signing
5. Validates advisory creation and processing at scale

#### Local Testing

For local development and debugging:

```shell
../run-test.sh rh-advisories-large-snapshot
```

**Prerequisites:**
- KUBECONFIG with access to staging cluster (stone-stg-rh01)
- Required environment variables (see "Required Environment Variables" section)
- Access to staging cluster namespaces

#### Namespace Configuration for Local Runs

By default, local test runs use `dev-release-team-tenant` as the tenant namespace (see [test.env](test.env)).

To override the tenant namespace for local testing (e.g., to match PaC behavior):

```shell
export tenant_namespace=rhtap-release-2-tenant
../run-test.sh rh-advisories-large-snapshot
```

**Prerequisites for using a different tenant namespace:**
- The namespace must exist on the target cluster
- Required secrets must be configured: `vault-password-secret`, `github-token-secret`, `kubeconfig-secret`
- Your ServiceAccount must have permissions to create resources in both tenant and managed namespaces
- The ReleasePlanAdmission must be configured in the managed namespace

- **Approval from Release Service team** for non-standard environments

**Important:** Pyxis and signing configuration are **hardcoded to staging** and cannot be changed. This test always uses:
- Pyxis: `stage` (staging Pyxis API)
- Signing: `staging-redhatbeta2` (staging signing keys)

This ensures safe testing and is consistent with all other e2e tests in this repository.

### Debugging

There is a `--skip-cleanup` option to the script in the event that you want to examine the resources
after a test has ended.

### Maintenance

- Should you require to add or update a secret, follow these steps:
```shell
ansible-vault decrypt vault/tenant-secrets.yaml --output "/tmp/tenant-secrets.yaml" --vault-password-file <vault password file>
```

```shell
vi /tmp/tenant-secrets.yaml
```

```shell
ansible-vault encrypt /tmp/tenant-secrets.yaml --output "vault/tenant-secrets.yaml" --vault-password-file <vault password file>
```

```shell
rm /tmp/tenant-secrets.yaml
```
