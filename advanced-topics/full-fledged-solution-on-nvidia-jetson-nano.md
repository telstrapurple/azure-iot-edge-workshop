# Full fledged solution on NVIDIA Jetson nano

Well, all the previous examples were done using a virtual edge device. I bet that was not extremely exciting. But now it's time to play with a real device. [NVIDIA Jetson Nano](https://developer.nvidia.com/embedded/jetson-nano-developer-kit) is a small computer with enough power to run AI models on the device directly and hence is a perfect device to run IoT Edge solution. It has  a GPU not as powerful as the famous GTX/etc family but powerful enough to run things like object classification and detection for multiple video streams in real-time. Also NVIDIA has an SDK called [DeepStream SDK ](https://developer.nvidia.com/deepstream-sdk)making video analytics easier than you would expect. 

![https://developer.nvidia.com/embedded/jetson-nano-developer-kit](../.gitbook/assets/image%20%2883%29.png)



The solution we are going to implement is basically identifying objects using a webcam connected to the device and showing object names/counts in a Power BI real-time dashboard. It works as follows:

1. An NVIDIA module processes one or more video streams and scores what objects are seen in those streams.
2. Another module takes the scoring results and parses them and finally pushes them to a Power BI streaming dataset which drives a real-time dashboard.

![](../.gitbook/assets/image%20%2848%29.png)

### Prepare the device

The device has to be prepared first with needed tools and connecting it with IoT hub.

* Follow [instruction from NVIDIA](https://developer.nvidia.com/embedded/learn/get-started-jetson-nano-devkit) to install the OS on the device SD card
* Register an IoT Edge device in the portal or CLI and make a note of connection string
* SSH into the device
* Install [IoT Edge runtime and wire up the connection string](https://docs.microsoft.com/en-us/azure/iot-edge/how-to-install-iot-edge-linux)
* For better performance with headless operation
  * Make sure the device runs in 10W power mode `sudo nvpmodel -m 0` 
  * Disable desktop interface `sudo systemctl set-default multi-user.target` 
* Make sure JetPack 4.4 is installed `sudo apt-cache show nvidia-jetpack` 

![](../.gitbook/assets/image%20%28207%29.png)

* Install [DeepStream SDK](https://docs.nvidia.com/metropolis/deepstream/dev-guide/index.html)
* Verify correct installation of DeepStream SDK. Navigate to samples folder of SDK and check SDK version. `deepstream-app --version` 

![](../.gitbook/assets/image%20%2872%29.png)

### Create IoT Edge solution with NVIDIA module

Now we will create edge solution with a market place module from NVIDIA doing object detection against sample video files.

* Create a new IoT Edge solution named **VideoAnalyticsSolution**
* From the dropdown to select module to add, pick the marketplace option
* Search for **NVIDIA** and pick **NVIDIA DeepStream SDK** 
* Make sure to pick any plan with **Jetson** in its name and click **Import**

![](../.gitbook/assets/image%20%28136%29.png)

* Unfortunately the module in marketplace is not updated for the latest SDK v5.0 so we will need to tweak the module element in deployment template file. The next few changes are based on NVIDIA documentation.
  * [DeepStream-l4t](https://ngc.nvidia.com/catalog/containers/nvidia:deepstream-l4t) module which is the one designed for IoT \(login required but it's easy to register\)
  * [All modules catalog](https://ngc.nvidia.com/catalog/containers?orderBy=modifiedDESC&pageNumber=1&query=&quickFilter=containers&filters=) this has heaps of other modules from NVIDIA \(not directly related to IoT and Edge\)
* Open `deployment.template.json` file and change module image from `marketplace.azurecr.io/nvidia/deepstream-iot-l4t:latest` to `nvcr.io/nvidia/deepstream-l4t:5.0-dp-20.04-iot` 
* Change working directory to `/opt/nvidia/deepstream/deepstream-5.0/sources/apps/sample_apps/deepstream-test5/configs/` 



DeepStream SDK works as a pipeline of prepacked steps that can be configured in a flexible way to achieve a sophisticated outcome. Configuration is done in a text file that controls what plugins are included in the pipeline and settings of each one of them.

![](../.gitbook/assets/image%20%28211%29.png)



To have a look on \(an identical\) config file used in our module, run the following:

```text
nano /opt/nvidia/deepstream/deepstream-5.0/sources/apps/sample_apps/deepstream-test5/configs/test5_config_file_src_infer_azure_iotedge.txt
```

You can see a couple file-based video sources used in this sample.

![](../.gitbook/assets/image%20%2863%29.png)

Also one of the pipeline sinks is of type message broker which sends result to edgeHub and then IoT hub.

![](../.gitbook/assets/image%20%2891%29.png)

Build solution and deploy to device and make sure module is started.

![](../.gitbook/assets/image%20%28197%29.png)

Now inspect the logs of NVIDIA module `iotedge logs -f --tail 10 NVIDIADeepStreamSDK` .

![](../.gitbook/assets/image%20%2868%29.png)

Good, seems it's working and later we will have a look on the meaning of such data but at least it has object labels and what appears to be bounding boxes.

From VS Code, monitor built-in events endpoint for Jetson Nano device. This proves that the module works fine doing AI model scoring and also communicating the results to edgeHub and IoT hub ultimately.

![](../.gitbook/assets/image%20%2867%29.png)

Run `sudo systemctl stop iotedge`  to stop IoT Edge because otherwise a flood of messages will be sent to the cloud and may trigger some throttling actions making problems with the remaining steps.

### Use our own config and visualise results

In this section we will see how to customise DeepStream SDK sample application and tweak config file to visualise results using VLC video player. Most of the settings to be tweaked are documented [here](https://docs.nvidia.com/metropolis/deepstream/5.0/dev-guide/index.html#page/DeepStream_Development_Guide/deepstream_app_config.3.2.html#wwpID0E04B0HA). We will build a new docker image based on the one provided by NVIDIA. This is one of the benefits of layered file system in docker. You can take one image and build on top of it a small layer of customisation.

* Change platform target to **arm64v8**, as we are going to build our own custom module we have to specify the architecture matching our target device.
* Copy a `.env` file from any previous solution and paste it at the root of current solution. Environment files would look like the following snippet.

```text
CONTAINER_REGISTRY_USERNAME=[YOUR-USERNAME]
CONTAINER_REGISTRY_PASSWORD=[YOUR-PASSWORd]
CONTAINER_REGISTRY_ADDRESS=[YOUR-CONTAINER-REGISTRY]
```

* Create a folder called **modules** at the root of solution and then another folder called **deepstreammodule** inside it.
* Add a `module.json` file with the following contents inside module folder.

```javascript
{
    "$schema-version": "0.0.1",
    "description": "",
    "image": {
        "repository": "${CONTAINER_REGISTRY_ADDRESS}/deepstreammodule",
        "tag": {
            "version": "0.0.1",
            "platforms": {
                "arm64v8": "./Dockerfile.arm64v8"
            }
        },
        "buildOptions": [],
        "contextPath": "./"
    }
}

```

* Create a `Dockerfile.arm64v8` file inside module folder. The core of this file is to create a new image based off NVIDIA's image and place a new file `rtsp_config.txt` into configs folder.

```text
FROM nvcr.io/nvidia/deepstream-l4t:5.0-dp-20.04-iot

ADD rtsp_config.txt /opt/nvidia/deepstream/deepstream-5.0/sources/apps/sample_apps/deepstream-test5/configs

CMD ["/bin/bash"]

WORKDIR /opt/nvidia/deepstream/deepstream-5.0
```

* Copy config file to your development machine and then move it into module folder and rename it to `rtsp_config.txt` .

```text
scp purple@192.168.0.30:/opt/nvidia/deepstream/deepstream-5.0/sources/apps/sample_apps/deepstream-test5/configs/test5_config_file_src_infer_azure_iotedge.txt .
```

* Open this config file and disable the first sink by making its enable setting = 0 

```text
[sink0]
enable=0
```

* Add a new [RTSP ](https://tools.ietf.org/html/rfc2326)sink

```text
[sink3]
enable=1
#Type - 1=FakeSink 2=EglSink 3=File 4=RTSPStreaming
type=4
#1=h264 2=h265
codec=1
sync=0
bitrate=4000000
# set below properties in case of RTSPStreaming
rtsp-port=8554
udp-port=5400
```

* Find **\[primary-gie\]** and change **interval** from **0 to 5**. This is the number of consecutive batches to skip for inference which makes the app a bit more responsive.
* Go to **\[tests\]** section and change **file-loop** to **1** to make DeepStream continuously loop over the test video files.
* Save the file.
* Open deployment template file,  and replace the `NVIDIADeepStreamSDK` element with the following. The module has been renamed to match the module in the new folder, also image tag dynamically points to the new custom module in your own container registry. Config file name points to the updated config in the image and some networking bits are there to publish RTSP port from the container to the host.

```javascript
"deepstreammodule": {
  "version": "1.0",
  "type": "docker",
  "status": "running",
  "restartPolicy": "always",
  "settings": {
    "image": "${MODULES.deepstreammodule}",
    "createOptions": {
      "Entrypoint": [
        "/usr/bin/deepstream-test5-app",
        "-c",
        "rtsp_config.txt",
        "-p",
        "1",
        "-m",
        "1"
      ],
      "ExposedPorts": {
        "8554/tcp": {}
      },
      "NetworkingConfig": {
        "EndpointsConfig": {
          "host": {}
        }
      },
      "HostConfig": {
        "runtime": "nvidia",
        "NetworkMode": "host",
        "PortBindings": {
          "8554/tcp": [
              {
                 "HostPort": "8554"
              }
          ]
        }
      },
      "WorkingDir": "/opt/nvidia/deepstream/deepstream-5.0/sources/apps/sample_apps/deepstream-test5/configs/"
    }
  }
}
```

* Update routing element to use the correct module name.

```text
"routes": {
   "DeepstreamToIoTHub": "FROM /messages/modules/deepstreammodule/outputs/* INTO $upstream"
}
```

* Build and publish the solution.
* Deploy the solution to the device and remember to pick the **arm64** deployment file.

Make sure messages still land on IoT hub or inspect module logs on the device directly.

Now time for some fun, open VLC and play this video stream `rtsp://192.168.0.30:8554/ds-test` .

Replace the IP with your Jetson Nano IP and  tada 🎉. It's the same video file repeated 4 times but a

![](../.gitbook/assets/image%20%2829%29.png)

That's amazing! Even though Jetson Nano is running headless, we can still view video stream\(s\) analysed and the outputs of AI model visualised on an RTSP output stream.

Next, lets add some Power BI sauce to the recipe.

### Create a Power BI streaming dataset

Power BI streaming dataset allows us to push data over HTTP and have it ready to visualise in a live dashboard. 

* Create a new workspace

![](../.gitbook/assets/image%20%2836%29.png)

* In the new workspace, create a streaming dataset and select type **API** in the next screen.

![](../.gitbook/assets/image%20%28176%29.png)

* Configure the dataset as follows.

![](../.gitbook/assets/image%20%28174%29.png)

* Make a note of the push URL and sample json payload and clock **Done**.
* We will create a Power BI dashboard later.

### Create a C\# module to parse video analytics results

To refresh our memory, the scoring results done by NVIDIA magic looks like the next snippet of JSON .

```javascript
{
  "version" : "4.0",
  "id" : 418,
  "@timestamp" : "2020-06-13T06:59:42.757Z",
  "sensorId" : "HWY_20_AND_LOCUST__EBA__4_11_2018_4_59_59_508_AM_UTC-07_00",
  "objects" : [
    "345|441.6|510|470.4|562.5|Bicycle",
    "352|480|510|521.6|577.5|Bicycle",
    "195|403.2|476.25|428.8|543.75|Person"
   ]
}
```

There is some funky sensor \(camera\) id, and a group of objects identified. Each object has:

* An object id which is a unique tracking id per each object across different frames. In NVIDIA terminology it's called tracking id and the SDK has different plugins for tracking objects.
*  Bounding box details.
* Object label.

I got the above info from NVIDIA documentation of test program we are using.

The role of this C\# module we are going to implement is to parse this JSON payload and send it to an Azure Stream Analytics module which will group objects in a certain time window and report how many objects are found in that window taking into consideration tracking IDs.

* Add a new C\# module to the solution and name it `DeepPayloadProcessorModule` , lengthy but still catchy 😂
* Select the correct Azure container registry to host images for this module.
* Open `Program.cs` and remove `PipeMessage` method and its wiring in `Init` method
* Add wiring to a new method in `Init` method

```csharp
await ioTHubModuleClient.SetInputMessageHandlerAsync("inferences", ReshapeMessageAndSendToASA, ioTHubModuleClient);
```

* Add the following method which is mainly parsing DeepStream JSON message and sending it back to edgeHub in a more tabular format that can be consumed by ASA. Add the missing namespace usings if needed.

```csharp
static async Task<MessageResponse> ReshapeMessageAndSendToASA(Message message, object userContext)
{
    System.Console.WriteLine("Received message from DeepStream!");

    var moduleClient = userContext as ModuleClient;
    if (moduleClient == null)
    {
        throw new InvalidOperationException("UserContext doesn't contain " + "expected values");
    }

    byte[] messageBytes = message.GetBytes();
    string messageString = Encoding.UTF8.GetString(messageBytes);
    if (string.IsNullOrWhiteSpace(messageString))
    {
        System.Console.WriteLine("Empty message!");
    }
    else
    {
        var parsed = JObject.Parse(messageString);
        var timestamp = parsed["@timestamp"].ToString();
        var sensorId = (parsed["sensorId"].ToString().Replace("HWY_20_AND_", "")).Substring(0,12);
        var objects = parsed["objects"] as JArray;

        var result = new List<Object>();
        foreach(var object_str in objects)
        {
            var parts = object_str.ToString().Split("|");
            result.Add(
                new {
                    timestamp = timestamp,
                    sensorId = sensorId,
                    trackingId = parts[0],
                    objectLabel = parts[5]
                }
            );
        }
        var payload = JsonConvert.SerializeObject(result);
        using (var pipeMessage = new Message(Encoding.UTF8.GetBytes(payload)))
        {
            await moduleClient.SendEventAsync("reshaped", pipeMessage);
            Console.WriteLine("=========== Reshaped DeepStream payload to ASA ==============");
            System.Console.WriteLine(payload);
        }
    }
    return MessageResponse.Completed;
}
```

* Update deployment template routes to route messages from NVIDIA module into the new C\# module and back into IoT hub. Note that plain inferences from NVIDIA module are not sent to the cloud anymore.

```text
"routes": {
    "DeepPayloadProcessorModuleToIoTHub": "FROM /messages/modules/DeepPayloadProcessorModule/outputs/* INTO $upstream",
    "DeepstreamToOutputProcessor": "FROM /messages/modules/deepstreammodule/outputs/* INTO BrokeredEndpoint(\"/modules/DeepPayloadProcessorModule/inputs/inferences\")"
}
```

* Build and deploy the solution.

Run `iotedge logs -f --tail 40 DeepPayloadProcessorModule` to inspect the logs of the new module.

![](../.gitbook/assets/image%20%2858%29.png)

You can also monitor messages sent to IoT hub to confirm C\# module works as expected.

### Add Azure Stream Analytics module

I will be brief here as we have went through enough details on how to  use ASA. The gist is to have a query to group objects found in last time window based on tracking Id.

```sql
SELECT 
    sensorId, System.Timestamp() AS windowEnd, objectLabel, COUNT (DISTINCT "trackingId") as count  
INTO
    [output]
FROM
    [input] TIMESTAMP BY [timestamp]

GROUP BY sensorId, objectLabel, TumblingWindow(Duration(second, 2))
```

Also make sure the output is formatted as an array not as line separated.

![](../.gitbook/assets/image%20%2832%29.png)

Once ASA job is created and published to storage account, add it to the solution.

Update routing to point output of processor module to input of ASA module.

```javascript
"routes": {
    "DeepstreamToPayloadProcessor": "FROM /messages/modules/deepstreammodule/outputs/* INTO BrokeredEndpoint(\"/modules/DeepPayloadProcessorModule/inputs/inferences\")",
    "PayloadProcessorToASA": "FROM /messages/modules/DeepPayloadProcessorModule/* INTO BrokeredEndpoint(\"/modules/VideoAnalyticsASAModule/inputs/input\")",
    "ASAToIoTHub": "FROM /messages/modules/VideoAnalyticsASAModule/outputs/* INTO $upstream"
}
```

Fix ASA image URL to pick an arm image **mcr.microsoft.com/azure-stream-analytics/azureiotedge:1.0.6-linux-arm32v7** otherwise ASA module cannot be loaded on Jetson Nano device.

Build and deploy solution.

Run `iotedge logs -f --tail 40 VideoAnalyticsASAModule` to inspect logs of ASA module.

![](../.gitbook/assets/image%20%28131%29.png)

In Azure CLI window, run `iot hub monitor-events --hub-name master-hub --consumer-group $default --device-id jetson` to monitor events sent to IoT hub.

![](../.gitbook/assets/image%20%28179%29.png)

Azure CLI formats the JSON payload but if monitored from VS Code we can simply see the JSON item array that can be submitted as is to Power BI Streaming dataset.

![](../.gitbook/assets/image%20%28111%29.png)

### Push Analytics to Power BI

Edit `program.cs` file of `DeepPayloadProcessorModule` and add this line to Init method.

```csharp
await ioTHubModuleClient.SetInputMessageHandlerAsync("topowerbi", SendToPowerBI, ioTHubModuleClient);
```

Add the following method but update the URL with your own streaming dataset URL. We can normally use environment variables and send this URL as a parameter to the container but for the sake of simplicity, a hard-coded value will do the job.

```csharp
static async Task<MessageResponse> SendToPowerBI(Message message, object userContext)
{
    Console.WriteLine($"{DateTime.Now.ToShortTimeString()}: About to send message to Power BI");
    var client = new HttpClient();
    var powerBIDatasetUrl = "[YOUR-STREAMING-DATASET-URL]";
    byte[] messageBytes = message.GetBytes();
    var payload =  new ByteArrayContent(messageBytes);
    System.Console.WriteLine(Encoding.UTF8.GetString(messageBytes));
    await client.PostAsync(powerBIDatasetUrl, payload);
    Console.WriteLine($"{DateTime.Now.ToShortTimeString()}: Sent message to Power BI");
    return MessageResponse.Completed;
}
```

This method is very straightforward, it take a message generated by ASA job and pushes it to Power BI. The simplicity comes from the fact that ASA output schema matches Power BI dataset schema.

Open `module.json` of `DeepPayloadProcessorModule` and increase the version of version element inside the tag element.  This forces IoT Edge runtime to pick the newer image from repository.

Update routes in deployment template file to feed the output of ASA module into the new input we implemented above that will push the message to Power BI dataset.

```csharp
"routes": {
    "DeepstreamToPayloadProcessor": "FROM /messages/modules/deepstreammodule/outputs/* INTO BrokeredEndpoint(\"/modules/DeepPayloadProcessorModule/inputs/inferences\")",
    "PayloadProcessorToASA": "FROM /messages/modules/DeepPayloadProcessorModule/* INTO BrokeredEndpoint(\"/modules/VideoAnalyticsASAModule/inputs/input\")",
    "ASAToIoTHub": "FROM /messages/modules/VideoAnalyticsASAModule/outputs/* INTO BrokeredEndpoint(\"/modules/DeepPayloadProcessorModule/inputs/topowerbi\")"
}
```

Build and deploy the latest changes to the device. Monitoring the messages from processor module will show traces of sending data to Power BI. 

_P.S. I have cleaned up other logging lines from the same module to keep the screenshot clean and easy to understand._

![](../.gitbook/assets/image%20%28129%29.png)

Open Power BI workspace and do the following steps to create a streaming dashboard

* Create a new dashboard
* Add real-time data tile

![](../.gitbook/assets/image%20%28141%29.png)



* Pick the streaming dataset defined before.
* Select clustered bar chart visualisation and configure it as follows.

![](../.gitbook/assets/image%20%289%29.png)

* Finalise by providing the title to the tile.

The tile will auto refresh every couple seconds and show count of objects identified.

![](../.gitbook/assets/image%20%2895%29.png)

### Use a CSI/USB camera

 It's good to play with pre-recorded videos but it's more fun to use a real camera and see how this solution works in real life. I will use a CSI camera on the Jetson device to continue with this sample.

* Run `ls /dev/video*` to inspect installed cameras

![](../.gitbook/assets/image%20%2818%29.png)

* Run `v4l2-ctl -d /dev/video0 --list-formats-ext` to inspect video modes available on a certain camera. This helps with what resolution and frames per second to use in a later step.

![](../.gitbook/assets/image%20%28128%29.png)

* Edit `rtsp_config.txt` and disable source 1 and change source 0 as follows to:
  * Switch from a recorded video file to a real camera \(CSI one\)
  * Configure resolution
  * Configure frames per second

```text
[source0]
enable=1
#Type - 1=CameraV4L2 2=URI 3=MultiURI 4=RTSP 5=CSI
type=5
camera-width=3264
camera-height=2464
camera-fps-n=21
camera-fps-d=1
camera-csi-sensor-id=0
# (0): memtype_device   - Memory type Device
# (1): memtype_pinned   - Memory type Host Pinned
# (2): memtype_unified  - Memory type Unified
cudadec-memtype=0
```

* Increment image version in `deepstreammodule/module.json` 
* Add the following snippet to **HostConfig** element of DeepStream module in `deployment.template.json` to allow the container to access CSI camera.

```text
"Binds": ["/tmp/argus_socket:/tmp/argus_socket"],
"IpcMode" : "host",
"Devices": [
{
    "PathOnHost": "/dev/video0",
    "PathInContainer": "/dev/video0",
    "CgroupPermissions": "mrw"
}]
```

* Build and deploy the solution to the device.

{% hint style="info" %}
If **edgeHub** is flooded with messages from the previous exercises using the video file sources, you can delete **edgeHub** container and restart IoT edge.

`sudo docker rm -f edgeHub` 

`sudo systemctl restart iotedge` 
{% endhint %}

Move around in front of the camera and check NVIDIA/C\# module logs. They should indicate a person object. Also open VLC and make sure you are being tracked as expected. The model linked with the config file we use is designed to identify people, vehicles and road signs. Open Power BI dashboard and make sure it reflects what the camera sees.

{% hint style="info" %}
If the camera cannot identify any objects, ASA job will not output any records and Power BI dashboard will just render the state of the last message received. C\# module can be improved to cater for this case by sending an empty message if nothing is received from ASA based on some timing logic or something. I haven't tried it but just a guess.
{% endhint %}

