# Map Azure IP list to SPN sign-ins 


✅ Maps Azure IP range & service to SPN signins counted per day


## 
- Update download link from [Azure IP Ranges and Service Tags – Public Cloud](https://www.microsoft.com/en-us/download/details.aspx?id=56519) to url in ``externaldata()``
- Inspired by similar sentinel query for tenant sign-ins ``Azure Portal Signin from another Azure Tenant`` 
- The use of evaluate bag_unpack() was heavily inspired by work of [Matthew Zorich](https://twitter.com/reprise_99)

**Query**

```sql

let lookBack = 180d;
let spns = AADServicePrincipalSignInLogs 
| where TimeGenerated > ago(lookBack)
| project authTime=TimeGenerated, AppId, ServicePrincipalName, aadIp = IPAddress
| summarize count() by bin(authTime, 1d), AppId, aadIp, ServicePrincipalName
| summarize  make_list(pack_all(true));
let azIp =externaldata(changeNumber: string, cloud: string, values: dynamic)
["https://download.microsoft.com/download/7/1/D/71D86715-5596-4529-9B13-DA13A5DE5B63/ServiceTags_Public_20220613.json"]
with(format='multijson')
| mv-expand values
| project  aId =values.id, prefix =values.properties.addressPrefixes
| mv-expand prefix
| project aId, prefix;
azIp
| mv-apply spn= toscalar(spns) to typeof(dynamic) on 
    (
extend matc = ipv4_is_in_range(tostring(spn.aadIp),tostring(prefix)) 
    )
| where matc == true
| evaluate bag_unpack(spn)

```

**Result preview**

![](20220622135214.png)  

## Automatic updates with Azure Function

```sql
let lookBack = 90d;
let spns = AADServicePrincipalSignInLogs 
| where TimeGenerated > ago(lookBack)
| project authTime=TimeGenerated, AppId, ServicePrincipalName, aadIp = IPAddress
| summarize count() by bin(authTime, 1d), AppId, aadIp, ServicePrincipalName
| summarize  make_list(pack_all(true));
let azIp =externaldata(changeNumber: string, cloud: string, values: dynamic)
["https://fn-azip-28667.azurewebsites.net/api/content/blob.json"
] with(format='multijson')
| mv-expand values
| project  aId =values.id, prefix =values.properties.addressPrefixes
| mv-expand prefix
| project aId, prefix;
azIp
| mv-apply spn= toscalar(spns) to typeof(dynamic) on 
    (
extend matc = ipv4_is_in_range(tostring(spn.aadIp),tostring(prefix)) 
    )
| where matc == true
| evaluate bag_unpack(spn)
```


https://fn-azip-28667.azurewebsites.net/api/content/blob.json

✅ - The function is based on following API https://docs.microsoft.com/en-us/rest/api/virtualnetwork/service-tags/list

✅ - This is just a poc. So For any serious use, I would use similar function on timer, and make output binding of Azure Storage. The timer could follow then similar update cadence, as the MS hosted file (this is HTTP trigger, so it is updated on each request, if there are updates)

⚠️ - Do not use this function in any production environment 

✅ - I made this demo only to see how close it would align with the MS LIST (it identified all the same IP's)

⚠️ - I did not add pagination check, as this is only POC

Function uses managed identity with very restricted role of ``Microsoft.Network/locations/serviceTags/read`` 

![image](https://user-images.githubusercontent.com/58001986/175968436-576153ee-9399-4bef-af1b-697bc4ad7cd3.png)


![image](https://user-images.githubusercontent.com/58001986/175968558-a3c585a4-1a01-427a-b1aa-8905b123da31.png)




### CORS

![image](https://user-images.githubusercontent.com/58001986/175968123-d63868c7-981e-4e6d-b3aa-cfee261bba92.png)

### function.json

```json
{
  "bindings": [
    {
      "authLevel": "anonymous",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "http": {
        "EnableChunkedRequestBinding": false
      },
      "methods": [
        "get","head","options"
      ],
      "route": "content/blob.json"
    },
    {
      "type": "http",
      "direction": "out",
      "name": "res"
    }
  ]
}

```



### index.js
- The code for getting tokens with this function is available [here](https://github.com/jsa2/Azure-Functions-LinuxStarterKit)

```js



const { axiosClient } = require('../src/axioshelpers')
const getToken = require('../src/token')

module.exports = async function (context, req) {

            var res = 'https://management.azure.com'

            var token = await getToken(res).catch((error) => {
                console.log(error)
                return context.done()
            })
            try {
                var data = await axiosClient({
                    url: `https://management.azure.com/subscriptions/{yourSUBSCRIPTION}/providers/Microsoft.Network/locations/westeurope/serviceTags?api-version=2021-08-01`,
                    method: "get",
                    headers: {
                        authorization: "Bearer " + token.access_token
                    }
                })

                return context.res = {
                    status: 200,
                    /* Defaults to 200 */
                    headers: {
                        "content-type": "application/json"
                    },
                    body: data?.data
                };

            } catch (error) {
                return context.res = {
                    status: 404,
                    /* Defaults to 200 */
                    body: {
                        err: error?.response?.data
                    }
                };

            }
}
```

**End**
