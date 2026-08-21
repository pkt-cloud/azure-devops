Existing Azure Web App → Key Vault → Azure SQL Connectivity Audit

Purpose

Before configuring a new environment, first reverse-engineer the existing working environment.

The goal is to understand exactly how the current Azure Web App connects privately to:

* Azure Key Vault
* Azure SQL Database

The audit should identify:

* App Service VNet Integration
* Integration subnet
* VNet DNS configuration
* Private DNS zones
* Private Endpoint configuration
* Private DNS Zone Groups
* Managed Identity
* Key Vault RBAC
* SQL connectivity configuration
* Route tables and NSGs, if applicable

All commands below are read-only.

⸻

1. Define Environment Variables

Run from Azure Cloud Shell using Bash.

WEBAPP_RG="<CURRENT_WEBAPP_RESOURCE_GROUP>"
WEBAPP_NAME="<CURRENT_WEBAPP_NAME>"
KV_RG="<CURRENT_KEYVAULT_RESOURCE_GROUP>"
KV_NAME="<CURRENT_KEYVAULT_NAME>"
SQL_RG="<CURRENT_SQL_RESOURCE_GROUP>"
SQL_SERVER="<CURRENT_SQL_SERVER_NAME>"

Do not store real subscription IDs, principal IDs, passwords, or secret values in documentation.

⸻

2. Confirm Azure Subscription

az account show \
  --query "{Subscription:name,SubscriptionId:id,TenantId:tenantId}" \
  -o table

Confirm that the CLI is connected to the subscription containing the current working environment.

⸻

3. Check Web App Basic Configuration

az webapp show \
  --resource-group $WEBAPP_RG \
  --name $WEBAPP_NAME \
  --query "{Name:name,Location:location,DefaultHostName:defaultHostName,PublicNetworkAccess:publicNetworkAccess,HttpsOnly:httpsOnly}" \
  -o json

Record:

Web App Name:
Location:
Public Network Access:
Default Host Name:

⸻

4. Check Web App VNet Integration

This is one of the most important checks.

az webapp vnet-integration list \
  --resource-group $WEBAPP_RG \
  --name $WEBAPP_NAME \
  -o json

Identify:

VNet:
Integration Subnet:
Subnet Resource ID:

Expected architecture:

Azure Web App
      │
      │ VNet Integration
      ▼
App Service Integration Subnet
      │
      ├── Key Vault Private Endpoint
      │
      └── SQL Private Endpoint

The Web App uses VNet Integration for outbound private connectivity.

A Web App Private Endpoint, if present, is primarily for private inbound connectivity to the Web App.

⸻

5. Inspect App Service Network Properties

Get the Web App resource ID:

WEBAPP_ID=$(az webapp show \
  --resource-group $WEBAPP_RG \
  --name $WEBAPP_NAME \
  --query id \
  -o tsv)

Then inspect network-related properties:

az resource show \
  --ids "$WEBAPP_ID" \
  --query "properties.{VNetSubnet:virtualNetworkSubnetId,VNetRouteAll:vnetRouteAllEnabled,DNS:dnsConfiguration}" \
  -o json

Record:

VNet Subnet:
Route All Enabled:
DNS Configuration:

⸻

6. Check App Service DNS Overrides

Check only relevant networking settings.

az webapp config appsettings list \
  --resource-group $WEBAPP_RG \
  --name $WEBAPP_NAME \
  --query "[?name=='WEBSITE_DNS_SERVER' || name=='WEBSITE_DNS_ALT_SERVER' || name=='WEBSITE_VNET_ROUTE_ALL'].{Setting:name,Value:value}" \
  -o table

Possible settings include:

WEBSITE_DNS_SERVER
WEBSITE_DNS_ALT_SERVER
WEBSITE_VNET_ROUTE_ALL

If no values are returned, the Web App may be relying on the VNet’s normal DNS configuration.

⸻

7. Check Web App Managed Identity

az webapp identity show \
  --resource-group $WEBAPP_RG \
  --name $WEBAPP_NAME \
  --query "{Type:type,PrincipalId:principalId,TenantId:tenantId}" \
  -o json

Expected:

Type: SystemAssigned
PrincipalId: <MASKED>
TenantId: <MASKED>

Store the Principal ID temporarily:

PRINCIPAL_ID=$(az webapp identity show \
  --resource-group $WEBAPP_RG \
  --name $WEBAPP_NAME \
  --query principalId \
  -o tsv)

Do not expose the Principal ID unnecessarily in documentation.

⸻

8. Check Key Vault Configuration

az keyvault show \
  --name $KV_NAME \
  --resource-group $KV_RG \
  --query "{Name:name,Location:location,RBAC:properties.enableRbacAuthorization,PublicNetworkAccess:properties.publicNetworkAccess,DefaultNetworkAction:properties.networkAcls.defaultAction}" \
  -o json

