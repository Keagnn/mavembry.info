---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Integrating Microsoft Intune with ServiceNow"
subtitle: ""
summary: ""
authors: []
tags: []
categories: []
date: 2020-05-15T16:42:45-07:00
lastmod: 2020-05-15T16:42:45-07:00
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: true

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---
<br>

 <b>Before you read:</b> If you are considering setting this integration up, I would highly suggest looking into [IntegrationHub ETL](https://docs.servicenow.com/bundle/rome-servicenow-platform/page/product/configuration-management/concept/integrationhub-etl.html). This is a free app from the ServiceNow store and it allows you to ingest 3rd party data into your CMDB which runs through the IRE engine.
<br>
<br>
Unless you are paying for Discovery or IntegrationHub, integrating with ServiceNow can definitely be a confusing task, but who wants to spend money just to create a basic integration? In this topic, I'll discuss how to setup an integration using the Microsoft Graph API.
<br>
<br>

In this guide, I am going to be pulling devices from Intune and importing them into the CMDB. There is very little documentation out there to help you with this integration, so this will provide you step-by-step instructions on setting this up. We will be using Azure to obtain the device data from Intune.
<br>

<h6>Azure setup</h6>

Contact your Azure admin to setup an application inside Azure to gain access to the API. The admin will need to follow these [Instructions](https://docs.microsoft.com/en-us/mem/intune/developer/intune-graph-apis). Once the app is registered in Azure, write down the following information:
<ul>
<li>Application Secret</li>
<li>Tenant ID</li>
<li>Client ID</li>
</ul>

Make sure the app has the `DeviceManagementManagedDevices.Read.All` application & delegated permission.  
<br>

<h6>Authentication</h6>

In the Microsoft [documentation](https://docs.microsoft.com/en-us/azure/active-directory/azuread-dev/v1-protocols-oauth-code), they tell us to use OAuth2.0 to authenticate. First, let's make sure your instance has OAuth2.0 enabled. Go in to your system properties, and make sure `com.snc.platform.security.oauth.is.active` is set to true.

In your ServiceNow instance, lets create an application registry. Navigate to `System OAuth > Application Registry`. Here are the fields you need to fill out:

| Field              | Value      
| ------------------ |:-------------------------------------:|
| Name               | Azure Authentication (can be anything)|
| Client ID          | Azure App client ID                   |
| Client Secret      | Azure App secret                      |
| Default Grant Type | Client Credentials                    |
| Token URL          | `https://login.microsoftonline.com/{tenantID}/oauth2/v2.0/token` |
| Redirect URL       | `https://instance-name.service-now.com/oauth_redirect.do`|

Now you will have to create an OAuth Entity Profile and choose the provider you just created. Once this is done, you will need to create the OAuth Entity Scope. The OAuth scope is `https://graph.microsoft.com/.default`.
<br>
<br>

<h6>Get the Payload</h6>

This is the fun part. We will now test the connection by generating a token with your application registry. Navigate to `System Web Services > REST Messages` and create a new one. Name the message anything you want, and add a description. The endpoint URL is what you can change depending on what information you need. Follow this [link](https://docs.microsoft.com/en-us/graph/api/resources/intune-devices-manageddevice?view=graph-rest-1.0) to find more managedDevice methods in the graph API.

For this guide, I want to list all devices in my environment. So the URL I will use is: `https://graph.microsoft.com/v1.0/deviceManagement/managedDevices`.

Select OAuth2.0 as the authentication type, and choose your profile you created above. Once complete, click the "Get OAuth Token" related link and make sure the OAuth2.0 flow completes. If it fails, you will need to double check your configuration in the registry you made.

After you have recieved the OAuth token, you can now test your REST message. Scroll down to HTTP Methods and open your Default GET record (you shouldn't have to change anything on this record, but make sure it is a GET request). Scroll down and click the `Test` related link.
<br>
<br>

<h6>Prepare for import</h6>

Once we have the JSON payload, we will need to create an import set table for these objects to import into. Copy/paste the payload in to any online [JSON beautifier](https://codebeautify.org/jsonviewer) tool, then convert one of the objects in the payload in to a CSV file.


E.g. I've copied one device along with it's metadeta. Your payload should look very similar if you are using the same endpoint.

```
 {
      "id": "",
      "userId": "",
      "deviceName": "",
      "managedDeviceOwnerType": "",
      "enrolledDateTime": "",
      "lastSyncDateTime": "",
      "operatingSystem": "",
      "complianceState": "",
      "jailBroken": "",
      "managementAgent": "",
      "osVersion": "",
      "easActivated": ,
      "easDeviceId": "",
      "easActivationDateTime": "" 
      ...etc
 }
     
```

Now I will [convert this to a CSV file](https://json-csv.com/). The whole point to this step is so the import set table has the correct fields for us to map to.

Once you have the CSV file, go to `Load Data` in the navigator and create a new table. Name it whatever you'd like, but remember it. Upload the CSV file you made and make sure the import set table has all the fields you want to map to from the payload.
<br>
<br>

<h6>Import the devices</h6>
In a real situation, you would likely want to import devices on a daily basis. In this case, we will create a scheduled job to run this REST message.
<br>
<br>

**Before I go any further, I'd like to give credit to [Jace Benson](http://jace.pro/) for assisting me on this. He is truly a master of ServiceNow.**

This is the part where you can decide on how you want to classify your CI's in your CMDB. In my payload, I only have two different classes; Windows and Android devices. I have created two different scheduled jobs, one for Windows devices and one for Android devices. There are alternative ways of classifying your devices into ServiceNow, this is just the way I went about it.


Go to `System Definition > Scheduled Jobs` and create a new scheduled job. I have it set to run daily and I've named it "Grab Windows devices from Intune".

Here is the script for the scheduled job:
```js
try {
    var machines = [];

    function getMachines(endpoint, machines) {
        gs.info('in getMachines with endpoint: ' + endpoint);
        var pagedR = new sn_ws.RESTMessageV2('NAME OF REST MESSAGE', 'Default GET'); // Replace the first parameter with the name of your REST message.
        if (endpoint !== null) {
            pagedR.setEndpoint(endpoint);
        }
        var pagedResponse = pagedR.execute();
        var pagedResponseBody = pagedResponse.getBody();
        var pagedhttpStatus = pagedResponse.getStatusCode();
        gs.info('windowsMachine response Status: ' + pagedResponseBody);
        var pagedObj = JSON.parse(pagedResponseBody);
        var newMachines = pagedObj.value.filter(function(device) {
            //This is the snippet that filters out only Windows devices. Hence the Windows only shceduled job.
            if (device.operatingSystem == "Windows") {
                return true;
            } else {
                return false;
            }
        });
        gs.info(endpoint + ' : ' + newMachines.length);
        machines = machines.concat(newMachines);
        if (pagedObj["@odata.nextLink"]) { // if it has paged results
            getMachines(pagedObj["@odata.nextLink"], machines);
        } else {
            gs.info('machines.length: ' + machines.length);
            machines.forEach(function(machine) {
                gs.info('in foreach');
                var intuneImport = new GlideRecord('IMPORT SET TABLE NAME'); // Replace with your Import set table name
                intuneImport.initialize();

                //Set each field in the table to the correct data from payload
                for (var key in machine) {
                    if (machine.hasOwnProperty(key)) {
                        var field = key.toLowerCase();
                        var value = (function() {
                            if (typeof machine[key] === "number") {
                                return machine[key].toString();
                            } else {
                                return machine[key];
                            }
                        })()

                        var actualField = 'u_' + field;
                        if (intuneImport.isValidField(actualField)) {
                            gs.info('setting (short)[' + actualField + ']:' + value);
                            intuneImport.setValue(actualField, value);
                        } else {
                            var begin = field.substring(0, 12);
                            var end = field.substring(field.length - 14, field.length);
                            var calculatedField = 'u_' + begin + '_' + end;
                            gs.info('setting (long)[' + calculatedField + ']:' + value);
                            intuneImport.setValue(calculatedField, value);
                        }
                    }
                }

                intuneImport.insert();
            });
        }
    }

    getMachines(null, machines);
} catch (ex) {
    var message = ex.message;
    gs.info('ERROR: ' + message);
}
```
<br>

***Make sure you replace line 6 with your REST message, and line 31 with your import set table name.***
<br>
<br>
This script is essentially calling the REST message you created, parsing the payload, creating a record for each device from your payload, and finally importing them into your import set table.
<br>

<h6>Transforming your CI's into the CMDB</h6>

Finally, you will need to create a transform map for your import set table. Navigate to `System Import Sets > Create Transform Map`. Name it anything you'd like and choose your import set table as the source table. The target table should be the CMDB table you want to import the devices into. Map the appropriate fields from the import set table to the CMDB table and make sure you set a coalesce on a field that is unique to each device.

You are finally DONE! Once the scheduled job runs, the transform map you created will automagically run and import the records into the CMDB table of your choice.

























