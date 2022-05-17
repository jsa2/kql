## Check Existence of ServicePrincipal AppID across Azure Resource Manager logs 

The idea of this check is, that we don't always know, in which attribute the AppId is stored in Azure Resource Manager logs. Thus this query will search in all attributes in the event entity for existence of AppID.

**For example:**
Value for AppId in Key Vault logs is ``identity_claim_appid_g``, whereas in ActivityLogs it is ``claims.appId `` or in SQL logs it is ``session_server_principal_name_s`` - This query does not care in which attribute the appId is stored, thus making it easier to search across mass.



## Running the query
 You need to have all logs (highlighted below) in relevant categories enabled in order to use this query:

1. Creates a list of AppId's from [AADServicePrincipalSignInLogs](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/aadserviceprincipalsigninlogs) and [AADManagedIdentitySignInLogs](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/aadmanagedidentitysigninlogs) logs
2. Searches with [mv-apply](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/mv-applyoperator) from [AzureDiagnostic](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/azurediagnostics) and [AzureActivity](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/azureactivity) "mass" using [pack_all()](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/packallfunction)

![image](https://user-images.githubusercontent.com/58001986/147085971-313d534b-fbd0-4a29-9b87-0a11d1fc8b3b.png)

## The Query
[ARM-Appid.kql](searchForAppIdinARM/ARM-Appid.kql)

## Result examples
- Tracking GH workload federated actions
![image](https://user-images.githubusercontent.com/58001986/147140668-e78c0d57-8c91-4508-bca2-6dd32fb16bca.png)
