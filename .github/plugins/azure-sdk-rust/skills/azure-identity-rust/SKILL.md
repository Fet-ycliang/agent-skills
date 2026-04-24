---
name: azure-identity-rust
description: |
  Azure Identity library for Rust. Authentication for Azure SDK clients using Microsoft Entra ID.
  Use for DeveloperToolsCredential, ManagedIdentityCredential, ClientSecretCredential, and token-based auth with Azure services.
  Triggers: "azure identity rust", "DeveloperToolsCredential", "authentication rust", "managed identity rust", "credential rust".
license: MIT
metadata:
  author: Microsoft
  package: azure_identity
---

# Azure Identity library for Rust

Authentication for Azure SDK clients using Microsoft Entra ID.

Use this skill when:

- An app needs to authenticate to Azure services from Rust
- You need `DeveloperToolsCredential` for local development
- You need `ManagedIdentityCredential` for Azure-hosted workloads
- You need service principal auth with secret or certificate

> **IMPORTANT:** Only use official `azure_*` crates published by the [azure-sdk](https://crates.io/users/azure-sdk) crates.io user. Do NOT use the deprecated `azure_sdk_*` crates (MindFlavor/AzureSDKForRust) or community crates (e.g., `azure_storage`, `azure_storage_blobs`). Official crates use underscores in names and none have version 0.21.0.

> **Note:** The Rust SDK does not support `DefaultAzureCredential`. Use `DeveloperToolsCredential` for local development and `ManagedIdentityCredential` for production.

## Installation

```sh
cargo add azure_identity tokio
```

> **Do not** add `azure_core` directly to `Cargo.toml`. It is re-exported by service crates.

## Environment Variables

```bash
# Service Principal (for CI/production)
AZURE_TENANT_ID=<your-tenant-id>
AZURE_CLIENT_ID=<your-client-id>
AZURE_CLIENT_SECRET=<your-client-secret>

# User-assigned Managed Identity (optional)
AZURE_CLIENT_ID=<managed-identity-client-id>
```

## Authentication

### DeveloperToolsCredential (Local Development)

Tries Azure CLI then Azure Developer CLI:

```rust
use azure_identity::DeveloperToolsCredential;
use azure_security_keyvault_secrets::SecretClient;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Local dev: DeveloperToolsCredential. Production: use ManagedIdentityCredential.
    let credential = DeveloperToolsCredential::new(None)?;
    let client = SecretClient::new(
        "https://<your-key-vault-name>.vault.azure.net/",
        credential.clone(),
        None,
    )?;

    let secret = client.get_secret("secret-name", None).await?.into_model()?;
    println!("Secret: {:?}", secret.value);

    Ok(())
}
```

Ensure you are logged in first:

```sh
az login
```

#### Credential Chain Order

| Order | Credential                  | Login Command    |
| ----- | --------------------------- | ---------------- |
| 1     | AzureCliCredential          | `az login`       |
| 2     | AzureDeveloperCliCredential | `azd auth login` |

### ManagedIdentityCredential (Production)

For Azure-hosted resources (VMs, App Service, Functions, AKS):

```rust
use azure_identity::ManagedIdentityCredential;

// System-assigned managed identity
let credential = ManagedIdentityCredential::new(None)?;

// User-assigned managed identity
let options = ManagedIdentityCredentialOptions {
    client_id: Some("<user-assigned-mi-client-id>".into()),
    ..Default::default()
};
let credential = ManagedIdentityCredential::new(Some(options))?;
```

### ClientSecretCredential (Service Principal)

For CI/CD pipelines and service accounts:

```rust
use azure_identity::ClientSecretCredential;

let credential = ClientSecretCredential::new(
    "<tenant-id>",
    "<client-id>",
    "<client-secret>",
    None,
)?;
```

## Credential Types

| Credential                    | Use Case                               |
| ----------------------------- | -------------------------------------- |
| `DeveloperToolsCredential`    | Local development — tries CLI tools    |
| `ManagedIdentityCredential`   | Azure VMs, App Service, Functions, AKS |
| `WorkloadIdentityCredential`  | Kubernetes workload identity           |
| `ClientSecretCredential`      | Service principal with secret          |
| `ClientCertificateCredential` | Service principal with certificate     |
| `AzureCliCredential`          | Direct Azure CLI auth                  |
| `AzureDeveloperCliCredential` | Direct azd CLI auth                    |
| `AzurePipelinesCredential`    | Azure Pipelines service connection     |
| `ClientAssertionCredential`   | Custom assertions (federated identity) |

## Best Practices

1. **Use `DeveloperToolsCredential`** for local dev, **`ManagedIdentityCredential`** for production
2. **Never hardcode credentials** — use environment variables for service principals
3. **Clone credentials** — pass `credential.clone()` when constructing multiple clients
4. **Clients are thread-safe** — reuse clients across threads
5. **Assign RBAC roles** — ensure the identity has appropriate roles for the target service

## Reference Links

| Resource      | Link                                    |
| ------------- | --------------------------------------- |
| API Reference | https://docs.rs/azure_identity          |
| crates.io     | https://crates.io/crates/azure_identity |
