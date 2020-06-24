# Azure Stream Analytics

![https://unsplash.com/photos/JKUTrJ4vK00](../.gitbook/assets/image%20%28134%29.png)

It's time now to stand on the shoulder of the giants. Say temperature sensor module is continuously collecting measurements and availing them to any other module on the device to consume \(or to edgeHub to submit directly to the cloud\). In our case, it's better do any required analysis locally and submit only insights to the cloud but not just raw data.

Assume the requirement is to analyze temperature measurements and do something if the average reading for the last **x** seconds/minutes goes beyond a certain threshold. This is easily done using Azure Stream Analytics. Fortunately many Azure services can run in containers including some cognitive services, stream analytics even plain SQL server and the new SQL database edge can run in containers.

Let's solve this problem using Azure Stream Analytics.

### Prepare Stream Analytics job

A stream analytics job can normally run in the cloud or can run on an IoT Edge device. To use it on a device, it has first to be prepared in the cloud at least to have metadata about input/output and query available to be pushed to the device.

* Open Azure portal.
* Create a new stream analytics job in same resource group but configure it as an edge job.

  An edge job runs on the device and does not incur Azure cost.

![](../.gitbook/assets/image%20%2819%29.png)

* Go to the job once created.
* In the inputs, add and **Edge Hub** input and give an alias on **input**

![](../.gitbook/assets/image%20%2838%29.png)

* In the output sections, do the same and add an Edge Hub output called **output**
* Edit the query section to have this query and save it.

```sql
SELECT 'reset' AS command
INTO output
FROM input 
TIMESTAMP BY timeCreated
GROUP BY TumblingWindow(second, 10)
HAVING Avg(machine.temperature) > 30
```

* The input/output and grouping parts are obvious. The new bit for an ASA \(Azure Stream Analytics\) query is the time window. The query will be executed in the context of data received in a certain time window. If average machine temperature is greater than threshold for the specified time window, a new record will be inserted in the output table which in our case means a message will be dropped into IoT Edge hub on the device.
* This message can be used for:
  * Control the device itself. The simulated temperature sensor is designed to be [reset if a certain message is received on a certain input](https://github.com/Azure/iotedge/blob/master/edge-modules/SimulatedTemperatureSensor/src/Program.cs#L121).
  * The message itself can be submitted to IoT hub in the cloud as a log entry to what important events happen on the device.
* Now ASA job is almost ready to function on the device. But there is an extra step to export the metadata of the job to some storage container to be consumed from the device. The device will run an ASA container and pull job configuration from that storage container.
* Go to **storage account settings** tab and click **Add storage account**
* Link the job with an existing storage account in same subscription \(create one if you don't have any\) and fill a name for a new container or pick an existing container and click **Save**.

![](../.gitbook/assets/image%20%28162%29.png)

* Once done click the **publish** button on the side menu and then click the other **Publish** button.
* Wait until Azure save the changes and provides a SAS URL for the job config blob. Copy that URL as it will be used later.

### IoT edge solution with ASA job

Now we will use VS Code to create a new edge solution and configure it with the simulated temperature sensor and also add an ASA module.

* Open VS Code and create a new IoT Edge solution.
* When prompted to select the module template, pick Azure Stream Analytics module.

![](../.gitbook/assets/image%20%28147%29.png)

* Give the module a name like `StreamingModule` and hit ENTER
* VS Code will next show all edge streaming job configured in your subscription

![](../.gitbook/assets/image%20%28101%29.png)

* Pick the correct job if you have more than one and then wait until VS Code grabs the metadata of the job.

![](../.gitbook/assets/image%20%2877%29.png)

* Main items changed in deployment template now are:
  * A single non-system module pointing to `mcr.microsoft.com/azure-stream-analytics/azureiotedge:1.0.6` . 
  * A **StreamingModule** element at the end of the file with information about how to pull job information from Azure.

{% hint style="info" %}
If you want to confirm what images are available on the relevant repository, we can use Docker REST API to [list the tags](https://docs.docker.com/registry/spec/api/#tags). For example, to grab the tags for ASA images navigate to [https://mcr.microsoft.com/v2/azure-stream-analytics/azureiotedge/tags/list](https://mcr.microsoft.com/v2/azure-stream-analytics/azureiotedge/tags/list).  That would be very useful if you want to confirm that an image for a specific platform like arm64 is available or not.
{% endhint %}

* It's time now to add simulated temperature sensor module. Use command palette to add a new edge module. Select module from marketplace option and then in the search screen select **Simulated Temperature Sensor** and import it. 
* At the bottom of deployment template file, change **SendInterval** of this module to be 2 seconds.

```javascript
"SimulatedTemperatureSensor": {
  "properties.desired": {
    "SendData": true,
    "SendInterval": 2
  }
}
```

* Update routing in deployment template as follows but adjust module names if needed.

```javascript
"routes": {          
    "FromSensorToAsa": "FROM /messages/modules/SimulatedTemperatureSensor/* INTO BrokeredEndpoint(\"/modules/StreamingModule/inputs/input\")",
    "FromAsaToSensor": "FROM /messages/modules/StreamingModule/* INTO BrokeredEndpoint(\"/modules/SimulatedTemperatureSensor/inputs/control\")",
    "FromAsaToCloud": "FROM /messages/modules/StreamingModule/* INTO $upstream"
}
    
```

### Testing

Using the simulator is generally fine for most cases but in this case and with the routing to send ASA messages to the cloud, you may get some timeout errors . To be on the safe side, let's try it on the real \(virtual 😅\) device.

![](../.gitbook/assets/image%20%2826%29.png)

To be on the safe side, let's try it on the real \(virtual 😅\) device.

{% hint style="info" %}
If there is an active deployment on the device from a prior exercise like CI/CD, better to remove that deployment using Azure CLI.
{% endhint %}

![](../.gitbook/assets/image%20%2855%29.png)

```text
iot edge deployment list --hub-name master-hub --output tsv
iot edge deployment delete --hub-name master-hub --deployment-id edgeops-dev-release-8
```

Make sure edge device VM is up and running:

* Build the solution from deployment template file context menu.
* Deploy the solution to simulated edge device.

The device now has simulated temperature sensor plus ASA module.

![](../.gitbook/assets/image%20%2890%29.png)

You can see that once average machine temperature goes above 30 for the last 10 seconds, ASA will output a non empty result which translates to a message on edgeHub which is sent to sensor module and resets it.

![](../.gitbook/assets/image%20%28184%29.png)

And ASA module logs can be inspected for more details about operational statistics.

`iotedge logs -f StreamingModule` 

![](../.gitbook/assets/image%20%2851%29.png)

Routing configuration ensures reset message is configured to be sent to the cloud as well so the system has a log of such events. Such messages could be inspected in VS Code by monitoring cloud to device messages.

![](../.gitbook/assets/image%20%2870%29.png)

{% hint style="info" %}
If ASA job query is updated, the job has to be republished in the portal and then in VS Code there is an option to refresh it.
{% endhint %}

![](../.gitbook/assets/image%20%2897%29.png)

![](../.gitbook/assets/image%20%28130%29.png)

