# Building a Serverless IoT Solution with Kotlin Azure Functions and SignalR

Follow me on [Twitter](https://twitter.com/dglover), [Project Source Code](https://github.com/gloveboxes/Go-Serverless-with-Python-Azure-Functions-and-SignalR), [Powerpoint Slides](https://github.com/gloveboxes/Go-Serverless-with-Python-Azure-Functions-and-SignalR/blob/master/docs/Python%20Serverless%20with%20Azure%20Functions.pptx), [PDF Slides](https://github.com/gloveboxes/Go-Serverless-with-Python-Azure-Functions-and-SignalR/blob/master/docs/Python%20Serverless%20with%20Azure%20Functions.pdf)

## Solution Overview

![solution overview](https://raw.githubusercontent.com/gloveboxes/Go-Serverless-with-Python-Azure-Functions-and-SignalR/master/docs/resources/solution-architecture.png)

This solution diagram overviews a typical IoT solution. [Azure IoT Hub](https://docs.microsoft.com/en-us/azure/iot-hub?WT.mc_id=github-blog-dglover) is responsible for internet scale, secure, bi-directional communication with devices and backend services.

Telemetry can be [routed](https://docs.microsoft.com/en-us/azure/iot-hub/tutorial-routing?WT.mc_id=github-blog-dglover) by Azure IoT Hub to various services and also to storage in [Apache Avro](https://avro.apache.org/docs/current/) or JSON format for purposes such as audit, integration or driving machine learning processes.

This posting takes a slice of this scenario and is about the straight through [serverless](https://en.wikipedia.org/wiki/Serverless_computing) processing of telemetry from Azure IoT Hub, via Python Azure Functions and Azure SignalR for a near real-time dashboard.

### Azure Services

The following Azure services are used in this solution and available in Free  tiers: [Azure IoT Hub](https://docs.microsoft.com/en-us/azure/iot-hub?WT.mc_id=github-blog-dglover), [Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions?WT.mc_id=github-blog-dglover), [Azure SignalR](https://docs.microsoft.com/en-us/azure/azure-signalr?WT.mc_id=github-blog-dglover), [Azure Storage](https://docs.microsoft.com/en-us/azure/storage?WT.mc_id=github-blog-dglover), [Azure Storage Static Websites](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blob-static-website?WT.mc_id=github-blog-dglover)

You can sign up for a [Free Azure Account](https://azure.microsoft.com/en-au/free?WT.mc_id=github-blog-dglover), if you are a student then be sure to sign up for [Azure for Students](https://azure.microsoft.com/en-au/free/students?WT.mc_id=github-blog-dglover), no credit card required.

## Developing Python Azure Functions

## Where to Start

Review the [Azure Functions Python Worker Guide](https://github.com/Azure/azure-functions-python-worker). There is information on the following topics:

- [Create your first Python function](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-first-function-python?WT.mc_id=github-blog-dglover)
- [Developer guide](https://docs.microsoft.com/en-us/azure/azure-functions/functions-reference-python?WT.mc_id=github-blog-dglover)
- [Binding API reference](https://docs.microsoft.com/en-us/python/api/azure-functions/azure.functions?view=azure-python&WT.mc_id=github-blog-dglover)
- [Develop using VS Code](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-first-function-vs-code?WT.mc_id=github-blog-dglover)
- [Create a Python Function on Linux using a custom docker image](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-function-linux-custom-image?WT.mc_id=github-blog-dglover)

## Solution Components (included in this GitHub repo)

1. Kotlin Azure Function. This Azure Function processes batches of telemetry, then calibrations and validates telemetry, and updates the Device State Azure Storage Table, and then passes the telemetry to the Azure SignalR service for near real-time web client update.

3. [Web Dashboard](https://enviro.z8.web.core.windows.net/enviromon.html). This Single Page Web App is hosted on Azure Storage as a Static Website. So it too is serverless.

## Design Considerations

### Optimistic Concurrency

First up, it is useful to understand [Event Hub Trigger Scaling](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-iot#trigger---scaling?WT.mc_id=github-blog-dglover) and how additional function instances can be started to process events.

I wanted to maintain a count in the Device State table of the number of times a device had sent telemetry. The solution implements [Azure Storage/CosmosDB Optimistic Concurrency](https://azure.microsoft.com/en-us/blog/managing-concurrency-in-microsoft-azure-storage-2?WT.mc_id=github-blog-dglover).

[Optimistic Concurrency (OCC)](https://en.wikipedia.org/wiki/Optimistic_concurrency_control) assumes that multiple transactions can frequently complete without interfering with each other. While running, transactions use data resources without acquiring locks on those resources. Before committing, each transaction verifies that no other transaction has modified the data it has read. OCC is generally used in environments with low data contention.

If there are multiple functions instances updating and there is a clash, I have implemented Exponential Backoff and added a random factor to allow for retry.

```kotlin
private  fun calcExponentialBackoff(attempt: Int) : Long{
    val base = occBase * Math.pow(2.0, attempt.toDouble())
    return ThreadLocalRandom.current().nextLong(occBase,min(occCap, base.toLong()))
}
```

From my limited testing Exponential Backoff was effective.

## Telemetry Processing

As each message is processed, retrieves the existing entity, updates the 'count', and updates the entity with the new telemetry. If there is an OCC clash then the process backs off for the calculated time and retries the update.

```kotlin
message.forEach { environment ->

    maxRetry = 0

    while (maxRetry < 10) {
        maxRetry++

        try {
            top = TableOperation.retrieve(_partitionKey, environment.deviceId, EnvironmentEntity::class.java)
            val existingEntity = deviceStateTable.execute(top).getResultAsType<EnvironmentEntity>()

            calibrate(environment)

            if (!validateTelemetry(environment)) {
                context.logger.info("Data failed validation.")
                break
            }

            with(environment) {
                partitionKey = _partitionKey
                rowKey = environment.deviceId
                timestamp = Date()
            }

            if (existingEntity?.etag != null) {
                environment.etag = existingEntity.etag
                environment.count = existingEntity.count
                environment.count++

                top = TableOperation.replace(environment)
                deviceStateTable.execute(top)
            } else {
                environment.count = 1
                top = TableOperation.insert(environment)
                deviceStateTable.execute(top)
            }

            deviceState[environment.deviceId!!] = environment

            break

        } catch (e: java.lang.Exception) {
            val interval = calcExponentialBackoff(maxRetry)
            Thread.sleep(interval)
            context.logger.info("Optimistic Consistency Backoff interval $interval")
        }
    }

    if (maxRetry >= 10){
        context.logger.info("Failed to commit")
    }
}
```

### Telemetry Calibration Optimization

You can either calibrate data on the device or in the cloud. I prefer to calibrate cloud-side. The calibration data could be loaded with Azure Function Data Binding but I prefer to lazy load the calibration data. There could be a lot of calibration data so it does not make sense to load it all at once when the function is triggered.

```kotlin
private fun calibrate(environment: EnvironmentEntity) {
    val calibrationData:CalibrationEntity?

    if (calibrationMap.containsKey(environment.deviceId)){
        calibrationData = calibrationMap[environment.deviceId]
    }
    else {
        top = TableOperation.retrieve(_partitionKey, environment.deviceId, CalibrationEntity::class.java)
        calibrationData = calibrationTable.execute(top).getResultAsType<CalibrationEntity>()
        calibrationMap[environment.deviceId] = calibrationData
    }

    with(environment) {
        calibrationData?.let {
            temperature = scale(temperature, it.TemperatureSlope, it.TemperatureYIntercept)
            humidity = scale(humidity, it.HumiditySlope, it.HumidityYIntercept)
            pressure = scale(pressure, it.PressureSlope, it.PressureYIntercept)
        }
    }
}
```

### Telemetry Validation

IoT solutions should validate telemetry to ensure data is within sensible ranges to allow for faulty sensors.

```python
private fun validateTelemetry(telemetry: EnvironmentEntity): Boolean {
    telemetry.temperature?.let {
        if (it < -10 || it > 70) {
            return false
        }
    }

    telemetry.humidity?.let {
        if (it < 0 || it > 100) {
            return false
        }
    }

    telemetry.pressure?.let {
        if (it < 0 || it > 1500) {
            return false
        }
    }

    return true
}
```

## Azure SignalR Integration

There is no Service-Side Azure SignalR SDK. To send telemetry from the Event Hub Trigger Azure Function to the Dashboard Web Client you need to call a HTTP Azure Function that is bound to the SignalR service. This SignalR Azure Function then sends the telemetry via SignalR as if the data was coming from a client-side app.

The flow for Azure SignalR integration is as follows:

1. The Web client makes a REST call to '**negotiate**', amongst other things, the SignalR '**Hubname**' is returned to the client.
2. The Web client then makes a REST call to '**getdevicestate**', this HTTP Trigger retrieves the state for all devices from the Device State Table. The data is returned to the client via SignalR via the same '**Hubname**' that was returned from the call to '**negotiate**'.
3. When new telemetry arrives via IoT Hub, the '**TelemetryProcessing**' trigger fires, the telemetry is updated in the Device State table and a REST call is made to the '**SendSignalRMessage**' and telemetry is sent to all the SignalR clients listening on the '**Hubname**' channel.

![](https://raw.githubusercontent.com/gloveboxes/Go-Serverless-with-Python-Azure-Functions-and-SignalR/master/docs/resources/service-side-signalr.png)

## Set Up Overview

This lab uses free of charge services on Azure. The following need to be set up:

1. Azure IoT Hub and Azure IoT Device
2. Azure SignalR Service
3. Deploy the Python Azure Function
4. Deploy the SignalR .NET Core Azure Function

### Step 1: Follow the Raspberry Pi Simulator Guide to set up Azure IoT Hub

**While in Python Azure Functions are in preview they are available in limited locations. For now, 'westus', and 'westeurope'. I recommend you create all the project resources in one of these locations.**

[Setting up the Raspberry Pi Simulator](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-raspberry-pi-web-simulator-get-started?WT.mc_id=github-blog-dglover)

![raspberry Pi Simulator](https://docs.microsoft.com/en-us/azure/iot-hub/media/iot-hub-raspberry-pi-web-simulator/3_banner.png?WT.mc_id=github-blog-dglover)

### Step 2: Create an Azure Resource Group

[az group create](https://docs.microsoft.com/en-us/cli/azure/group?view=azure-cli-latest#az-group-create&WT.mc_id=github-blog-dglover)

```bash
az group create -l westus -n enviromon-python
```

### Step 3: Create a Azure Signal Service

- [az signalr create](https://docs.microsoft.com/en-us/cli/azure/signalr?view=azure-cli-latest#az-signalr-create&WT.mc_id=github-blog-dglover) creates the Azure SignalR Service
- [az signalr key list](https://docs.microsoft.com/en-us/cli/azure/ext/signalr/signalr/key?view=azure-cli-latest#ext-signalr-az-signalr-key-list&WT.mc_id=github-blog-dglover) returns the connection string you need for the SignalR .NET Core Azure Function.

```bash
az signalr create -n <Your SignalR Name> -g enviromon-python --sku Free_DS2 --unit-count 1
az signalr key list -n <Your SignalR Name> -g enviromon-python
```

### Step 4: Create a Storage Account

[az storage account create](https://docs.microsoft.com/en-us/cli/azure/storage/account?view=azure-cli-latest#az-storage-account-create&WT.mc_id=github-blog-dglover)

```bash
az storage account create -n enviromonstorage -g enviromon-python -l westus --sku Standard_LRS --kind StorageV2
```

### Step 5: Clone the project

```bash
git clone https://github.com/gloveboxes/Go-Serverless-with-Python-Azure-Functions-and-SignalR.git
```

### Step 6: Deploy the SignalR .NET Core Azure Function

```bash
cd  Go-Serverless-with-Python-Azure-Functions-and-SignalR

cd dotnet-signalr-functions

cp local.settings.sample.json local.settings.json

code .
```

### Step 7: Update the local.settings.json

```json
{
    "IsEncrypted": false,
    "Values": {
        "AzureWebJobsStorage": "<The Storage Connection String for enviromonstorage>",
        "FUNCTIONS_WORKER_RUNTIME": "dotnet",
        "StorageConnectionString":"<The Storage Connection String for enviromonstorage>",
        "AzureSignalRConnectionString": "<The SignalR Coonection String from Step 3>"
    },
    "Host": {
        "LocalHttpPort": 7071,
        "CORS": "http://127.0.0.1:5500,http://localhost:5500,https://azure-samples.github.io",
        "CORSCredentials": true
    }
}
```

### Step 9: Open the Python Functions Project with Visual Studio Code

Change to the directory where you cloned to the project to, the change to the iothub-python-functions directory, then start Visual Studio Code.

From Terminal on Linux and macOS, or Powershell on Windows.

```bash

cd  Go-Serverless-with-Python-Azure-Functions-and-SignalR

cd iothub-python-functions

cp local.settings.sample.json local.settings.json

code .
```

### Step 10: Update the local.settings.json

```json
{
  "IsEncrypted": false,
  "Values": {
    "FUNCTIONS_WORKER_RUNTIME": "python",
    "AzureWebJobsStorage": "<The Storage Connection String for enviromonstorage>",
    "IoTHubConnectionString": "<The IoT Hub Connection String>",
    "PartitionKey": "<Storage Partition Key - arbitrary - for example the name of city/town>",
    "StorageConnectionString": "<The Storage Connection String for enviromonstorage>",
    "SignalrUrl": "<SendSignalrMessage Invoke URL>"
  }
}
```

### Step 11: Deploy the Python Azure Function

1. Open a terminal window in Visual Studio. From the main menu, select View -> Terminal
2. Deploy the Azure Function

```bash
func azure functionapp publish enviromon-python --publish-local-settings --build-native-deps  
```

### Step 12: Enable Static Websites for Azure Storage

The Dashboard project contains the Static Website project.

Follow the guide for [Static website hosting in Azure Storage](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blob-static-website?WT.mc_id=github-blog-dglover).

The page used for this sample is enviromon.html. Be sure to modify the "apiBaseUrl" url in the web page javascript to point your instance of the SignalR Azure Function.

Copy the contents of the dashboard project to the static website.

### Step 13: Enable CORS for the SignalR .NET Core Azure Function

[az functionapp cors add](https://docs.microsoft.com/en-us/cli/azure/functionapp/cors?view=azure-cli-latest#az-functionapp-cors-add&WT.mc_id=github-blog-dglover)

```bash
az functionapp cors add -g enviromon-python -n <Your SignalR Function Name> --allowed-origins <https://my-static-website-url>
```

### Step 14: Start the Dashboard

From your web browser, navigate to https://your-start-web-site/enviromon.html

The telemetry from the Raspberry Pi Simulator will be displayed on the dashboard.

![dashboard](https://raw.githubusercontent.com/gloveboxes/Go-Serverless-with-Python-Azure-Functions-and-SignalR/master/docs/resources/dashboard.png)