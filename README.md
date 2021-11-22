# Cloud One Application Security Azure Functions (Python)

- [Cloud One Application Security Azure Functions (Python)](#cloud-one-application-security-azure-functions-python)
  - [Create](#create)
  - [Publish](#publish)
  - [Cleanup](#cleanup)

> Requires python <= 3.8

## Create

```sh
$ # Build Function
$ func init myfunction --worker-runtime python
```

```sh
$ cd myfunction/
$ APP_NAME="HttpTriggerPY"
$ func new --language python --template "HttpTrigger" --name ${APP_NAME}
```

Modify the `requirements.txt` to add `trend_app_protect`

```txt
# Do not include azure-functions-worker as it may conflict with the Azure Functions platform
azure-functions
trend_app_protect
```

```sh
$ pip3 install -r requirements.txt --user
$ func extensions install
```

Add TREND_AP_KEY and TREND_AP_SECRET to `local_settings.json`

```json
{
  "IsEncrypted": false,
  "Values": {
    "FUNCTIONS_WORKER_RUNTIME": "python",
    "AzureWebJobsStorage": "",
    "TREND_AP_KEY": "f093xxxx-xxxx-xxxx-xxxx-6436xxxxxxxx",
    "TREND_AP_SECRET": "b14exxxx-xxxx-xxxx-xxxx-1b5bxxxxxxxx"
  }
}
```

An `__init__.py` vulnerable for *Illegal File Access* and *Remote Command Execution*:

```python
import logging

import azure.functions as func
import os

from trend_app_protect.api.azure_function import protect_function

@protect_function
def main(req: func.HttpRequest, context: func.Context) -> func.HttpResponse:
    logging.info('Python HTTP trigger function processed a request.')

    filename = req.params.get('filename')
    execute = req.params.get('execute')
    if not filename and not execute:
        try:
            req_body = req.get_json()
        except ValueError:
            pass
        else:
            filename = req_body.get('filename')
            execute = req_body.get('execute')

    if filename:
        with open(filename) as f: s = f.read()
        return func.HttpResponse(f"{s}")
    elif execute:
        s = os.popen(execute)
        o = s.read()
        return func.HttpResponse(f"{o}")
    else:
        return func.HttpResponse(
             "This HTTP triggered function executed successfully. Pass a name in the query string or in the request body for a personalized response.",
             status_code=200
        )
```

```sh
$ # Test function locally
$ func host start
```

Example curls:

```sh
$ curl http://localhost:7071/api/HttpTriggerPY?filename=/etc/passwd

$ curl http://localhost:7071/api/HttpTriggerPY?execute=env
```

## Publish

```sh
$ # Publish to Azure
$ LOCATION=westeurope
$ RESOURCE_GROUP=${APP_NAME}rg
$ STORAGE_ACCOUNT=httptriggerpy$(openssl rand -hex 4)

$ az group create --name ${RESOURCE_GROUP} --location ${LOCATION}
```

```sh
$ az storage account create --name ${STORAGE_ACCOUNT} --sku Standard_LRS --resource-group ${RESOURCE_GROUP}
```

```sh
$ az functionapp create --consumption-plan-location ${LOCATION} --runtime python --runtime-version 3.8 --functions-version 3 --name ${APP_NAME} --os-type linux --resource-group ${RESOURCE_GROUP} --storage-account ${STORAGE_ACCOUNT}
```

```sh
$ func azure functionapp publish ${APP_NAME}
```

If you get `Can't find app with name "HttpTriggerPY"` with aboves command, retry it after a couple of seconds.

Set TREND_AP_KEY and TREND_AP_SECRET

```sh
$ az functionapp config appsettings set --name ${APP_NAME} -g ${RESOURCE_GROUP} --settings TREND_AP_KEY=$(cat local.settings.json | jq -r '.Values.TREND_AP_KEY')
$ az functionapp config appsettings set --name ${APP_NAME} -g ${RESOURCE_GROUP} --settings TREND_AP_SECRET=$(cat local.settings.json | jq -r '.Values.TREND_AP_SECRET')
```

Try in your browser or use curl:

<https://httptriggerpy.azurewebsites.net/api/httptriggerpy?code=sxrLLr...jHVqQ==&filename=/etc/passwd>

<https://httptriggerpy.azurewebsites.net/api/httptriggerpy?code=sxrLLr...jHVqQ==&execute=hostname>

```sh
$ curl -XPOST https://httptriggerpy.azurewebsites.net/api/httptriggerpy?code=sxrLLr...jHVqQ== -d '{"execute": "hostname"}'
```

## Cleanup

```sh
$ az group delete --name ${RESOURCE_GROUP}
```
