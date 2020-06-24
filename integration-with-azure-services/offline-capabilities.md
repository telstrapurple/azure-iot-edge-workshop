# Offline Capabilities

![https://unsplash.com/photos/QOkr2RY4DT4](../.gitbook/assets/image%20%2888%29.png)

### Introduction

Well, you may ask why an offline capabilities is part of integration with Azure services section. You are right, it's not strictly related but easy to try with Azure Stream Analytics example done in the previous page.

One of the use cases or advantages of Azure IoT Edge is to allow the device to function with intermittent network connection or maybe with no network connection altogether. Complete offline functionality is not something to be done for very long periods of times. In all cases, the device should be inspected for being healthy and running. But still, it is very critical for many cases to have the system doing it's core function without the need of a live connection. Think about your smoke alarm, it can sense smoke and trigger the alarm if needed. This is what we are trying to achieve in this scenario.

You recall the ASA example with temperature sensor being reset if temperature goes above 30. The reset command is also submitted to the cloud to be stored at IoT hub for later analysis or archival purposes. This is a clear example of IoT Edge design goals. It needs to handle environment changes at the device side instantly and certain bits of data could be sent to the cloud as well if needed.

But would would happen if the network goes down for a while in this specific case. Let's have a look.

### Extend ASA job query

Let's start with the final outcome from [ASA page](azure-stream-analytics.md). To help with understanding and verifying things later, it's better to enrich the ASA query with a timestamp field on the output message showing the time of the last event processed within the relevant time window. **MaxEventTime** is the new column that would show the time of the last event of the window processed.

```sql
SELECT 'reset' AS command, Max(timeCreated) as maxEventTime
INTO output
FROM input 
TIMESTAMP BY timeCreated
GROUP BY TumblingWindow(second, 10)
HAVING Avg(machine.temperature) > 30
```

 Save the query and republish the job from the portal. In deployment template file, click the link named **Check configuration update for ASA job** and select yes to update the job. You can also tweak **SendInterval** of simulated temperature sensor to be 1 second for quicker results if you want. Build and deploy the solution to edge device.

You can start fresh on edge device in case temperature module reached its 500 message limit.

`sudo systemctl restart iotedge` 

Monitor the logs of temperature sensor module.

`iotedge logs -f --tail 20 SimulatedTemperatureSensor` 

![](../.gitbook/assets/image%20%28150%29.png)

Cool, the reset message has an extra element showing UTC time of last event in window\(batch\). You can also monitor device to cloud messages in VS Code. The obvious thing to notice is that the **local GMT +10** timestamp from VS Code \(nearly\) matches the UTC timestamp from ASA message.

![](../.gitbook/assets/image%20%28103%29.png)

### Simulating network connection dropped

EdgeHub module on the device communicates with IoT Hub over [AMQP ](https://en.wikipedia.org/wiki/Advanced_Message_Queuing_Protocol)or [MQTT](https://en.wikipedia.org/wiki/MQTT). When working with real edge devices, we could unplug network cable or turn off WiFi. Because we use a simulated edge device in an Azure VM, we would follow a different approach which is a bit convoluted.

1. Use Azure CLI to deny outbound network activity on AMQP port \(default upstream protocol\)
2. Restart edgeHub module. This is necessary to force edgeHub to create upstream connection from scratch and fail initially. It seems that blocking the port which edgeHub has an already established connection has no impact on connection functionality.
3. Wait for a while until a bunch of reset messages accumulate and are supposed to be sent to the cloud
4. Use Azure CLI to remove that deny port rule. EdgeHub will sense the connection can be established and send the pending messages to the cloud.
5. Verify many messages are sent almost at once.

{% hint style="info" %}
Please note that due to disk space limits on the device, there is a TTL \(time to live\) after which edgeHub will start dropping messages specially if the rate of message generation is too fast. So depending on the expected rate and disk space on the device, TTL setting could be adjusted.
{% endhint %}

### Running the simulation

To start fresh, restart IoT Edge runtime on the device and wait until ASA starts sending its reset messages.

`sudo systemctl restart iotedge` 

Let's first stop AMQP port, resource group and network security group names should be adjusted to match settings on your installation.

`network nsg rule create -g iotedge-rg --nsg-name yousry-edgevmNSG --name denyAMQP --priority 100 --access Deny --destination-port-ranges 5671 --direction Outbound --protocol *`

Restart edgeHub module on the device.

`iotedge restart edgeHub` 

Monitor temperature module and confirm reset messages are still sent and temperature is adjusted accordingly.

`iotedge logs -f --tail 20 SimulatedTemperatureSensor` 

If you are monitoring cloud to device messages using VS Code, you will notice nothing is sent since the time AMQP port was blocked. Now enable the port once again by deleting the rule created before.

`network nsg rule delete -g iotedge-rg --nsg-name yousry-edgevmNSG --name denyAMQP` 

Inspect logs of edgeHub and confirm there was some exceptions when the port was blocked, connection re-establishment and processing pending messages.

`iotedge logs --tail 80 -f edgeHub` 

![](../.gitbook/assets/image%20%28132%29.png)

From VS Code C2D monitoring, you can spot a small flood of messages after the last one before blocking the port. Those messages were delivered almost the same time to the cloud while their event timestamps show prior times.

![](../.gitbook/assets/image%20%28186%29.png)

In the above screenshot:

* Blue box is for the last message delivered to IoT hub before losing connection.
* Red box is for all the messages delivered after connection was re-established. It's obvious VS Code timestamp \(which is when the message is delivered to IoT hub\) is a single second interval while the message payloads show different values of **maxEventTime**.
* Green box is back to normal operation when VS Code timestamp and message timestamp are nearly in sync.

### Conclusion

It's really good that offline capability is provided with virtually no extra effort needed by developers. Modules can still communicate with each other and handle any environment changes as required by their implementation. Messages to be sent to the cloud are submitted once the connection is available again. This design pattern handles the needs to be responsive on the device even in offline cases and also send any pending data to be synced to the cloud once the device is back online. That's pretty cool.



