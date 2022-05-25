## Check Existence of ServicePrincipal AppID across Azure Resource Manager logs 

The idea of this check is, that we don't always know, in which field the AppId is stored in Azure Resource Manager logs. Thus this query will search in all fields in the event entity for existence of AppID.

**For example:**
Value for AppId in Key Vault logs is ``identity_claim_appid_g``, whereas in ActivityLogs it is 


1. Creates a list of AppId's
2. Searches with mv-apply from "mass" of ``pack_all()``


## SPN and Azure Resource Manager Correlation for possible misuse of access token

This check will show a use case where Access Token is used from **different** IP address, than it was claimed by the Client Credentials flow. While there are legitimate use cases where the IP address changes from the token, it is also flag for stolen, or misuse of the access token.
- One likely false positive is scenario where there are two back-ends using the same client credentials, or some sort of load-balancing. These could likely be evened out in the query as further IP matching

### Logic

- Access token by same appId is used from a different IP within 4200 seconds (70 minutes, allows some skew)
- Access token ip does not match the callerIP in Azure Activity logs

```sql
let lookBack = 10d;
let spns = AADServicePrincipalSignInLogs 
| where TimeGenerated > ago(lookBack)
| project authTime=TimeGenerated, AppId, ServicePrincipalName, aadIp = IPAddress
| summarize  make_list(pack_all(true));
let activityCor = AzureActivity 
| where TimeGenerated > ago(lookBack)
| where Caller !has "@" and isnotempty( Claims_d)
| extend parsedClaims = parse_json(Claims_d)
| extend activityTime = TimeGenerated
| extend AppId = tostring(parsedClaims.appid);
activityCor
| mv-apply spn= toscalar(spns) to typeof(dynamic) on 
    (
    extend appMatch = AppId == spn.AppId
    | extend spnTime = spn.authTime
    | extend isInRange =(datetime_diff('second',todatetime(activityTime),todatetime(spnTime)))
    | extend ServicePrincipalName = spn.ServicePrincipalName
    | extend spnIp = spn.aadIp
    | where appMatch == true 
    | where isInRange between (0 .. 4200)
    | where spn.aadIp != CallerIpAddress
    )
| project OperationNameValue, appMatch, spnTime, TimeGenerated, isInRange, AppId,tostring(ServicePrincipalName), CallerIpAddress, tostring(spnIp)
```

**Simple version**
```
/// Append to the main query
| extend s = todatetime(spnTime)
| extend details = strcat(split(OperationNameValue, "MICROSOFT.")[1], '-IdpIP : ',spnIp, '- ArmIP : ',CallerIpAddress )
| summarize count() by bin(s, 70m), ServicePrincipalName, ['details']
| summarize make_set(['details']) by ServicePrincipalName

```
