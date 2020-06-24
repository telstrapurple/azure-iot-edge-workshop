# Develop custom modules

![https://unsplash.com/photos/1LWXx9kwlLw](../.gitbook/assets/image%20%28193%29.png)

The sample implemented in [previous page](deploy-marketplace-modules.md) was mainly based on deploying a ready-made module to an edge device. Let's say the client is happy with the temperature telemetry pushed to the cloud but wants to extend the solution with a couple new features:

* Submit temperature telemetry above a certain threshold only
* Increase the frequency of collecting/inspecting telemetry on the device from 5 seconds to 1 second

Now it's time to implement an IoT edge solution to handle those new requirements. As this solution will include a custom module, we need to have a container registry to host docker image for that custom module. The two obvious options are docker hub or Azure container registry \(ACR\). I will pick ACR for this exercise. 

To create a container registry run the below command. You need to change the registry name to make it globally unique.

`acr create --resource-group iotedge-rg --admin-enabled true --sku Standard --name yousrycr` 

![](../.gitbook/assets/image%20%28200%29.png)

If the above command fails or takes more than usual to run, you can ask Azure to explicitly use container registry in your subscription.

`az provider register --namespace 'Microsoft.ContainerRegistry'` 

Take a note of registry credentials as we will need them shortly.

`acr credential show --name yousrycr` 

![](../.gitbook/assets/image%20%2822%29.png)

Login details for ACR would look like:

* **Login server:** \[your-registry-name\].azurecr.io
* **User name:** \[your-registry-name\]
* **Password:** Password obtained from latest command above

### Create and configure IoT edge solution

Open VS Code and using command palette \(Ctrl+Shift+P\) select **New IoT Edge Solution.** Pick an empty folder to hold the code for this solution and give the solution a name.

There are many options to scaffold the solution and in our case, pick **C\# module** then give a name to the module like **TemperatureProcessingModule**. 

![](../.gitbook/assets/image%20%2881%29.png)

The next step will be a question about where to host docker image of this custom module. This time replace **localhost:5000** with ACR registry name created before.

![](../.gitbook/assets/image%20%2825%29.png)

VS Code explorer view should look like this.

![](../.gitbook/assets/image%20%28115%29.png)

As it's very critical to understand the components of an IoT edge solution, the following files are worth some description:

* `deployment.template.json` has information about what modules involved in the solution and how they interact with each other and where they are hosted.
* `.gitignore` is the classic git ignore file but here it's mainly to skip config folder \(will show up later\) and `.env` file.
* `.env` file holds credentials of container registry used to hold modules image\(s\) hence it should be ignored.
* `modules` folder contains a folder per each custom module in the solution.
* Any module folder should have the following files in addition to source code files for the module itself:
  * `module.json` defines how to package and build docker image for this module.
  * A bunch of docker files  with instructions to build module image for different configurations

First make sure `.env` file has credentials for your ACR instance. If not fill them manually such that the file should look like this.

```text

CONTAINER_REGISTRY_USERNAME_yousrycr=YOUR-ACR-NAME-WITHOUT-azurecr.io
CONTAINER_REGISTRY_PASSWORD_yousrycr=YOUR-SUPER-SECRET-PASSWORD

```

{% hint style="info" %}
To make things easy for CI/CD later, better to remove the lower case part of environment variables or make them upper case. Release pipeline in Azure DevOps changes variable names to [uppercase](https://github.com/microsoft/azure-pipelines-agent/issues/1645). If you do this change, replicate the same change in other solution files \(Replace in files\).
{% endhint %}

Sometimes you may need also to log in to this ACR registry in VS Code terminal window.

`docker login --username [your-acr-name] --password YOUR-PASSWORD [your-acr-name].azurecr.io` 

### Dissecting Deployment Template File

Deployment template file is the core to any IoT edge solution as it defines how a full deployment of IoT edge modules should be deployed to a device.

Any IoT edge solution includes two system modules which are edgeAgent and edgeHub.

![](../.gitbook/assets/image%20%2893%29.png)

edgeAgent has information about the container runtime used in this deployment, system modules involved \(usually edgeAgent & edgeHub\) and any other modules involved.

![](../.gitbook/assets/image%20%2810%29.png)

**runtime** section included information about container registry and its credentials in case it's not a public registry. Credentials are populated dynamically using `.env` file.

```javascript
"runtime": {
          "type": "docker",
          "settings": {
            "minDockerVersion": "v1.25",
            "loggingOptions": "",
            "registryCredentials": {
              "yousrycr": {
                "username": "$CONTAINER_REGISTRY_USERNAME_yousrycr",
                "password": "$CONTAINER_REGISTRY_PASSWORD_yousrycr",
                "address": "yousrycr.azurecr.io"
              }
            }
          }
        }
```

**systemModules** section list what are the images used for edgeAgent & edgeHub containers and what ports to open on edgeHub container. Those ports are mainly for AMQP/MQTT/HTTPS communications as those are the 3 main protocols edgeHub uses to talk to IoT hub in the cloud. 

{% hint style="info" %}
Use the version part of system module image names like to control which specific image to pull from registry. There is more info in [Update the IoT Edge security daemon and runtime](https://docs.microsoft.com/en-us/azure/iot-edge/how-to-update-iot-edge#understand-iot-edge-tags) docs page. 
{% endhint %}

```javascript
"systemModules": {
          "edgeAgent": {
            "type": "docker",
            "settings": {
              "image": "mcr.microsoft.com/azureiotedge-agent:1.0",
              "createOptions": {}
            }
          },
          "edgeHub": {
            "type": "docker",
            "status": "running",
            "restartPolicy": "always",
            "settings": {
              "image": "mcr.microsoft.com/azureiotedge-hub:1.0",
              "createOptions": {
                "HostConfig": {
                  "PortBindings": {
                    "5671/tcp": [
                      {
                        "HostPort": "5671"
                      }
                    ],
                    "8883/tcp": [
                      {
                        "HostPort": "8883"
                      }
                    ],
                    "443/tcp": [
                      {
                        "HostPort": "443"
                      }
                    ]
                  }
                }
              }
            }
          }
        }
```

**modules** element has the details about what custom modules are involved in this deployment. A module could simply point to an image developed by Microsoft or a 3rd party similar to simulated temperature sensor module. A module could be also a custom module with source code in the same solution like our **TemperatureProcessingModule** module.  When the solution is built, expressions like `${MODULES.TemperatureProcessingModule}` are replaced with stuff like `yousrycr.azurecr.io/temperatureprocessingmodule:0.0.1-amd64` .

```javascript
"modules": {
          "TemperatureProcessingModule": {
            "version": "1.0",
            "type": "docker",
            "status": "running",
            "restartPolicy": "always",
            "settings": {
              "image": "${MODULES.TemperatureProcessingModule}",
              "createOptions": {}
            }
          },
          "SimulatedTemperatureSensor": {
            "version": "1.0",
            "type": "docker",
            "status": "running",
            "restartPolicy": "always",
            "settings": {
              "image": "mcr.microsoft.com/azureiotedge-simulated-temperature-sensor:1.0",
              "createOptions": {}
            }
          }
        }
```

Final section of the template is **$edgeHube** element which defines how modules communicate messages among them and IoT hub in the cloud. In our case, simulated temperature sensor module sends messages on an output named **temperatureOutput** and those messages are collected by the processing module on an input named **input1.** Also any messages generated by the processing module will be pushed to IoT hub in the cloud. edgeHub module plays the broker role to achieve those tasks.

```javascript
"$edgeHub": {
      "properties.desired": {
        "schemaVersion": "1.0",
        "routes": {
          "TemperatureProcessingModuleToIoTHub": "FROM /messages/modules/TemperatureProcessingModule/outputs/* INTO $upstream",
          "sensorToTemperatureProcessingModule": "FROM /messages/modules/SimulatedTemperatureSensor/outputs/temperatureOutput INTO BrokeredEndpoint(\"/modules/TemperatureProcessingModule/inputs/input1\")"
        },
        "storeAndForwardConfiguration": {
          "timeToLiveSecs": 7200
        }
      }
    }
```

As we can see, the scaffolded solution includes a Microsoft ready-made modules and another custom modules we will have a look on later.

### Building the solution

As IoT edge solutions can be deployed on several hardware types, we should define target hardware first before building the solution. From command palette, select **Select Azure IoT Edge Solution Default Platform** and then pick **amd64** which is mainly Linux containers on Linux. The ARM options are for ARM hardware like Raspberry Pi or NVIDIA Jetson Nano. The **windows amd64** is for Windows containers on Windows. For more details have a look on [IoT Edge supported systems](https://docs.microsoft.com/en-us/azure/iot-edge/support).

![](../.gitbook/assets/image%20%2896%29.png)

Next, right click deployment template file and select **Build and Push IoT Edge Solution.**

![](../.gitbook/assets/image%20%28109%29.png)

This will trigger a few actions:

* Building docker images for any custom modules
* This may involve pulling other docker images involved in build/compilation process. [Multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/) are very common in this space.
* Push the final built image\(s\) to the designated container registry
* Create a config folder \(here it comes why it's git ignored\) containing a modified copy of deployment template file that has concrete values for things like module images and registry credential. This generated file is the one that can be used to push to the device to apply the required deployment configuration

Run `docker images` in a new shell window to confirm if things worked fine. You should see the custom module build image and any other supporting images involved in the build process.

![](../.gitbook/assets/image%20%28146%29.png)

In Azure CLI or portal, you can confirm that the custom module image has been pushed successfully.

`acr repository list --name yousrycr` 

![](../.gitbook/assets/image%20%2849%29.png)

### Deploy the solution to the device

Now it's time to deploy this solution to the device and test how it works. We haven't seen the code of the custom module yet but it's simply echoing the messages it receives from temperature sensor module. So we will have a look on it shortly at least to change it to satisfy the client's requirements needed.

Pushing a deployment configuration to a device can be done from the portal as we have seen before or can be done from VS Code for development purposes or Azure DevOps for CI/CD scenarios which we will see later.

Now right click on the edge device in VS Code and select **Create Deployment fro Single Device.** 

![](../.gitbook/assets/image%20%28213%29.png)

A file selection dialog will pop up, select the deployment file in config folder.

![](../.gitbook/assets/image%20%288%29.png)

Wait a few seconds and output window will show deployment status.

![](../.gitbook/assets/image%20%2898%29.png)

SSH into the edge device and run `sudo iotedge list` . The simulated temperature sensor module along with the custom processing module are both deployed and running successfully.

![](../.gitbook/assets/image%20%28177%29.png)

While that looks promising, there is a gotcha here. if you inspect the logs of both modules, you **may** see something like the following.

![](../.gitbook/assets/image%20%28124%29.png)

The processing module has been initialized but it does not seem to be doing anything. Temperature sensor declares that it's done sending 500 messages. So the gotcha here is basically:

*  Simulated sensor module has been deployed already the device before and was running fine.
* Any new deployment with same module without any change in how it's configured will not result in restarting the module.
* That module has a max of 500 messages to send and they have been completely sent long time before we deploy the our customer solution configuration.
* Hence the processor module has not received anything to process and was just waiting for input.

To start fresh, a specific module can be restarted or even the whole IoT edge runtime.

```bash
sudo iotedge restart SimulatedTemperatureSensor
sudo systemctl restart iotedge 
```

### Develop custom logic

If you open `program.cs`

Add a file named `Temperature.cs`

### Tweak configuration of temperature sensor module

Before playing with C\# code, the requirement of inspecting temperature reading every 1 seconds instead of 5 seconds can be easily done. Module element within deployment template file can be tweaked with something called desired properties to control its functionality from the backend. Desired properties can be tweaked for the case of starting the module with certain config or changing the config at runtime by backend application if needed. Desired properties are part of a larger topic called [device twins](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-device-twins) and the same concept applies to modules as well. Let's see how to change temperature sensor module twin's desired properties.

If any non-system module needs some custom configuration, it can be added right after $edgeHub element and VS Code has auto complete to help although the individual custom properties of each module need to be known from its documentation or source code.

![](../.gitbook/assets/image%20%28121%29.png)

Add the following snippet after $edgeHub element. The main change is controlling the send interval of temperature sensor module to be 1 seconds instead of the default value of 5 seconds.

```javascript
,
"SimulatedTemperatureSensor": {
  "properties.desired": {
    "SendData": true,
    "SendInterval": 1
  }
}
```

Because there is no change in any module docker image, we can simply right click `deployment.template.json` and select **Generate IoT Edge Deployment Manifest.** After that deploy that configuration to the device as before and inspect the running containers and logs. And it's clear that sensor module generates telemetry messages every second as configured. Also the processing module has picked messages and done something with them.

![](../.gitbook/assets/image%20%28206%29.png)

### Implement custom logic 

`Program.cs` file has the core functionality of the C\# custom module. It does some housekeeping of connecting to edgeHub using a class called `ModuleClient`. It also wires up a method to receive data on an input named **input1.** The method simply echos back the received message to edgeHub. What we are going to to is to skip echoing the message if the temperature is below a certain threshold.

First add a file named `Temperature.cs` to the C\# project folder and fill it with the below content.

```csharp
using System;
using Newtonsoft.Json;

namespace TemperatureProcessingModule
{
    public partial class MessagePayload
    {
        [JsonProperty("machine")]
        public Machine Machine { get; set; }

        [JsonProperty("ambient")]
        public Ambient Ambient { get; set; }

        [JsonProperty("timeCreated")]
        public DateTimeOffset TimeCreated { get; set; }
    }

    public partial class Ambient
    {
        [JsonProperty("temperature")]
        public double Temperature { get; set; }

        [JsonProperty("humidity")]
        public long Humidity { get; set; }
    }

    public partial class Machine
    {
        [JsonProperty("temperature")]
        public double Temperature { get; set; }

        [JsonProperty("pressure")]
        public double Pressure { get; set; }
    }
}
```

Switch back to `Program.cs` and add the below snippet after the line `string messageString = Encoding.UTF8.GetString(messageBytes);` 

```csharp
var temperatureThreshold = 30;
var payload = JsonConvert.DeserializeObject<MessagePayload>(messageString);
if (payload.Ambient.Temperature < temperatureThreshold)
{
    System.Console.WriteLine($"Ambient temperature {payload.Ambient.Temperature} below threshold of {temperatureThreshold}! No need to submit data to IoT hub.");
    return MessageResponse.Completed;
}
```

Build and push IoT edge solution and then deploy it to the device. If you inspect logs of the custom module, you will see that the new logic added above executes as expected and no messages are pushed to the cloud unless ambient temperature is greater than or equal threshold.

Let's have a look next on how to develop and debug locally without the need to deploy code to the device.



