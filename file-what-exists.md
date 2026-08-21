Before touching the new environment, Azure CLI is ideal because it gives us the exact resource IDs, VNet integration, DNS configuration, private endpoints, DNS zone groups, identity and RBAC configuration.

Use Azure Cloud Shell – Bash for the commands below. All of these are read-only.

1. Set the existing environment variables

WEBAPP_RG="<CURRENT-WEBAPP-RG>"
WEBAPP_NAME="<CURRENT-WEBAPP-NAME>"
KV_RG="<CURRENT-KEYVAULT-RG>"
KV_NAME="<CURRENT-KEYVAULT-NAME>"
SQL_RG="<CURRENT-SQL-RG>"
SQL_SERVER="<CURRENT-SQL-SERVER-NAME>"

First confirm you’re in the right subscription:

az account show \
  --query "{Subscription:name,SubscriptionId:id,TenantId:tenantId}" \
  -o table

⸻

2. Inspect the current Web App

az webapp show \
  --resource-group $WEBAPP_RG \
  --name $WEBAPP_NAME \
  --query "{Name:name,Location:location,DefaultHostName:defaultHostName,PublicNetworkAccess:publicNetworkAccess,HttpsOnly:httpsOnly}" \
  -o json

Most importantly, check its VNet Integration:

az webapp vnet-integration list \
  --resource-group $WEBAPP_RG \
  --name $WEBAPP_NAME \
  -o json

This is one of the most important outputs. It tells us exactly which VNet/subnet the working Web App is using. az webapp vnet-integration list is the supported CLI command for inspecting App Service VNet integration. 

Also inspect the underlying App Service network properties:

WEBAPP_ID=$(az webapp show \
  -g $WEBAPP_RG \
  -n $WEBAPP_NAME \
  --query id -o tsv)
az resource show \
  --ids "$WEBAPP_ID" \
  --query "properties.{VNetSubnet:virtualNetworkSubnetId,VNetRouteAll:vnetRouteAllEnabled,DNS:dnsConfiguration}" \
  -o json

This is useful because earlier Azure architectures sometimes use vnetRouteAllEnabled, while newer configurations can also expose routing/DNS through site properties.

⸻

3. Check whether any special DNS settings were added

This is especially important given the DNS troubleshooting we did previously.

Run:

az webapp config appsettings list \
  -g $WEBAPP_RG \
  -n $WEBAPP_NAME \
  --query "[?name=='WEBSITE_DNS_SERVER' || name=='WEBSITE_DNS_ALT_SERVER' || name=='WEBSITE_VNET_ROUTE_ALL'].{Setting:name,Value:value}" \
  -o table

This only retrieves the relevant network/DNS settings; it does not dump all application secrets.

Possible result:

Setting                    Value
-------------------------  -------------
WEBSITE_DNS_SERVER         <DNS-IP>
WEBSITE_DNS_ALT_SERVER     <DNS-IP>
WEBSITE_VNET_ROUTE_ALL     1

Or it may return nothing, which is also meaningful.

⸻

4. Inspect the Web App Managed Identity

az webapp identity show \
  --resource-group $WEBAPP_RG \
  --name $WEBAPP_NAME \
  --query "{Type:type,PrincipalId:principalId,TenantId:tenantId}" \
  -o json

Expected:

{
  "Type": "SystemAssigned",
  "PrincipalId": "<MASKED>",
  "TenantId": "<MASKED>"
}

Save the Principal ID for RBAC inspection:

PRINCIPAL_ID=$(az webapp identity show \
  -g $WEBAPP_RG \
  -n $WEBAPP_NAME \
  --query principalId \
  -o tsv)

⸻

5. Inspect the existing Key Vault configuration

az keyvault show \
  --name $KV_NAME \
  --resource-group $KV_RG \
  --query "{Name:name,Location:location,RBAC:properties.enableRbacAuthorization,PublicNetworkAccess:properties.publicNetworkAccess,DefaultNetworkAction:properties.networkAcls.defaultAction}" \
  -o json

This answers several important questions immediately:

Does it use Azure RBAC?
Is public access enabled or disabled?
Is networking default Allow or Deny?

Then get the resource ID:

KV_ID=$(az keyvault show \
  -n $KV_NAME \
  -g $KV_RG \
  --query id \
  -o tsv)
echo $KV_ID

⸻

6. Verify Web App → Key Vault RBAC

Now check specifically what the Web App Managed Identity has on the Key Vault:

az role assignment list \
  --assignee $PRINCIPAL_ID \
  --scope $KV_ID \
  --include-inherited \
  --query "[].{Role:roleDefinitionName,Scope:scope}" \
  -o table

This should reveal something like:

Role                      Scope
------------------------  --------------------------------
Key Vault Secrets User    /subscriptions/.../vaults/<KV>

