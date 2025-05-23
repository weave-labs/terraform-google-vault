# Vault Google Cloud Run Module

This is a Terraform module to deploy a [Vault](https://www.vaultproject.io/) instance on Google's [Cloud Run](https://cloud.google.com/run/) service. Vault is an open-source secrets management tool that generally is run in a high-availability (HA) cluster. This implementation is a single instance with auto-unseal and no HA support. Cloud Run is a way to easily run a container on Google Cloud without an orchestrator. This module makes use of the following Google Cloud resources:

- Google Cloud Run
- Google Cloud Storage
- Google Cloud Key Management Service

---

## Table of Contents

- [Getting Started](#getting-started)
- [Variables](#variables)
  - [`name`](#name)
  - [`location`](#location)
  - [`project`](#project)
  - [`vault_image`](#vault_image)
  - [`bucket_force_destroy`](#bucket_force_destroy-optional)
  - [`container_concurrency`](#container_concurrency-optional)
  - [`vpc_connector`](#vpc_connector-optional)
  - [`vault_ui`](#vault_ui-optional)
  - [`vault_api_addr`](#vault_api_addr-optional)
  - [`vault_kms_keyring_name`](#vault_kms_keyring_name-optional)
  - [`vault_kms_key_rotation`](#vault_kms_key_rotation-optional)
  - [`vault_kms_key_algorithm`](#vault_kms_key_algorithm-optional)
  - [`vault_kms_key_protection_level`](#vault_kms_key_protection_level-optional)
  - [`vault_service_account_id`](#vault_service_account_id-optional)
  - [`vault_storage_bucket_name`](#vault_storage_bucket_name-optional)
- [Security Concerns](#security-concerns)
- [Caveats](#caveats)
  - [Google Cloud Container Registry](#google-cloud-container-registry)

## Getting Started

To get started, a Google Cloud Project is needed. This should be created ahead of time or using Terraform, but is outside the scope of this module. This project ID is provided to the module invocation and a basic implementation would look like the following:

```hcl
provider "google" {}

data "google_client_config" "current" {}

module "vault" {
  providers = {
    google = google
  }

  source      = "git::https://github.com/mbrancato/terraform-google-vault.git"
  name        = "vault"
  project     = data.google_client_config.current.project
  location    = data.google_client_config.current.region
  vault_image = "us.gcr.io/${data.google_client_config.current.project}/vault:1.6.1"
}
```

After creating the resources, the Vault instance may be initialized.

Set the `VAULT_ADDR` environment variable. See [Vault URL](#vault-url).

```
$ export VAULT_ADDR=https://vault-jsn3uj5s1c-sg.a.run.app
```

Ensure the vault is operational (might take a minute or two), uninitialized and sealed.

```
$ vault status
Key                      Value
---                      -----
Recovery Seal Type       gcpckms
Initialized              false
Sealed                   true
Total Recovery Shares    0
Threshold                0
Unseal Progress          0/0
Unseal Nonce             n/a
Version                  n/a
HA Enabled               false
```

Initialize the vault.

```
$ vault operator init
Recovery Key 1: ...
Recovery Key 2: ...
Recovery Key 3: ...
Recovery Key 4: ...
Recovery Key 5: ...

Initial Root Token: s....

Success! Vault is initialized

Recovery key initialized with 5 key shares and a key threshold of 3. Please
securely distribute the key shares printed above.
```

From here, Vault is operational. Configure the auth methods needed and other settings. The Cloud Run Service may scale the container to zero, but the server configuration and unseal keys are configured. When restarting, the Vault should unseal itself automatically using the Google KMS. For more information on deploying Vault, read [Deploy Vault](https://learn.hashicorp.com/vault/getting-started/deploy).

## Variables

### `name`

- Application name.

### `location`

- Google location where resources are to be created.

### `project`

- Google project ID.

### `vault_image`

- Vault docker image.

### `bucket_force_destroy` (optional)

- CAUTION: Set force_destroy for Storage Bucket. This is where the vault data is stored. Setting this to true will allow terraform destroy to delete the bucket.
  - default - `false`

### `container_concurrency` (optional)

- Max number of connections per container instance.
  - default - `80`

### `vpc_connector` (optional)

- ID for the [Serverless VPC connector](https://cloud.google.com/vpc/docs/configure-serverless-vpc-access) to be used, if any, for private VPC access.
  - Creation of the connector is out of scope of this module, see [google_vpc_access_connector](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/vpc_access_connector).
  - default - `null`

### `vault_ui` (optional)

- Enable Vault UI.
  - default - `false`

### `vault_api_addr` (optional)

- Full HTTP endpoint of Vault Server if using a custom domain name. Leave blank otherwise.
  - default - `""`

### `vault_kms_keyring_name` (optional)

- Name of the Google KMS keyring to use.
  - default - `"${var.name}-${lower(random_id.vault.hex)}-kr"`

### `vault_kms_key_rotation` (optional)

- The period for KMS key rotation.
  - Note: key rotations will lead to multiple active KMS keys and will result in an increasing monthly bill. Setting to `null` should disable rotation (not recommended).
  - default - `"7776000s"` (90 days)

### `vault_kms_key_algorithm` (optional)

- The cryptographic algorithm to be used with the KMS key.
  - Specify a supported [CryptoKeyVersionAlgorithm](https://cloud.google.com/kms/docs/reference/rest/v1/CryptoKeyVersionAlgorithm) value.
  - default - `"GOOGLE_SYMMETRIC_ENCRYPTION"`

### `vault_kms_key_protection_level` (optional)

- The protection level to be used with the KMS key.
  - Specify the [protection level](https://cloud.google.com/kms/docs/algorithms#protection_levels) to be used (SOFTWARE, HSM, EXTERNAL).
  - default - `"SOFTWARE"`

### `vault_service_account_id` (optional)

- ID for the service account to be used. This is the part of the service account email before the `@` symbol.
  - default - `"vault-sa"`

### `vault_storage_bucket_name` (optional)

- Storage bucket name to be used.
  - default - `"${var.name}-${lower(random_id.vault.hex)}-bucket"`

## Security Concerns

The following things may be of concern from a security perspective:

- When not using a VPC connector, this is a publicly accessible Vault instance. Anyone with the DNS name can connect to it.
- By default, Vault is running on shared compute infrastructure. The [Google Terraform provider](https://github.com/hashicorp/terraform-provider-google) does not yet support Cloud Run on Anthos / GKE to deploy on single-tenant VMs.

## Caveats

**PLEASE READ**

### Google Cloud Container Registry

Cloud Run will only run containers hosted on `gcr.io` (GCR) and its subdomains. This means that the Vault container will need to be pushed to GCR in the Google Cloud Project. Terraform cannot currently create the container registry and it is automatically created using `docker push`. Read the [documentation](https://cloud.google.com/container-registry/docs/pushing-and-pulling) for more details on pushing containers to GCR.

A quick way to get Vault into GCR for a GCP project:

```
gcloud auth configure-docker
docker pull hashicorp/vault:latest
docker tag hashicorp/vault:1.6.1 gcr.io/{{ project_id }}/vault:1.6.1
docker push gcr.io/{{ project_id }}/vault:1.6.1
```

## Adding Github JWT Auth Role

### Phase 1: Configure Vault

These steps typically require Vault admin privileges. You can use the Vault UI or CLI (vault). I'll provide the CLI commands for clarity.

1. Enable the JWT Auth Method: If you haven't already enabled the JWT auth method, run:

```bash
vault auth enable jwt
```

(This only needs to be done once per Vault instance). Configure the JWT Auth Method for GitHub OIDC: This tells Vault how to find GitHub's public keys to verify the JWTs sent by GitHub Actions.

```bash
vault write auth/jwt/config \
    oidc_discovery_url="https://token.actions.githubusercontent.com" \
    bound_issuer="https://token.actions.githubusercontent.com"
```

- oidc_discovery_url: Points to GitHub's OIDC configuration endpoint.
- bound_issuer: Specifies the expected issuer claim in the JWTs from GitHub Actions.

2. Create a Vault Policy: Define permissions for what secrets this GitHub Action workflow can access.

- Create a policy file (e.g., weavelabs-io-policy.hcl):

```hcl
# Policy allowing read access to secrets for weavelabs.io deployment
path "kv/data/cloud-run/weavelabs-io" {
  capabilities = ["read"]
}

# Add other paths if needed
# path "other/secret/path" {
#   capabilities = ["read"]
# }
```

(Adjust the path to where you store your SECRET in Vault)

3. Write the policy to Vault:

```bash
vault policy write weavelabs-io-policy weavelabs-io-policy.hcl
```

4. Create a Vault Role: This role links the GitHub OIDC identity (based on repository, branch, etc.) to the Vault policy created above.

```bash
vault write auth/jwt/role/weavelabs-io-deploy \
    role_type="jwt" \
    user_claim="actor" \
    bound_claims_type="glob" \
    bound_claims='{
        "sub": "repo:weave-labs/weavelabs.io:ref:refs/heads/main",
        "repository": "weave-labs/weavelabs.io"
    }' \
    policies="weavelabs-io-policy" \
    ttl="15m" # How long the Vault token issued to the action is valid
```

- auth/jwt/role/weavelabs-io-deploy: Defines the role name (weavelabs-io-deploy). You'll use this name in your GitHub workflow secrets.
- role_type="jwt": Specifies the authentication type.
- user_claim="actor": Uses the GitHub Actions actor claim to identify the entity in Vault logs/audits.
- bound_claims_type="glob": Allows using glob patterns in bound_claims.
- bound_claims: Crucial for security. This defines which GitHub Actions runs are allowed to assume this role.
- "sub": "repo:weave-labs/weavelabs.io:ref:refs/heads/main": Restricts this role strictly to runs from the main branch (refs/heads/main) of the weave-labs/weavelabs.io - repository. Adjust if you deploy from other branches or tags.
- "repository": "weave-labs/weavelabs.io": An additional check that the token originated from this specific repository.
- policies="weavelabs-io-policy": Attaches the policy created in step 3.
- ttl="15m": Sets a short time-to-live for the Vault token granted to the GitHub Action.

### Phase 2: Configure GitHub Repository Secrets

Now, you need to tell your GitHub Actions workflow how to connect to Vault and which role to use.

1. Navigate to your Repository: Go to https://github.com/weave-labs/weavelabs.io. Go to Settings -> Secrets and variables -> Actions.
2. Click "New repository secret" and add the following:
   - Name: VAULT_ADDR Value: https://your.vault.address:8200 (Replace with the actual URL of your Vault server).
   - Name: VAULT_ROLE Value: weavelabs-io-deploy (This must match the role name you created in Vault Step 4).
