## Check Existence of ServicePrincipal AppID across Azure Resource Manager logs 

The idea of this check is, that we don't always know, in which field the AppId is stored in Azure Resource Manager logs. Thus this query will search in all fields in the event entity for existence of AppID.

**For example:**
Value for AppId in Key Vault logs is ``identity_claim_appid_g``, whereas in ActivityLogs it is 


1. Creates a list of AppId's
2. Searches with mv-apply from "mass" of ``pack_all()``