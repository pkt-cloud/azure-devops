# Azure Web App to Key Vault Private Connectivity Setup

## Purpose

This document describes how to establish and verify connectivity between an **Azure Web App** and an **Azure Key Vault** using:

- App Service VNet Integration
- Key Vault Private Endpoint
- Private DNS
- App Service System-Assigned Managed Identity
- Azure RBAC
- `Key Vault Secrets User` role

All environment-specific names and IDs are represented using placeholders.

---

## Architecture

```text
Azure Web App
     │
     ├── System-Assigned Managed Identity
     │            │
     │            └── Key Vault Secrets User
     │
     └── VNet Integration
                  │
                  ▼
             Private DNS
                  │
                  ▼
      privatelink.vaultcore.azure.net
                  │
                  ▼
        Key Vault Private Endpoint
                  │
                  ▼
            Azure Key Vault
```

Two separate requirements must be satisfied:

```text
Network:
Web App → VNet Integration → Private DNS → Private Endpoint → Key Vault

Authorization:
Web App Managed Identity → Key Vault RBAC → Key Vault Secrets User
```

---

## Prerequisites

Replace the following placeholders with environment-specific values:

```bash
APP_RG="<WEBAPP_RESOURCE_GROUP>"
APP_NAME="<WEBAPP_NAME>"

KV_RG="<KEYVAULT_RESOURCE_GROUP>"
KV_NAME="<KEYVAULT_NAME>"
```

Example naming format:

```text
APP_RG   = <app-resource-group>
APP_NAME = <environment-webapp>

KV_RG    = <keyvault-resource-group>
KV_NAME  = <environment-keyvault>
```

Do not store actual subscription IDs, principal IDs, passwords, secrets, or other sensitive values in documentation or source control.

---

# Step 1 — Verify Web App Managed Identity

Check whether System-Assigned Managed Identity is enabled:

```bash
az webapp identity show \
  --resource-group $APP_RG \
  --name $APP_NAME \
  --query "{type:type, principalId:principalId}" \
  -o table
```

Expected:

```text
Type              PrincipalId
----------------  ------------------------------------
SystemAssigned    <MANAGED_IDENTITY_PRINCIPAL_ID>
```

If Managed Identity is not enabled:

```bash
az webapp identity assign \
  --resource-group $APP_RG \
  --name $APP_NAME
```

---

# Step 2 — Capture the Web App Principal ID

Retrieve the Managed Identity Principal ID:

```bash
PRINCIPAL_ID=$(az webapp identity show \
  --resource-group $APP_RG \
  --name $APP_NAME \
  --query principalId \
  -o tsv)
```

Optional verification:

```bash
echo $PRINCIPAL_ID
```

Expected:

```text
<MANAGED_IDENTITY_PRINCIPAL_ID>
```

Do not place the actual Principal ID in permanent documentation unless required.

---

# Step 3 — Verify Key Vault Uses Azure RBAC

Run:

```bash
az keyvault show \
  --name $KV_NAME \
  --resource-group $KV_RG \
  --query properties.enableRbacAuthorization \
  -o tsv
```

Expected:

```text
true
```

If the result is:

```text
false
```

the Key Vault is using the legacy **Access Policy** authorization model. The RBAC role assignment described below will not provide Key Vault data-plane permission until Azure RBAC is being used.

---

# Step 4 — Get the Key Vault Resource ID

```bash
KV_ID=$(az keyvault show \
  --name $KV_NAME \
  --resource-group $KV_RG \
  --query id \
  -o tsv)
```

Optional verification:

```bash
echo $KV_ID
```

Expected format:

```text
/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP>/providers/Microsoft.KeyVault/vaults/<KEYVAULT_NAME>
```

---

# Step 5 — Assign Key Vault Secrets User

Assign the Web App's Managed Identity permission to read Key Vault secrets:

```bash
az role assignment create \
  --assignee-object-id $PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope $KV_ID
```

The resulting relationship is:

```text
<Web App Managed Identity>
           │
           ▼
Key Vault Secrets User
           │
           ▼
      <Key Vault>
```

`Key Vault Secrets User` is appropriate when the application only needs to retrieve secret values.

Do not assign broader roles such as `Key Vault Administrator` or `Key Vault Secrets Officer` unless the application genuinely needs to create, modify, or delete secrets.

---

# Step 6 — Verify the Role Assignment

Run:

```bash
az role assignment list \
  --assignee $PRINCIPAL_ID \
  --scope $KV_ID \
  --include-inherited \
  --query "[].{Role:roleDefinitionName,Scope:scope}" \
  -o table
```

Expected:

```text
Role                      Scope
------------------------  ---------------------------------------------
Key Vault Secrets User    /subscriptions/<MASKED>/.../<KEYVAULT_NAME>
```

This confirms the Web App's Managed Identity has Key Vault secret-read authorization.

---

# Step 7 — Verify Web App VNet Integration

In Azure Portal:

```text
Azure Web App
→ Networking
→ Virtual network integration
```

Verify the Web App is connected to the intended:

```text
VNet   : <ENVIRONMENT_VNET>
Subnet : <APP_SERVICE_INTEGRATION_SUBNET>
```

The integration subnet is what enables the Web App to reach private resources through the VNet.

---

# Step 8 — Verify Key Vault Private Endpoint

Navigate to:

```text
Key Vault
→ Networking
→ Private endpoint connections
```