This tells us exactly what role the working environment uses.

⸻

7. Inspect Azure SQL configuration

Get the SQL Server configuration:

az sql server show \
  --resource-group $SQL_RG \
  --name $SQL_SERVER \
  --query "{Name:name,FQDN:fullyQualifiedDomainName,Location:location,PublicNetworkAccess:publicNetworkAccess,MinimumTLS:minimalTlsVersion}" \
  -o json

Save its resource ID:

SQL_ID=$(az sql server show \
  -g $SQL_RG \
  -n $SQL_SERVER \
  --query id \
  -o tsv)
echo $SQL_ID

The important FQDN should remain something like:

<sql-server>.database.windows.net

even though traffic ultimately goes to a private IP.

⸻

8. Discover all relevant Private Endpoints

This command is very useful:

az network private-endpoint list \
  --query "[].{PE:name,ResourceGroup:resourceGroup,Target:privateLinkServiceConnections[0].privateLinkServiceId,Group:privateLinkServiceConnections[0].groupIds[0],Status:privateLinkServiceConnections[0].privateLinkServiceConnectionState.status,Subnet:subnet.id}" \
  -o table

Look for targets ending in:

Microsoft.KeyVault/vaults/<KEYVAULT>

and:

Microsoft.Sql/servers/<SQL-SERVER>

You should find something conceptually like:

Private Endpoint       Target                 Group       Status
---------------------  ---------------------  ----------  --------
<kv-pe>                .../vaults/<KV>        vault       Approved
<sql-pe>               .../servers/<SQL>      sqlServer   Approved

This tells us:

* exact PE names
* PE resource groups
* subnets
* target resource
* connection status

⸻

9. Inspect each Private Endpoint deeply

After identifying the Key Vault PE:

az network private-endpoint show \
  --resource-group <KV-PE-RG> \
  --name <KV-PE-NAME> \
  -o json

For SQL:

az network private-endpoint show \
  --resource-group <SQL-PE-RG> \
  --name <SQL-PE-NAME> \
  -o json

az network private-endpoint show is the supported command for retrieving PE configuration. 

⸻

10. This is a particularly important check: DNS Zone Group

For Key Vault:

az network private-endpoint dns-zone-group list \
  --resource-group <KV-PE-RG> \
  --endpoint-name <KV-PE-NAME> \
  -o json

For SQL:

az network private-endpoint dns-zone-group list \
  --resource-group <SQL-PE-RG> \
  --endpoint-name <SQL-PE-NAME> \
  -o json

We want to discover whether the existing working setup has:

Key Vault PE
   ↓
privatelink.vaultcore.azure.net

and:

SQL PE
   ↓
privatelink.database.windows.net

This is one of the checks I particularly want you to run.

Microsoft notes that if a private endpoint has no DNS zone group, Azure does not automatically maintain the corresponding Private DNS record; some other DNS mechanism must then exist. 

⸻

11. Discover where the Private DNS zones actually exist

Run:

az network private-dns zone list \
  --query "[?name=='privatelink.vaultcore.azure.net' || name=='privatelink.database.windows.net'].{Zone:name,ResourceGroup:resourceGroup}" \
  -o table

Expected:

Zone                                    ResourceGroup
--------------------------------------  --------------------
privatelink.database.windows.net        <DNS-RG>
privatelink.vaultcore.azure.net         <DNS-RG>

This is important because the DNS zones may be in a central networking resource group, not the application RG.

⸻

12. Check DNS A records

For Key Vault:

az network private-dns record-set a list \
  --resource-group <DNS-RG> \
  --zone-name privatelink.vaultcore.azure.net \
  -o table

For SQL:

az network private-dns record-set a list \
  --resource-group <DNS-RG> \
  --zone-name privatelink.database.windows.net \
  -o table

We want to see records similar to:

<KeyVaultName>     → 10.x.x.x
<SQLServerName>    → 10.x.x.x

Azure CLI provides commands for listing and inspecting Private DNS zones and their records. 

⸻

13. Check which VNet is linked to each Private DNS zone

For Key Vault:

az network private-dns link vnet list \
  --resource-group <DNS-RG> \
  --zone-name privatelink.vaultcore.azure.net \
  -o table

For SQL:

az network private-dns link vnet list \
  --resource-group <DNS-RG> \
  --zone-name privatelink.database.windows.net \
  -o table

This is critical.

We want to determine whether:

Web App Integration VNet
        │
        ├── linked → privatelink.database.windows.net
        │
        └── linked → privatelink.vaultcore.azure.net

Or whether the environment instead uses corporate/custom DNS forwarding.

⸻

14. Inspect the VNet’s DNS servers

From Step 2, identify:

VNET_RG
VNET_NAME

Then:

