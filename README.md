# Create Python Virtual Environment

## Creating your first Python Azure Function

To understand how to create your first Python Azure function then read the "[Create your first Python function in Azure ](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-first-function-python)" article.

## Solution overview

![solution overview](./docs/resources/python-azure-functions-solution.png)

## Architectural Considerations

### Optimistic Concurrency

I wanted to maintain a count in the Device State table of the number of times a device had sent telemetry. Depending on the workload, Azure Functions can auto scale the number of instances running and there is a possibility of data corruption. I could have used a transactional data store, but for this solution, that would be overkill and expensive. So I have implemented Azure Storage/CosmosDB Optimistic Concurrency to deal with the possibility that two processes may attempt to update the same storage entity at the same time.

[Managing Concurrency in Microsoft Azure Storage](https://azure.microsoft.com/en-au/blog/managing-concurrency-in-microsoft-azure-storage-2/)

```python
def updateDeviceState(telemetry):
    success = False
    mergeRetry = 0

    while not success and mergeRetry < 10:
        mergeRetry = mergeRetry + 1
        count = 0
        entity = {}

        try:
            # get existing telemetry entity
            entity = table_service.get_entity(
                environmentTable, partitionKey, telemetry['DeviceId'])
            if 'Count' in entity:
                count = entity['Count']
        except:
            count = 0

        count = count + 1
        updateEntity(telemetry, entity, count)
        
        if 'etag' in entity:    # if etag found then record existed
            try:
                # try a merge - it will fail if etag doesn't match
                etag = table_service.merge_entity(
                    environmentTable, entity, if_match=entity['etag'])
                success = True
            except:
                success = False
        else:
            try:
                table_service.insert_entity(environmentTable, entity)
                success = True
            except:
                success = False
```

### Telemetry Calibration Optimization

Rather than loading all the calibration data in when the Azure Function starts I lazy load calibration data into a Python dictionary.

```python
def getCalibrationData(deviceId):
    if deviceId not in calibrationDictionary:
        try:
            calibrationDictionary[deviceId] = table_service.get_entity(
                calibrationTable, partitionKey, deviceId)
        except:
            calibrationDictionary[deviceId] = None

    return calibrationDictionary[deviceId]
```



In Bash

```bash
python3.6 -m venv .env
source .env/bin/activate
```

In PowerShell
```powershell
py -3.6 -m venv .env
.env\scripts\activate
```

## Opening Python Project

Be sure that the virtual Python environment you created is active and select in Visual Studio Code

## Deploy Function

func azure functionapp publish enviromon-python --build-native-deps

## Solution components

1. [Image Classifier running on Azure IoT Edge].(https://github.com/gloveboxes/Creating-an-image-recognition-solution-with-Azure-IoT-Edge-and-Azure-Cognitive-Services). This project is configured for Raspberry Pi 2, 3 (A+ or B+) or Linux Desktop as Docker camera pass-through is required.

2. Azure IoT Hub Python Functions (included in this GitHub repo). This project is responsible for reacting to new telemetry from Azure IoT Hub, updating the Classified Storage Table and passing telemetry to the Azure SignalR service for near real-time web client update.

3. Azure SignalR .NET Core Function (Included in this GitHub repo). This Azure Function is responsible for distribution of new telemetry messaged out to the Web Client (https://enviro.z8.web.core.windows.net/image.html)

4. [Web Dashboard](https://enviro.z8.web.core.windows.net/classified.html) (Included in this GitHub repo). This is a Single Page Web App that is hosted on Azure Storage as a Static Website. So it too is serverless. The page used for this sample is classified.html. Be sure to modify the "apiBaseUrl" url to point your instance of the Azure SignalR Azure Function you install.