Record:

RBAC Enabled:
Public Network Access:
Default Network Action:

For Azure RBAC-based Key Vault access, expect:

RBAC: true

⸻

9. Get Key Vault Resource ID

KV_ID=$(az keyvault show \
  --name $KV_NAME \
  --resource-group $KV_RG \
  --query id \
  -o tsv)

⸻

10. Check Web App → Key Vault RBAC

Check only the Web App Managed Identity’s permissions against the Key Vault.

az role assignment list \
  --assignee $PRINCIPAL_ID \
  --scope $KV_ID \
  --include-inherited \
  --query "[].{Role:roleDefinitionName,Scope:scope}" \
  -o table

For an application that only reads secrets, expect:

Key Vault Secrets User

Architecture:

Web App
   │
   └── System-Assigned Managed Identity
                │
                ▼
       Key Vault Secrets User
                │
                ▼
            Key Vault

This role controls authorization.

It is independent of DNS and private network connectivity.

⸻

11. Check Azure SQL Server Configuration

az sql server show \
  --resource-group $SQL_RG \
  --name $SQL_SERVER \
  --query "{Name:name,FQDN:fullyQualifiedDomainName,Location:location,PublicNetworkAccess:publicNetworkAccess,MinimumTLS:minimalTlsVersion}" \
  -o json

Record:

SQL Server:
FQDN:
Public Network Access:
TLS Version:

The application should normally connect using:

<SQL_SERVER>.database.windows.net

and not the private endpoint IP directly.

⸻

12. Discover Existing Private Endpoints

Run:

az network private-endpoint list \
  --query "[].{PE:name,ResourceGroup:resourceGroup,Target:privateLinkServiceConnections[0].privateLinkServiceId,Group:privateLinkServiceConnections[0].groupIds[0],Status:privateLinkServiceConnections[0].privateLinkServiceConnectionState.status,Subnet:subnet.id}" \
  -o table

Identify private endpoints targeting:

Microsoft.KeyVault/vaults/<KEYVAULT_NAME>

and:

Microsoft.Sql/servers/<SQL_SERVER_NAME>

Record:

Resource	Private Endpoint	PE Resource Group	Subnet	Status
Key Vault	<KV_PE>	<PE_RG>	<SUBNET>	Approved
Azure SQL	<SQL_PE>	<PE_RG>	<SUBNET>	Approved

⸻

13. Inspect Key Vault Private Endpoint

az network private-endpoint show \
  --resource-group <KV_PRIVATE_ENDPOINT_RG> \
  --name <KV_PRIVATE_ENDPOINT_NAME> \
  -o json

Verify:

Target resource = Key Vault
Group ID = vault
Connection status = Approved
Subnet = expected PE subnet

⸻

14. Inspect SQL Private Endpoint

az network private-endpoint show \
  --resource-group <SQL_PRIVATE_ENDPOINT_RG> \
  --name <SQL_PRIVATE_ENDPOINT_NAME> \
  -o json

Verify:

Target resource = SQL Server
Group ID = sqlServer
Connection status = Approved
Subnet = expected PE subnet

⸻

15. Check Private Endpoint DNS Zone Groups

This is an important check.

Key Vault

az network private-endpoint dns-zone-group list \
  --resource-group <KV_PRIVATE_ENDPOINT_RG> \
  --endpoint-name <KV_PRIVATE_ENDPOINT_NAME> \
  -o json

Expected Private DNS zone:

privatelink.vaultcore.azure.net

Azure SQL

az network private-endpoint dns-zone-group list \
  --resource-group <SQL_PRIVATE_ENDPOINT_RG> \
  --endpoint-name <SQL_PRIVATE_ENDPOINT_NAME> \
  -o json

Expected Private DNS zone:

privatelink.database.windows.net

Expected relationships:

Key Vault PE
     │
     └── privatelink.vaultcore.azure.net
SQL PE
     │
     └── privatelink.database.windows.net

⸻

16. Locate Private DNS Zones

az network private-dns zone list \
  --query "[?name=='privatelink.vaultcore.azure.net' || name=='privatelink.database.windows.net'].{Zone:name,ResourceGroup:resourceGroup}" \
  -o table

Record:

Private DNS Zone	Resource Group
privatelink.vaultcore.azure.net	<DNS_RG>
privatelink.database.windows.net	<DNS_RG>

These zones may exist in a centralized networking resource group rather than the application’s resource group.

⸻

17. Check Key Vault Private DNS Record

az network private-dns record-set a list \
  --resource-group <DNS_RESOURCE_GROUP> \
  --zone-name privatelink.vaultcore.azure.net \
  -o table

Look for:

<KEYVAULT_NAME> → <PRIVATE_IP>

Example:

