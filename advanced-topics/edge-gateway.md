# Edge Gateway

![https://unsplash.com/photos/fPxOowbR6ls](../.gitbook/assets/image%20%28165%29.png)

One of the major use cases of Azure IoT Edge is to act as a gateway for downstream \(leaf\) devices. Sometimes those devices cannot communicate directly with IoT hub. Security requirements in certain environment prevents IoT devices to access the cloud directly. All traffic should be sent from a single machine with strict security controls. In all such cases, an IoT Edge device can act as a gateway to tunnel communication from leaf devices to the cloud. IoT Edge device acts also as a transient IoT hub so that when connection to the cloud is down, devices can still function normally and send telemetry to IoT Edge device. Once connection is restored, data will be sent to the cloud as expected.

### Setup edge device to forward data to cloud

Let's start with our `edge-device` in Azure portal.

* Open the device details page in the portal.
* Click **Set Modules**
* Remove any existing modules \(which will leave the device with edgeAgent and edgeHub\)
* Click **Next: Routes**
* Add a new route named `allToCloud` with a value of `FROM /* INTO $upstream` 
* Proceed with the next confirmation steps and submit the deployment

This deployment simply makes the edge device a transparent router that forwards any messages generated on the device or received from leaf devices to IoT hub in the cloud.

SSH into the device VM and make sure the system modules are the only ones running on the device.

![](../.gitbook/assets/image%20%28142%29.png)

### Generate IoT Edge device CA certificates

Leaf devices will communicate with IoT Edge device only. Edge device will act as if it's the IoT hub for them. To allow those devices to communicate with edge device, they have to use certificates. The workflow is simple. A CA certificate is issued and installed on edge device. Every leaf device will need to have a child certificate with the same CA certificate as it's parent.

For development purposes, IoT Edge repo has some scripts to create those certificates.

* SSH into the device VM
* Clone IoT Edge repo

```text
git clone https://github.com/Azure/iotedge.git
```

* Create a folder to hold generated certificates

```text
 mkdir certificates
 cd certificates
```

* Copy needed files to this folder

```text
cp ../iotedge/tools/CACertificates/*.cnf .
cp ../iotedge/tools/CACertificates/certGen.sh .
```

* Run the shell script from IoT Edge repo to create a root and intermediate certitifcate

```text
./certGen.sh create_root_and_intermediate
```

* If everything works fine, certificates will be created and there will be a warning that those certificates are for non production use.

![](../.gitbook/assets/image%20%2835%29.png)

* Root certificate should be located at `~/certificates/certs/azure-iot-test-only.root.ca.cert.pem` 
* Run the following to create edge device CA certificate

```text
./certGen.sh create_edge_device_ca_certificate "EdgeDeviceCA"
```

![](../.gitbook/assets/image%20%28156%29.png)

### Configure IoT Edge device to use certificates

Now it's time to configure IoT Edge to use the CA certificate created above. You will edit IoT Edge config file and point it to the CA certificate.

* Open config file `sudo nano /etc/iotedge/config.yaml` 
* Un-comment the certificates element and set it as follows \(adjust user name if needed if you have a name other than **edgeadmin**\)

```yaml
certificates:
  device_ca_cert: "/home/edgeadmin/certificates/certs/iot-edge-device-ca-EdgeDeviceCA-full-chain.cert.pem"
  device_ca_pk: "/home/edgeadmin/certificates/private/iot-edge-device-ca-EdgeDeviceCA.key.pem"
  trusted_ca_certs: "/home/edgeadmin/certificates/certs/azure-iot-test-only.root.ca.cert.pem"
```

* Save the file and exit nano
* Restart IoT Edge service

```text
sudo systemctl restart iotedge
```

* Run `sudo iotedge check` and make sure certificate line shows an OK

![](../.gitbook/assets/image%20%28157%29.png)

### Open ports on IoT Edge device

Leaf devices connect to IoT Edge as if they are connecting to a real IoT Hub. the following protocols are supported:

<table>
  <thead>
    <tr>
      <th style="text-align:left">Protocol</th>
      <th style="text-align:left">Port to open</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align:left">MQTT</td>
      <td style="text-align:left">8883</td>
    </tr>
    <tr>
      <td style="text-align:left">AMQP</td>
      <td style="text-align:left">5671</td>
    </tr>
    <tr>
      <td style="text-align:left">
        <p>HTTPS</p>
        <p>MQTT + WS</p>
        <p>AMQP + WS</p>
      </td>
      <td style="text-align:left">443</td>
    </tr>
  </tbody>
</table>

As our edge device is an Azure VM, we need to allow the inbound port of the protocol we will use which is **AMQP** in our case. You might need to adjust the network security group name for your own VM.

```text
network nsg rule create -g iotedge-rg --nsg-name yousry-edgevmNSG --name AllowAMQP --priority 100 --access Allow --destination-port-ranges 5671 --direction Inbound --protocol *
```

You will need also to double check that Ubuntu firewall is disabled on edge VM just in case.

![](../.gitbook/assets/image%20%287%29.png)

### Create IoT Edge device host name

IoT Edge device acting as a gateway needs to be accessible using a host name as the device is an Azure VM. For more classic scenarios where edge device and leaf devices are hosted on same network, then we probably wouldn't need this step.

Go to edge device VM in portal and click the public IP link.

![](../.gitbook/assets/image%20%28144%29.png)

Assign and DNS name label and save it.  DNS name label has to match the virtual machine name hence the VM name has to be unique somehow initially.

![](../.gitbook/assets/image%20%28148%29.png)

Once assigned the edge device can be addressed as `yousry-edgevm.southeastasia.cloudapp.azure.com` .  You may need to wait a few minutes until DNS records get sync-ed. I tried to restart the VM and change IP assignment from dynamic to static but I guess it was just a matter of waiting depending on what DNS servers your machine uses.

![](../.gitbook/assets/image%20%28175%29.png)

Anyway, SSH into the VM using the IP or DNS name and edit config file.

```text
sudo nano /etc/iotedge/config.yaml
```

Change **hostname** entry to be DNS name of the VM \(`yousry-edgevm.southeastasia.cloudapp.azure.com`\) . IoT Edge allow the host name to be only one of two options:

* Plain machine name
* A FQDN where the first portion is the machine name

We are using the second option here because it's the option that allows proper TLS communication between leaf devices and edge device.

Save the file and restart and check status if IoT Edge

```text
sudo systemctl restart iotedge
sudo iotedge check
```

### Create a few leaf devices

Use Azure CLI to create a few leaf devices and configure the edge device as their parent.

```bash
iot hub device-identity create --hub-name master-hub --auth-method shared_private_key --set-parent edge-device --device-id LeafDevice1
iot hub device-identity create --hub-name master-hub --auth-method shared_private_key --set-parent edge-device --device-id LeafDevice2
iot hub device-identity create --hub-name master-hub --auth-method shared_private_key --set-parent edge-device --device-id LeafDevice3
iot hub device-identity create --hub-name master-hub --auth-method shared_private_key --set-parent edge-device --device-id LeafDevice4
```

Open IoT Hub in portal and select **IoT Devices** from **Explorer** section, then click one of the leaf devices created.

![](../.gitbook/assets/image%20%2846%29.png)

Make sure parent device correctly linked to the edge device.

Devices can be also inspected in VS Code.

![](../.gitbook/assets/image%20%28190%29.png)

Because we will need the connection strings for all leaf devices, run the following snippet to show connection strings for all devices and take a note of them. Repeat for all devices involved.

```text
iot hub device-identity show-connection-string --hub-name master-hub --device-id LeafDevice1
```

### Send telemetry from leaf devices

Next step is to simulate sending telemetry from leaf devices to the cloud via edge device \(the gateway\). A simple dotnet core console app will be used and 4 instances of it will run in docker containers. This is not a requirement but just an easy way to simulate multiple edge devices running in parallel.

* On your development machine, create a new folder and bootstrap a dotnet core console app and open it in VS Code.

```text
mkdir LeafDeviceApp
cd LeafDeviceApp
dotnet new console
code .
```

* In VS terminal window, add IoT Hub device client nuget 

```text
dotnet add package Microsoft.Azure.Devices.Client
```

* Change main method as follows. It will first call a helper method to install root CA certificate and then can another method for continuous generation of sample telemetry messages. Two environment variables are needed. One for device connection string and another one to label messages generated from current device. IoT hub will correctly identify source device but it's nice also to tag those messages with another label.

```csharp
static async Task Main(string[] args)
{

    InstallCACert();

    var connectionString = System.Environment.GetEnvironmentVariable("CONNECTION_STRING");
    var deviceLabel = System.Environment.GetEnvironmentVariable("DEVICE_LABEL");

    var deviceClient = DeviceClient.CreateFromConnectionString(connectionString, TransportType.Amqp);
    System.Console.WriteLine("Created device client instance.");
    await SendDeviceToCloudMessagesAsync(deviceClient, deviceLabel);
    Console.ReadLine();
}
```

* Add method to install CA certificate. It simply boilerplate code.

```csharp
static void InstallCACert()
{
    string trustedCACertPath = "azure-iot-test-only.root.ca.cert.pem";
    Console.WriteLine($"Attempting to install CA certificate: {trustedCACertPath}");
    using(var store = new X509Store(StoreName.Root, StoreLocation.CurrentUser))
    {
        store.Open(OpenFlags.ReadWrite);
        store.Add(new X509Certificate2(X509Certificate.CreateFromCertFile(trustedCACertPath)));
        Console.WriteLine($"Successfully added certificate: {trustedCACertPath}");
    }
}
```

* Add method to generate sample telemetry messages and send them to IoT hub \(via the gateway\)

```csharp
private static async Task SendDeviceToCloudMessagesAsync(DeviceClient deviceClient, string deviceLabel)
{
    System.Console.WriteLine("Sending telemetry messages to the cloud...");
    while (true)
    {
        Random rand = new Random();
        var telemetry = new
        {
            temperature = rand.NextDouble() * 40,
            humidity = rand.NextDouble() * 80,
            device = deviceLabel,
            timestamp = System.DateTime.UtcNow
        };

        var messageString = JsonConvert.SerializeObject(telemetry);
        var message = new Message(Encoding.UTF8.GetBytes(messageString));

        await deviceClient.SendEventAsync(message);
        Console.WriteLine($"{DateTime.Now} > Sending message: {messageString}");

        await Task.Delay(1000);
    }
}
```

* Add a `Dockerfile` to build an image for the leaf device

```text
FROM mcr.microsoft.com/dotnet/core/sdk:3.1-buster AS build-env
WORKDIR /app

COPY *.csproj ./
RUN dotnet restore

COPY . ./
RUN dotnet publish -c Release -o out

FROM mcr.microsoft.com/dotnet/core/runtime:3.1-buster-slim
WORKDIR /app
COPY --from=build-env /app/out ./

RUN useradd -ms /bin/bash appuser
USER appuser

ENTRYPOINT ["dotnet", "LeafDeviceApp.dll"]
```

* Add a `.dockerignore` file to not include any build artefacts from current machine into docker build process.

```text
[b|B]in
[O|o]bj
```

* Download root CA certificate from edge device VM. Make sure to copy the downloaded file to the project root directory if you download it to another folder. And remember to update the IP to your edge VM public IP or DNS name.

```text
scp -p edgeadmin@52.163.61.25:~/certificates/certs/azure-iot-test-only.root.ca.cert.pem .
```

* Add the following snippet to `LeafDeviceApp.csproj` file to include certificate file in output directory.

```markup
<ItemGroup> 
  <Content Include="*.pem"> 
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory> 
  </Content> 
</ItemGroup>
```

* Build docker image for leaf device.

```text
docker build --tag leaf-device .
```

* Add a `docker-compose.yml` file to project folder that will host 4 containers for our leaf devices. Plug in your devices connection strings and update gateway host name if needed.

```text
version: '3.3'

services:
   leafDevice1:
     image: leaf-device
     environment:
       CONNECTION_STRING: <Device1ConnectionString>;GatewayHostName=yousry-edgevm.southeastasia.cloudapp.azure.com
       DEVICE_LABEL: deviceA

   leafDevice2:
     image: leaf-device
     environment:
       CONNECTION_STRING: <Device2ConnectionString>;GatewayHostName=yousry-edgevm.southeastasia.cloudapp.azure.com
       DEVICE_LABEL: deviceB

   leafDevice3:
     image: leaf-device
     environment:
       CONNECTION_STRING: <Device3ConnectionString>;GatewayHostName=yousry-edgevm.southeastasia.cloudapp.azure.com
       DEVICE_LABEL: deviceC

   leafDevice4:
     image: leaf-device
     environment:
       CONNECTION_STRING: <Device4ConnectionString>;GatewayHostName=yousry-edgevm.southeastasia.cloudapp.azure.com
       DEVICE_LABEL: deviceD
```

* From root project folder run `docker-compose up` 

![](../.gitbook/assets/image%20%28145%29.png)

* You will see 4 containers running everyone acts as a leaf device and all of them communicate to IoT hub via IoT Edge device gateway. Leaf devices are not aware of the gateway existence.
* In Azure CLI, monitor messages received by IoT hub to confirm end to end functionality.

```text
iot hub monitor-events --hub-name master-hub
```

![](../.gitbook/assets/image%20%28210%29.png)

* A device label element is added to message payload although that's not strictly necessary as IoT hub could simply identify source device. Source device has been forwarded correctly from IoT Edge gateway acting as a transparent proxy.

### Wrap up

You can try [offline capabilities](../integration-with-azure-services/offline-capabilities.md) ideas with this example to verify that no message will be lost if internet connection on edge device is lost for some time.

Exit docker compose by pressing Ctrl+C and shut down edge VM if needed.

