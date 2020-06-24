# Introduction to Azure IoT hub

![https://unsplash.com/photos/zDxlNcdUzxk](../.gitbook/assets/image%20%2887%29.png)

Azure IoT hub is a managed service for basically telemetry collection, device management and security and other features required to make developing IoT solutions as easy as possible. 

Main roles of IoT hub are:

* Central messaging for bi-directional communication in the cloud
* Provisioning, identity registry and device management
* Security
* Message routing and enrichment
* Monitoring & diagnostics

As IoT hub is the starting point for any IoT solution regardless if there is edge capabilities or not, we have to get some hands on with basic features of IoT hub.

### Telemetry ingestion using CLI

The first thing we can try is telemetry ingestion. There is no need to have a physical device or to write an application. Azure CLI can be used to simulate a device sending messages to IoT hub.

As the hub we have is pretty empty with no devices, it's obvious that at least one device has to be registered first. A device registration is required for any physical/simulated device to interact with IoT hub.

Let's create our first device!

`iot hub device-identity create --hub-name master-hub --device-id simulated-device`

![](../.gitbook/assets/image%20%28106%29.png)

IoT hub extension in the CLI can simulate a device sending messages to IoT hub. Run the following command to simulate sending one message per second till a 1000 messages are sent. We will not wait till all messages are sent though.

`iot device simulate --hub-name master-hub --device-id simulated-device --msg-count 1000 --msg-interval 1`

Go to Azure portal Iot Hub page, click on **Metrics** in monitoring section**.** Select **Total number of messages** as the metric and **Average** as the aggregation. ****There might be some delay in metrics rendering but eventually there will be a spike of message received.

![](../.gitbook/assets/image%20%28202%29.png)

### Telemetry ingestion using VS Code

IoT extension in VS Code has an easy way to ingest device to cloud messages as well. From **Azure IoT Hub** section in explorer side bar, click on the ellipses button and select **Send D2C Message to IoT Hub.**

![](../.gitbook/assets/image%20%2844%29.png)

A window will open to:

1. Select a device or group of devices
2. Specify number of messages to send and interval between them
3. Message payload which can be plain text or [dummy JSON](https://github.com/webroo/dummy-json) template.

![](../.gitbook/assets/image%20%2816%29.png)

### Telemetry ingestion using SDK

Ingesting data or telemetry into the IoT hub will normally be done using an application/module running on an IoT device. Microsoft provides SDKs for interacting with IoT hub. The SDKs are divided into:

1. **Device SDK** enables building applications at device side for things like sending messages to IoT hub and optionally receive backend messages/updates.
2. **Service SDK** very similar to device SDK but runs at backend side.
3. **Provisioning SDKs** are for provisioning devices using _Device Provisioning Service_

Now we will build a .NET Core console application to ingest telemetry into IoT hub using C\# and device SDK which is provided as a nuget package. Before building the application we need to have connection string of the simulated device.

There are many connection strings involved in IoT applications:

* **Device connection string** is used by applications running at device itself
* **Service connection string** is used by backend applications
* **Hub/Admin connection string** is used to manage IoT hub itself

All these connections strings can be retrieved from Azure portal or using CLI. Sometimes a connection string applies to the whole hub like admin/service connection strings and sometimes it applies to a specific device.

Run the following command to obtain device connection string for the simulated device we have.

`az iot hub device-identity show-connection-string --hub-name master-hub --device-id simulated-device` 

![](../.gitbook/assets/image%20%28153%29.png)

Keep a note of this connection string because we will use it shortly.

Now let's create our .NET Core console app.

* Create a .NET Core console application and install device SDK nuget package and make sure it compiles.

```text
dotnet new console --name TelemetryApp
cd TelemetryApp
dotnet add package Microsoft.Azure.Devices.Client
dotnet add package Newtonsoft.Json
dotnet build
```

* Then open `TelemetryApp` folder using VS Code.
* Replace the contents of `Program.cs` with the following snippet and update the connection string variable with your device connection string.

```csharp
using System;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Azure.Devices.Client;
using Newtonsoft.Json;

namespace TelemetryApp
{
    class Program
    {
        static async Task Main(string[] args)
        {
            string deviceConnectionString = "YOUR-DEVICE-CONNECTION-STRING";
            using (var deviceClient = DeviceClient.CreateFromConnectionString(deviceConnectionString, TransportType.Amqp))
            {
                await SendEventAsync(deviceClient, 100).ConfigureAwait(false);
            }
        }

        static async Task SendEventAsync(DeviceClient _deviceClient, int messageCount)
        {
            Console.WriteLine("Device sending {0} messages to IoTHub...", messageCount);
            var random = new Random();
            var temperatureThreshold = 30;

            for (int i = 0; i < messageCount; i++)
            {
                var message = new {
                    MessageId = Guid.NewGuid(),
                    Temprature = random.Next(20, 35),
                    Humidity = random.Next(60, 80),
                    Timestamp = DateTime.UtcNow
                };

                var dataBuffer = JsonConvert.SerializeObject(message);

                using (var eventMessage = new Message(Encoding.UTF8.GetBytes(dataBuffer)))
                {
                    eventMessage.Properties.Add("temperatureAlert", (message.Temprature > temperatureThreshold) ? "true" : "false");
                    Console.WriteLine($"{DateTime.Now.ToLocalTime()}> Sending message: {i}, Data: [{dataBuffer}]");

                    await _deviceClient.SendEventAsync(eventMessage).ConfigureAwait(false);
                    Thread.Sleep(1000);
                }
            }
        }
    }
}

```

* Run the application using Ctrl+F5 or `dotnet run` from VS terminal window

![](../.gitbook/assets/image%20%28201%29.png)

* The application simply creates a message object \(anonymous type\) in C\#, converts it to JSON and sends the binary equivalent value to IoT hub along with some meta property indicating if the temperature is above certain threshold.
* Azure portal metrics page can be used as before to see how many messages ingested into the hub but there is an easier way in VS Code itself.
* While the application is running, right click the device node in Azure IoT hub pane, and select **Start Monitoring Built-in Event Endpoint**

![](../.gitbook/assets/image%20%28160%29.png)

* This effectively monitors all messages landing  at IoT hub and displays them in VS Code

![](../.gitbook/assets/image%20%2845%29.png)

* All these messages land in IoT Hub backend which wraps an Azure Event Hubs instance so any consumer app/service can connect to this event hub and consume those messages using Event Hub APIs.
* Actually, Azure IoT hub provides a page in portal to show the underlying event hub details. You can grab event hub compatible endpoint and use it in your application as a plain event hub. There is the classic concept of consumer groups so that you can have independent readers.

![](../.gitbook/assets/image%20%2894%29.png)

* Messages can be also routed to other destinations like Azure Blob Storage or Event Grid but that's beyond the scope of this exercise 😀.

In this page we have seen how to register a device in IoT hub, simulate sending messages from the device using CLI and wrote an application that can be deployed to a device to collect and send telemetry.

That's was pretty easy!

Now it's time to live on the edge and move on to Azure IoT edge.