az network vnet show \
  --resource-group <VNET-RG> \
  --name <VNET-NAME> \
  --query "{Name:name,AddressSpace:addressSpace.addressPrefixes,DnsServers:dhcpOptions.dnsServers}" \
  -o json

This output is extremely important.

If you see:

"DnsServers": []

the VNet is using Azure-provided DNS.

If you see something like:

"DnsServers": [
    "10.x.x.x",
    "10.x.x.x"
]

then the environment is using custom/corporate DNS.

Microsoft specifically recommends checking dhcpOptions.dnsServers when diagnosing private DNS behavior. 

⸻

15. Inspect the exact App Service integration subnet

az network vnet subnet show \
  --resource-group <VNET-RG> \
  --vnet-name <VNET-NAME> \
  --name <APP-INTEGRATION-SUBNET> \
  --query "{Name:name,AddressPrefix:addressPrefix,Delegations:delegations[].serviceName,NSG:networkSecurityGroup.id,RouteTable:routeTable.id,PrivateEndpointPolicies:privateEndpointNetworkPolicies}" \
  -o json

For App Service integration, you would normally expect:

Delegation:
Microsoft.Web/serverFarms

Microsoft’s current App Service guidance continues to use the integration subnet and Microsoft.Web/serverFarms delegation for VNet integration. 

⸻

16. Check whether a route table is involved

If Step 15 returns a route table, inspect it:

az network route-table show \
  --resource-group <ROUTE-TABLE-RG> \
  --name <ROUTE-TABLE-NAME> \
  -o json

Then:

az network route-table route list \
  --resource-group <ROUTE-TABLE-RG> \
  --route-table-name <ROUTE-TABLE-NAME> \
  -o table

This tells us whether traffic is being sent through:

Firewall
NVA
VPN
ExpressRoute
Internet
Virtual network

and could explain why the current setup behaves differently from a simple Azure-native design.

⸻

17. Determine how the application gets SQL/Key Vault configuration

Do not dump connection-string values.

List only connection-string names and types:

az webapp config connection-string list \
  --resource-group $WEBAPP_RG \
  --name $WEBAPP_NAME \
  --query "[].{Name:name,Type:type}" \
  -o table

Then list only App Setting names, not values:

az webapp config appsettings list \
  -g $WEBAPP_RG \
  -n $WEBAPP_NAME \
  --query "[].name" \
  -o tsv | grep -Ei "sql|database|db|keyvault|vault|clientid|managed"

This can tell us whether the working app is using something like:

KeyVaultURL
SQLConnectionString
DatabaseConnection
ClientID

without exposing the actual secrets.

⸻

What I want to reconstruct

Once these outputs are collected, we can make an exact diagram like:

CURRENT WORKING ENVIRONMENT
Azure Web App
    │
    ├── System Managed Identity
    │       │
    │       └── Key Vault Secrets User
    │
    └── VNet Integration
             │
             ▼
       <VNET>/<SUBNET>
             │
        DNS servers
             │
       ┌─────┴──────┐
       │            │
       ▼            ▼
Private DNS      Private DNS
KV zone          SQL zone
       │            │
       ▼            ▼
KV Private PE    SQL Private PE
10.x.x.x         10.x.x.x
       │            │
       ▼            ▼
Key Vault       Azure SQL

The first commands I would run now

Don’t run everything at once. Start with these six:

az webapp vnet-integration list -g $WEBAPP_RG -n $WEBAPP_NAME -o json
az webapp identity show -g $WEBAPP_RG -n $WEBAPP_NAME -o json
az webapp config appsettings list -g $WEBAPP_RG -n $WEBAPP_NAME --query "[?name=='WEBSITE_DNS_SERVER' || name=='WEBSITE_DNS_ALT_SERVER' || name=='WEBSITE_VNET_ROUTE_ALL'].{Setting:name,Value:value}" -o table
az keyvault show -g $KV_RG -n $KV_NAME --query "{RBAC:properties.enableRbacAuthorization,PublicNetworkAccess:properties.publicNetworkAccess,DefaultAction:properties.networkAcls.defaultAction}" -o json
az network private-endpoint list --query "[].{PE:name,RG:resourceGroup,Target:privateLinkServiceConnections[0].privateLinkServiceId,Group:privateLinkServiceConnections[0].groupIds[0],Status:privateLinkServiceConnections[0].privateLinkServiceConnectionState.status,Subnet:subnet.id}" -o table
az network private-dns zone list --query "[?name=='privatelink.vaultcore.azure.net' || name=='privatelink.database.windows.net'].{Zone:name,RG:resourceGroup}" -o table

Send me those six outputs first—you can mask subscription IDs/principal IDs. From them, I can map your existing architecture and tell you exactly which next CLI commands to run, rather than guessing the VNet/PE/DNS resource groups. Runtime DNS (Resolve-DnsName) still needs to be tested from the Web App worker/Kudu after we understand the configuration.