<KEYVAULT_NAME> → 10.x.x.x

⸻

18. Check SQL Private DNS Record

az network private-dns record-set a list \
  --resource-group <DNS_RESOURCE_GROUP> \
  --zone-name privatelink.database.windows.net \
  -o table

Look for:

<SQL_SERVER_NAME> → <PRIVATE_IP>

Example:

<SQL_SERVER_NAME> → 10.x.x.x

⸻

19. Check Private DNS VNet Links

Key Vault DNS Zone

az network private-dns link vnet list \
  --resource-group <DNS_RESOURCE_GROUP> \
  --zone-name privatelink.vaultcore.azure.net \
  -o table

SQL DNS Zone

az network private-dns link vnet list \
  --resource-group <DNS_RESOURCE_GROUP> \
  --zone-name privatelink.database.windows.net \
  -o table

Determine whether the Web App integration VNet is directly linked to the zones.

Expected design may be:

Web App Integration VNet
       │
       ├── privatelink.vaultcore.azure.net
       │
       └── privatelink.database.windows.net

If the VNet is not linked directly, investigate whether corporate/custom DNS forwards these namespaces to Azure Private DNS or Azure Private Resolver.

⸻

20. Check VNet DNS Configuration

Using the VNet discovered from the Web App’s VNet Integration:

az network vnet show \
  --resource-group <VNET_RESOURCE_GROUP> \
  --name <VNET_NAME> \
  --query "{Name:name,AddressSpace:addressSpace.addressPrefixes,DnsServers:dhcpOptions.dnsServers}" \
  -o json

If the result is:

"DnsServers": []

the VNet is using Azure-provided DNS.

If the result contains private IPs:

"DnsServers": [
  "10.x.x.x",
  "10.x.x.x"
]

the environment uses custom/corporate DNS.

In that situation, verify that those DNS servers can resolve or forward:

privatelink.vaultcore.azure.net
privatelink.database.windows.net

⸻

21. Inspect App Service Integration Subnet

az network vnet subnet show \
  --resource-group <VNET_RESOURCE_GROUP> \
  --vnet-name <VNET_NAME> \
  --name <APP_INTEGRATION_SUBNET> \
  --query "{Name:name,AddressPrefix:addressPrefix,Delegations:delegations[].serviceName,NSG:networkSecurityGroup.id,RouteTable:routeTable.id,PrivateEndpointPolicies:privateEndpointNetworkPolicies}" \
  -o json

The App Service integration subnet should normally show:

Microsoft.Web/serverFarms

under delegation.

Record:

Subnet:
Address Prefix:
Delegation:
NSG:
Route Table:

⸻

22. Check Route Table

If the integration subnet has a route table:

az network route-table show \
  --resource-group <ROUTE_TABLE_RESOURCE_GROUP> \
  --name <ROUTE_TABLE_NAME> \
  -o json

Then:

az network route-table route list \
  --resource-group <ROUTE_TABLE_RESOURCE_GROUP> \
  --route-table-name <ROUTE_TABLE_NAME> \
  -o table

Check whether traffic is routed through:

Azure Firewall
NVA
VPN
ExpressRoute
Internet
Virtual Network

⸻

23. Check Application Connection Configuration Safely

Do not display connection string values.

List only connection-string names and types:

az webapp config connection-string list \
  --resource-group $WEBAPP_RG \
  --name $WEBAPP_NAME \
  --query "[].{Name:name,Type:type}" \
  -o table

List potentially relevant application-setting names only:

az webapp config appsettings list \
  --resource-group $WEBAPP_RG \
  --name $WEBAPP_NAME \
  --query "[].name" \
  -o tsv | grep -Ei "sql|database|db|keyvault|vault|clientid|managed"

This helps determine whether the application is using settings such as:

KeyVaultURL
SQLConnectionString
DatabaseConnection
ClientID

without exposing secret values.

⸻

24. Runtime Validation From the Existing Web App

Configuration inspection should be followed by tests from the running Web App environment.

From Kudu/SSH/diagnostic console:

Key Vault DNS

Resolve-DnsName <KEYVAULT_NAME>.vault.azure.net

Expected:

<KEYVAULT_NAME>.vault.azure.net
        ↓
<KEYVAULT_NAME>.privatelink.vaultcore.azure.net
        ↓
10.x.x.x

SQL DNS

Resolve-DnsName <SQL_SERVER_NAME>.database.windows.net

Expected:

<SQL_SERVER_NAME>.database.windows.net
        ↓
<SQL_SERVER_NAME>.privatelink.database.windows.net
        ↓
10.x.x.x

⸻

25. Runtime TCP Validation

Key Vault

Test-NetConnection <KEYVAULT_NAME>.vault.azure.net -Port 443

Expected:

TcpTestSucceeded : True

Azure SQL