Verify:

```text
Private Endpoint : <KEYVAULT_PRIVATE_ENDPOINT>
Connection State : Approved
```

The Key Vault Private Endpoint should reside in the intended Private Endpoint subnet.

Example architecture:

```text
<VNet>
 ├── <APP-INTEGRATION-SUBNET>
 │       └── Azure Web App VNet Integration
 │
 └── <PRIVATE-ENDPOINT-SUBNET>
         └── Key Vault Private Endpoint
```

---

# Step 9 — Verify Private DNS

The required Private DNS zone for Azure Key Vault is:

```text
privatelink.vaultcore.azure.net
```

Verify that the zone is linked to the VNet used by the Web App, or that the organization's custom DNS infrastructure correctly forwards this namespace to Azure Private DNS.

The expected DNS flow is:

```text
<KEYVAULT_NAME>.vault.azure.net
              │
              ▼
<KEYVAULT_NAME>.privatelink.vaultcore.azure.net
              │
              ▼
          <PRIVATE_IP>
```

---

# Step 10 — Verify DNS From the Web App

Run this from the Web App's Kudu/diagnostic environment:

```powershell
Resolve-DnsName <KEYVAULT_NAME>.vault.azure.net
```

Alternatively:

```powershell
nslookup <KEYVAULT_NAME>.vault.azure.net
```

Expected:

```text
<KEYVAULT_NAME>.vault.azure.net
        ↓
<KEYVAULT_NAME>.privatelink.vaultcore.azure.net
        ↓
10.x.x.x
```

The final IP should be a **private IP address**.

If the hostname resolves to a public IP, investigate:

```text
VNet Integration
Private DNS zone
VNet DNS configuration
DNS zone VNet links
Corporate/custom DNS forwarding
Private Endpoint DNS zone group
```

Do not troubleshoot RBAC or application credentials until DNS is correct.

---

# Step 11 — Verify TCP Connectivity

From the Web App environment:

```powershell
Test-NetConnection <KEYVAULT_NAME>.vault.azure.net -Port 443
```

Expected:

```text
TcpTestSucceeded : True
```

This confirms the network path:

```text
Web App
   ↓
VNet Integration
   ↓
DNS
   ↓
Private Endpoint IP
   ↓
TCP 443
   ↓
Key Vault
```

---

# Step 12 — Test Managed Identity Token Generation

From the Web App/Kudu PowerShell environment:

```powershell
$resource = "https://vault.azure.net"

$tokenResponse = Invoke-RestMethod `
    -Uri "$env:IDENTITY_ENDPOINT?resource=$resource&api-version=2019-08-01" `
    -Headers @{
        "X-IDENTITY-HEADER" = $env:IDENTITY_HEADER
    }

$token = $tokenResponse.access_token
```

Do **not** print or log the access token.

Successful token generation confirms:

```text
Web App
   ↓
Managed Identity
   ↓
Microsoft Entra ID
   ↓
Access Token
```

---

# Step 13 — Test Actual Secret Retrieval

Use a non-sensitive test secret where possible.

```powershell
$kv = "https://<KEYVAULT_NAME>.vault.azure.net"
$secretName = "<TEST_SECRET_NAME>"

$response = Invoke-RestMethod `
    -Uri "$kv/secrets/$secretName?api-version=7.4" `
    -Headers @{
        Authorization = "Bearer $token"
    }
```

Do not print the secret value in shared logs.

A successful response proves:

```text
Web App
   ↓
Managed Identity
   ↓
Key Vault RBAC
   ↓
Private Network
   ↓
Key Vault
   ↓
Secret retrieval
```

---

# Troubleshooting Interpretation

| Result | Likely Area |
|---|---|
| DNS returns public IP | Private DNS/VNet/DNS configuration |
| DNS fails completely | DNS forwarding or VNet configuration |
| Private IP resolves but TCP 443 fails | NSG/routing/private endpoint/network |
| TCP 443 works but token generation fails | Web App Managed Identity |
| Token works but Key Vault returns `403` | Key Vault RBAC |
| Key Vault returns `404` | Secret name/version may be incorrect |
| Secret retrieval succeeds | End-to-end connectivity is working |

---

# Final Validation Checklist

- [ ] Web App System-Assigned Managed Identity is enabled.
- [ ] Managed Identity Principal ID is available.
- [ ] Key Vault uses Azure RBAC.
- [ ] Web App identity has `Key Vault Secrets User`.
- [ ] Web App has VNet Integration configured.
- [ ] Key Vault Private Endpoint status is `Approved`.
- [ ] `privatelink.vaultcore.azure.net` is correctly configured.
- [ ] Key Vault hostname resolves to a private IP from the Web App.
- [ ] TCP port `443` succeeds from the Web App.
- [ ] Managed Identity can obtain a Key Vault access token.
- [ ] Web App can retrieve a test secret successfully.

## Success Criteria

The Web App → Key Vault connection should only be considered fully established when all four layers pass:

```text
1. DNS            ✓  Key Vault resolves to private IP
2. Network        ✓  TCP 443 succeeds
3. Authentication ✓  Managed Identity obtains token
4. Authorization  ✓  Secret retrieval succeeds
```

The Azure RBAC role assignment establishes **authorization**. The **VNet Integration + Private DNS + Private Endpoint** establish the private network path. Both are required for a fully private Web App-to-Key Vault connection.