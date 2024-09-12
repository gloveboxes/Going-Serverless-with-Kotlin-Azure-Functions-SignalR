# Building a Serverless IoT Solution with Kotlin Azure Functions and SignalR

Follow me on [Twitter](https://twitter.com/dglover), [Project Source Code](https://github.com/gloveboxes/Going-Serverless-with-Kotlin-Azure-Functions-SignalR), [Powerpoint Slides](https://github.com/gloveboxes/Going-Serverless-with-Kotlin-Azure-Functions-SignalR/blob/master/docs/Kotlin%20Serverless%20with%20Azure%20Functions%20Final%20-%20March%202019.pptx), [PDF Slides](https://github.com/gloveboxes/Going-Serverless-with-Kotlin-Azure-Functions-SignalR/blob/master/docs/Kotlin%20Serverless%20with%20Azure%20Functions%20Final%20-%20March%202019.pdf)

## Solution Overview

![solution overview](https://raw.githubusercontent.com/gloveboxes/Going-Serverless-with-Kotlin-Azure-Functions-SignalR/master/docs/resources/solution-architecture.png)

This solution diagram overviews a typical IoT solution. [Azure IoT Hub](https://docs.microsoft.com/azure/iot-hub?WT.mc_id=iot-0000-dglover) is responsible for internet scale, secure, bi-directional communication with devices and backend services.

Telemetry can be [routed](https://docs.microsoft.com/azure/iot-hub/tutorial-routing?WT.mc_id=iot-0000-dglover) by Azure IoT Hub to various services and also to storage in [Apache Avro](https://avro.apache.org/docs/current/) or JSON format for purposes such as audit, integration or driving machine learning processes.

This posting takes a slice of this scenario and is about the straight through [serverless](https://en.wikipedia.org/wiki/Serverless_computing) processing of telemetry from Azure IoT Hub, via Kotlin Azure Functions and Azure SignalR for a near real-time dashboard.

### Azure Services

The following Azure services are used in this solution and available in Free  tiers: [Azure IoT Hub](https://docs.microsoft.com/azure/iot-hub?WT.mc_id=iot-0000-dglover), [Azure Functions](https://docs.microsoft.com/azure/azure-functions?WT.mc_id=iot-0000-dglover), [Azure SignalR](https://docs.microsoft.com/azure/azure-signalr?WT.mc_id=iot-0000-dglover), [Azure Storage](https://docs.microsoft.com/azure/storage?WT.mc_id=iot-0000-dglover), [Azure Storage Static Websites](https://docs.microsoft.com/azure/storage/blobs/storage-blob-static-website?WT.mc_id=iot-0000-dglover)

You can sign up for a [Free Azure Account](https://azure.microsoft.com/free?WT.mc_id=iot-0000-dglover), if you are a student then be sure to sign up for [Azure for Students](https://azure.microsoft.com/free/students?WT.mc_id=iot-0000-dglover), no credit card required.

## Developing Kotlin Azure Functions

The [Creating Kotlin based Azure Function with IntelliJ ](https://dev.to/azure/creating-kotlin-based-azure-function-with-intellij-486b) guide has all the information on insrallation requirements, JDKs, IntelliJ, Azure CLI etc and has a step by step guide to creating your first Kotlin Azure Functions with IntelliJ.

## Resources for Java and Kotlin Azure Functions

- [Create your first function with Java and Maven](https://docs.microsoft.com/azure/azure-functions/functions-create-first-java-maven?WT.mc_id=iot-0000-dglover)
- [Create your first Azure function with Java and IntelliJ](https://docs.microsoft.com/azure/azure-functions/functions-create-maven-intellij?WT.mc_id=iot-0000-dglover)
- [Announcing the general availability of Java support in Azure Functions](https://azure.microsoft.com/blog/announcing-the-general-availability-of-java-support-in-azure-functions?WT.mc_id=iot-0000-dglover)
- [Azure Functions triggers and bindings concepts](https://docs.microsoft.com/azure/azure-functions/functions-triggers-bindings?WT.mc_id=iot-0000-dglover)
- [Maven Plugin for Azure Functions](https://github.com/Microsoft/azure-maven-plugins/tree/develop/azure-functions-maven-plugin)
- [Azure Functions Java developer guide](https://docs.microsoft.com/azure/azure-functions/functions-reference-java?WT.mc_id=iot-0000-dglover)
- [ Java API for Azure Functions](https://docs.microsoft.com/java/api/com.microsoft.azure.functions.annotation?view=azure-java-stable%3FWT.mc_id%3Dgithub-blog-dglover&WT.mc_id=iot-0000-dglover)

## Solution Components (included in this GitHub repo)

1. Kotlin Azure Function. This Azure Function processes batches of telemetry, then calibrations and validates telemetry, and updates the Device State Azure Storage Table, and then passes the telemetry to the Azure SignalR service for near real-time web client update.

3. [Web Dashboard](https://enviro.z8.web.core.windows.net/enviromon.html). This Single Page Web App is hosted on Azure Storage as a Static Website. So it too is serverless.

## Design Considerations

### Optimistic Concurrency

First up, it is useful to understand [Event Hub Trigger Scaling](https://docs.microsoft.com/azure/azure-functions/functions-bindings-event-iot?WT.mc_id=iot-0000-dglover#trigger---scaling?WT.mc_id=github-blog-dglover) and how additional function instances can be started to process events.

I wanted to maintain a count in the Device State table of the number of times a device had sent telemetry. The solution implements [Azure Storage/CosmosDB Optimistic Concurrency](https://azure.microsoft.com/blog/managing-concurrency-in-microsoft-azure-storage-2?WT.mc_id=iot-0000-dglover).

[Optimistic Concurrency (OCC)](https://en.wikipedia.org/wiki/Optimistic_concurrency_control) assumes that multiple transactions can frequently complete without interfering with each other. While running, transactions use data resources without acquiring locks on those resources. Before committing, each transaction verifies that no other transaction has modified the data it has read. OCC is generally used in environments with low data contention.

If there are multiple functions instances updating and there is a clash, I have implemented Exponential Backoff and added a random factor to allow for retry.

    Pseudo code: random(occBase, min(occCap, occBase * 2 ^ attempt))

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

```kotlin
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

There is no Service-Side Azure SignalR SDK. To send telemetry from the Event Hub Trigger Azure Function to the Dashboard Web Client you need to call an HTTP Azure Function that is bound to the SignalR service. This SignalR Azure Function then sends the telemetry via SignalR as if the data was coming from a client-side app.

The flow for Azure SignalR integration is as follows:

1. The Web client makes a REST call to '**negotiate**', amongst other things, the SignalR '**Hubname**' is returned to the client.
2. The Web client then makes a REST call to '**getdevicestate**', this HTTP Trigger retrieves the state for all devices from the Device State Table. The data is returned to the client via SignalR via the same '**Hubname**' that was returned from the call to '**negotiate**'.
3. When new telemetry arrives via IoT Hub, the '**TelemetryProcessing**' trigger fires, the telemetry is updated in the Device State table and a REST call is made to the '**SendSignalRMessage**' and telemetry is sent to all the SignalR clients listening on the '**Hubname**' channel.

![](https://raw.githubusercontent.com/gloveboxes/Going-Serverless-with-Kotlin-Azure-Functions-SignalR/master/docs//resources/service-side-signalr.png)

## Set Up Overview

This lab uses free of charge services on Azure. The following need to be set up:

1. Azure IoT Hub and Azure IoT Device
2. Azure SignalR Service
3. Deploy the Kotlin Azure Function
4. Deploy the SignalR .NET Core Azure Function

### Step 1: Follow the Raspberry Pi Simulator Guide to set up Azure IoT Hub


[Setting up the Raspberry Pi Simulator](https://docs.microsoft.com/azure/iot-hub/iot-hub-raspberry-pi-web-simulator-get-started?WT.mc_id=iot-0000-dglover)

![raspberry Pi Simulator](https://docs.microsoft.com/azure/iot-hub/media/iot-hub-raspberry-pi-web-simulator/3_banner.png?WT.mc_id=iot-0000-dglover)

### Step 2: Create an Azure Resource Group

[az group create](https://docs.microsoft.com/cli/azure/group?view=azure-cli-latest&WT.mc_id=iot-0000-dglover#az-group-create&WT.mc_id=github-blog-dglover)

```bash
az group create -l westus -n enviromon-kotlin
```

### Step 3: Create a Azure Signal Service

- [az signalr create](https://docs.microsoft.com/cli/azure/signalr?view=azure-cli-latest&WT.mc_id=iot-0000-dglover#az-signalr-create&WT.mc_id=github-blog-dglover) creates the Azure SignalR Service
- [az signalr key list](https://docs.microsoft.com/cli/azure/ext/signalr/signalr/key?view=azure-cli-latest&WT.mc_id=iot-0000-dglover#ext-signalr-az-signalr-key-list&WT.mc_id=github-blog-dglover) returns the connection string you need for the SignalR .NET Core Azure Function.

```bash
az signalr create -n <Your SignalR Name> -g enviromon-kotlin --sku Free_DS2 --unit-count 1
az signalr key list -n <Your SignalR Name> -g enviromon-kotlin
```

### Step 4: Create a Storage Account

[az storage account create](https://docs.microsoft.com/cli/azure/storage/account?view=azure-cli-latest&WT.mc_id=iot-0000-dglover#az-storage-account-create&WT.mc_id=github-blog-dglover)

```bash
az storage account create -n enviromonstorage -g enviromon-kotlin -l westus --sku Standard_LRS --kind StorageV2
```

### Step 5: Clone the project

```bash
git clone https://github.com/gloveboxes/Going-Serverless-with-Kotlin-Azure-Functions-SignalR.git
```

### Step 6: Deploy the SignalR .NET Core Azure Function

```bash
cd  Going-Serverless-with-Kotlin-Azure-Functions-SignalR

cd iot-serverless-kotlin-azure-functions-signalr

cp local.settings.sample.json local.settings.json

```

### Step 7: Open the Kotlin Azure Functions Project with IntelliJ

From Terminal on Linux and macOS, or Powershell on Windows.

```bash

cd  Going-Serverless-with-Kotlin-Azure-Functions-SignalR

cd iot-serverless-kotlin-azure-functions-signalr

cp local.settings.sample.json local.settings.json

```

Start **IntelliJ** and open the project in the '**iot-serverless-kotlin-azure-functions-signalr**' folder.

### Step 8: Update the local.settings.json

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "",
    "FUNCTIONS_WORKER_RUNTIME": "java",
    "StorageConnectionString": "",
    "PartitionKey": "Sydney",
    "IotHubConnectionString": "",
    "AzureSignalRConnectionString": "",
    "AzureSignalRUrl": ""
  },
  "Host": {
    "LocalHttpPort": 7071,
    "CORS": "http://127.0.0.1:5500,http://localhost:5500",
    "CORSCredentials": true
  }
}
```

### Step 9: Deploy the Kotlin Azure Function

Open the '**Maven**' tab, run '**clean**', the '**package**', then '**azure-functions:deploy**'

![](https://raw.githubusercontent.com/gloveboxes/Going-Serverless-with-Kotlin-Azure-Functions-SignalR/master/docs/resources/azure-function-deploy.jpg)

### Step 10: Enable Static Websites for Azure Storage

The Dashboard project contains the Static Website project.

Follow the guide for [Static website hosting in Azure Storage](https://docs.microsoft.com/azure/storage/blobs/storage-blob-static-website?WT.mc_id=iot-0000-dglover).

The page used for this sample is enviromon.html. Be sure to modify the "apiBaseUrl" url in the web page javascript to point your instance of the SignalR Azure Function.

Copy the contents of the dashboard project to the static website.

### Step 11: Enable CORS for the SignalR .NET Core Azure Function

[az functionapp cors add](https://docs.microsoft.com/cli/azure/functionapp/cors?view=azure-cli-latest&WT.mc_id=iot-0000-dglover#az-functionapp-cors-add&WT.mc_id=github-blog-dglover)

```bash
az functionapp cors add -g enviromon-kotlin -n <Your SignalR Function Name> --allowed-origins <https://my-static-website-url>
```

### Step 12: Start the Dashboard

From your web browser, navigate to https://your-start-web-site/enviromon.html

The telemetry from the Raspberry Pi Simulator will be displayed on the dashboard.

![dashboard](https://raw.githubusercontent.com/gloveboxes/Going-Serverless-with-Kotlin-Azure-Functions-SignalR/master/docs//resources/dashboard.png)