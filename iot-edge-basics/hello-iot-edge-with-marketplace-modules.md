# Hello IoT Edge

## Introducing IoT Edge

![https://unsplash.com/photos/67L18R4tW\_w](../.gitbook/assets/image%20%2871%29.png)

We have barely scratched the surface of IoT hub and it looks cool so why do we need another thing called IoT Edge. There are many scenarios where plain IoT hub features are not enough or optimal to solve the business problem, for example:

* When a fast response is required and the latency for an API call to the cloud is not acceptable
* When IoT devices cannot connect directly to the cloud for security or networking reasons
* When the network connection to the cloud is not guaranteed to be up 100% of the time and offline functionality is key
* When it's very costly to blindly push all telemetry collected to the cloud
* When it's much better to run ML at the device itself for latency and reliability reasons

The following diagram shows one of the use cases where the edge device acts as a getaway for other downstream IoT devices.

![](../.gitbook/assets/image%20%28133%29.png)

### So what is Azure IoT Edge? <a id="decision-criteria"></a>

Azure IoT Edge is a cloud service running in Azure plus a runtime that runs on an edge device. The runtime manages solution code running on the device which is deployed as one or more docker containers. An IoT Edge solution is a group of docker containers that can work independently or communicate with each other or with IoT hub with the help of edge runtime. Each container is called a module which may be a 3rd party module or a custom module developed for a specific use case.

IoT Edge enables you to:

* Code runs on the device directly so response to any environment changes is almost instant
* Manage a fleet of devices from a single place.
* Deploy solutions using docker containers which when done right can speed up and simplify development.
* AI/ML models can run on the device directly for faster response and less dependency on the cloud
* Lowe barrier to module development which is done using languages such as C, C\#, Java, Node.js, and Python. 
* Telemetry can be captured and analysed on the device directly. Only useful information can be send to the cloud to reduce storage and ingestion costs.
* Modules can run offline which is very critical for cases with intermittent or offline connectivity to the cloud.
* Security is handled by the runtime using X.509 certificates and hardware modules can be used for even higher levels of security.
* IoT devices can act as a gateway for devices which don't have network connection to the cloud or don't support network protocols to communicate with the cloud.
* Large ecosystem of partners and open source developers publish modules that can be used out of the box.

### Create and configure an edge device

Edge devices are simply more powerful IoT devices and they could be even full blown server machines. The spectrum starts actually from devices like Raspberry Pi till Ubuntu or Windows servers. The main requirement on those devices is the capability to host a container engine. Microsoft officially supports [Moby ](https://mobyproject.org/)but docker works fine as well and for the purpose of this exercise they are equally fine.

To kick-off our edge journey, let's register a device identity with IoT hub first.

`iot hub device-identity create --hub-name master-hub --device-id edge-device --edge-enabled true` 

The main difference now when creating edge-enabled devices is the extra parameter called **edge-enabled.** This instructs IoT hub to register a device with edge functionality that we will see later.

![](../.gitbook/assets/image%20%2865%29.png)

Now IoT hub view in VS code will show the new device created with a different icon indicating it's an edge device.

![](../.gitbook/assets/image%20%28212%29.png)

Device connection string is required to wire up the edge device with IoT hub later. So run the following command and take a note of the connection string. IoT hub devices uses SDK and device connection string is used to authenticate with IoT hub. Edge devices follow a similar approach but this authentication or registration thing is done once globally at the device level and we will see that shortly.

`az iot hub device-identity show-connection-string --hub-name master-hub --device-id edge-device` 

The next step is to create an Azure VM running the edge device. Normally an edge device could be a server/device in a factory or something like that but developing against a VM is acceptable as well. 

`vm create --resource-group iotedge-rg --image UbuntuLTS --size Standard_A4_v2 --authentication-type password --admin-username edgeadmin --admin-password EdgeAdmin!@# --name yousry-edgevm` 

The VM name above better to be tweaked just in case there will be global collision as it will be also used as a DNS name label. Also the user names and password are fine to be changed or you can even use SSH keys if you like. Once created, we will have CLI output showing public IP address we can use to SSH into the VM.

![](../.gitbook/assets/image%20%282%29.png)

{% hint style="info" %}
In this exercise, we will see how to install IoT edge runtime on the virtual machine from scratch. Microsoft provides an Ubuntu VM on Azure with the runtime pre-installed pending to connect it with the device only.
{% endhint %}

![](../.gitbook/assets/image%20%28151%29.png)

Now let's SSH into the new VM in another shell window. SSH port will be probably open as this is the default outcome with the VM creation command above but things may change and if port 22 is not open, it can be opened from the portal or CLI. We have an Ubuntu 18 server with 4 cores and 8GB of memory.

![](../.gitbook/assets/image%20%28180%29.png)

IoT edge is a runtime on Windows/Linux that works as the glue that connects running containers on the device and wiring them to IoT hub plus handle security as well.

We will follow the [instructions ](https://docs.microsoft.com/en-us/azure/iot-edge/how-to-install-iot-edge-linux)from Microsoft for installing IoT edge on a Linux machine but I will distill it here for Ubuntu case only. To prepare for IoT edge installation, run the following to register Microsoft package repositories.

```bash
curl https://packages.microsoft.com/config/ubuntu/18.04/multiarch/prod.list > ./microsoft-prod.list
sudo cp ./microsoft-prod.list /etc/apt/sources.list.d/
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg
sudo cp ./microsoft.gpg /etc/apt/trusted.gpg.d/
```

Next we need to install Moby \(or docker if you want\).

```bash
sudo apt-get update
sudo apt-get --yes install moby-engine
sudo apt-get --yes install moby-cli
```

After that, run the following to install IoT edge runtime which is sometimes also called security daemon.

```text
sudo apt-get update
sudo apt-get --yes install iotedge
```

The output of the above commands contains some hints about how to register IoT hub device identity on current machine.

![](../.gitbook/assets/image%20%285%29.png)

For production deployments, TPM modules and certificates are the reliable security options available but for development it's enough to use shared keys and device connection strings.

Edit `/etc/iotedge/config.yaml` file and update connection accordingly.

`sudo nano /etc/iotedge/config.yaml` 

![](../.gitbook/assets/image%20%2850%29.png)

Next restart IoT edge on the VM.

`sudo systemctl restart iotedge` 

Now, you can check the status of IoT edge service.

`sudo systemctl status iotedge` 

![](../.gitbook/assets/image%20%28107%29.png)

You can also run the below command for more detailed analysis of IoT edge configuration.

`sudo iotedge check` 

![](../.gitbook/assets/image%20%2853%29.png)

There are some warnings and errors here and they are related to production readiness. The most important one is about using self-signed development X.509 certificates. For more details on why it's important to use real certificates, check out the documentation of [production readiness](https://docs.microsoft.com/en-us/azure/iot-edge/production-checklist#install-production-certificates).

Finally to inspect what modules are running on the device, run this command.

`sudo iotedge list` 

![](../.gitbook/assets/image%20%2860%29.png)

Currently, edgeAgent is the only module running but later once we deploy our solutions to the device we will find other own modules plus another system module which is edgeHub.

Run the following to have a look behind the scenes.

`sudo docker ps`

![](../.gitbook/assets/image%20%2886%29.png)

To have a look on our device in Azure portal, click **IoT Edge** link inside IoT hub side menu to view the list of edge devices we have. The device will appear there as expected but it has runtime response column of 417 which means there is no deployment configuration set. Runtime responses are very similar to HTTP responses so hopefully once a valid deployment configuration is set on the device, that status will change to 200.

![](../.gitbook/assets/image%20%286%29.png)

Next, let's move on and deploy some modules from marketplace to have this edge device doing something useful.