Test-NetConnection <SQL_SERVER_NAME>.database.windows.net -Port 1433

Expected:

TcpTestSucceeded : True

DNS and TCP tests do not require Key Vault RBAC.

RBAC is only required when the application attempts to access Key Vault data such as a secret.

⸻

Recommended Initial Audit Commands

Start with these commands before performing deeper analysis.

1. Web App VNet Integration

az webapp vnet-integration list \
  -g $WEBAPP_RG \
  -n $WEBAPP_NAME \
  -o json

2. Web App Managed Identity

az webapp identity show \
  -g $WEBAPP_RG \
  -n $WEBAPP_NAME \
  -o json

3. App Service DNS Overrides

az webapp config appsettings list \
  -g $WEBAPP_RG \
  -n $WEBAPP_NAME \
  --query "[?name=='WEBSITE_DNS_SERVER' || name=='WEBSITE_DNS_ALT_SERVER' || name=='WEBSITE_VNET_ROUTE_ALL'].{Setting:name,Value:value}" \
  -o table

4. Key Vault Security Configuration

az keyvault show \
  -g $KV_RG \
  -n $KV_NAME \
  --query "{RBAC:properties.enableRbacAuthorization,PublicNetworkAccess:properties.publicNetworkAccess,DefaultAction:properties.networkAcls.defaultAction}" \
  -o json

5. Private Endpoints

az network private-endpoint list \
  --query "[].{PE:name,RG:resourceGroup,Target:privateLinkServiceConnections[0].privateLinkServiceId,Group:privateLinkServiceConnections[0].groupIds[0],Status:privateLinkServiceConnections[0].privateLinkServiceConnectionState.status,Subnet:subnet.id}" \
  -o table

6. Relevant Private DNS Zones

az network private-dns zone list \
  --query "[?name=='privatelink.vaultcore.azure.net' || name=='privatelink.database.windows.net'].{Zone:name,RG:resourceGroup}" \
  -o table

⸻

Expected Existing Architecture

The audit should eventually allow the working environment to be documented as:

                       Azure Web App
                            │
              ┌─────────────┴─────────────┐
              │                           │
      Managed Identity              VNet Integration
              │                           │
              ▼                           ▼
 Key Vault Secrets User        App Integration Subnet
              │                           │
              │                  VNet / DNS configuration
              │                           │
              │                 ┌─────────┴─────────┐
              │                 │                   │
              ▼                 ▼                   ▼
          Key Vault        Key Vault PE          SQL PE
                               │                   │
                               ▼                   ▼
                    Private DNS Zone       Private DNS Zone
                    vaultcore.azure.net    database.windows.net
                               │                   │
                               ▼                   ▼
                         Private IP            Private IP
                               │                   │
                               ▼                   ▼
                         Key Vault            Azure SQL

⸻

Troubleshooting Order

Always investigate in this sequence:

1. Web App VNet Integration
          ↓
2. Integration VNet/Subnet
          ↓
3. VNet DNS configuration
          ↓
4. Private DNS zones / DNS forwarding
          ↓
5. Private Endpoint DNS Zone Groups
          ↓
6. DNS resolves to private IP
          ↓
7. TCP 443 / TCP 1433 connectivity
          ↓
8. Web App Managed Identity
          ↓
9. Key Vault RBAC
          ↓
10. Application-level SQL / Key Vault access

This prevents authentication issues from being confused with network or DNS problems.

⸻

Existing Environment Audit Checklist

* Correct Azure subscription confirmed.
* Web App VNet Integration identified.
* Integration VNet identified.
* Integration subnet identified.
* Subnet delegation identified.
* VNet DNS servers identified.
* App Service DNS overrides checked.
* Route All configuration checked.
* Route table checked.
* NSG identified.
* Web App Managed Identity identified.
* Key Vault RBAC mode checked.
* Web App → Key Vault role assignment checked.
* Key Vault Private Endpoint identified.
* SQL Private Endpoint identified.
* Both PE connections are Approved.
* Key Vault DNS Zone Group checked.
* SQL DNS Zone Group checked.
* privatelink.vaultcore.azure.net located.
* privatelink.database.windows.net located.
* DNS A records checked.
* Private DNS VNet links checked.
* Key Vault resolves to private IP from Web App.
* SQL resolves to private IP from Web App.
* Key Vault TCP 443 succeeds.
* SQL TCP 1433 succeeds.
* Application configuration method identified.
* End-to-end Key Vault access confirmed.
* End-to-end SQL access confirmed.

Objective

Do not reproduce the new environment based only on assumptions.

First establish a documented as-is architecture of the currently working environment. Once the current networking, DNS, Private Endpoint, identity, and RBAC relationships are known, the same proven pattern can be reproduced for the new environment with environment-specific resource names and IP ranges.