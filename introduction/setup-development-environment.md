# Setup development environment

![https://unsplash.com/photos/9HI8UJMSdZA](../.gitbook/assets/image%20%2874%29.png)

The following items are required to proceed with this master class/workshop.

* Azure subscription with some decent credit, around 30 AUD would be enough
* Azure DevOps organisation with permission to create projects
* [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) 
* [Visual Studio Code](https://code.visualstudio.com/download)
* The following VS Code extensions 
  1. [Azure IoT Tools](https://marketplace.visualstudio.com/items?itemName=vsciot-vscode.azure-iot-tools) which is an extension pack that includes IoT hub, IoT edge and IoT Device Workbench
  2. [C\#](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csharp)
  3. [Docker](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-docker)
* [Docker Community Edition](https://hub.docker.com/search?offering=community&type=edition) \(with Linux containers\)
* [Git](https://git-scm.com/downloads)
* .[NET Core 3.1 SDK](https://dotnet.microsoft.com/download/dotnet-core/3.1)

### Azure CLI configuration

We will assume that you will have your CLI logged in to your Azure account. It's also nice to configure the CLI with some defaults like showing results in tables.

![Azure CLI configuration](../.gitbook/assets/image%20%28208%29.png)

Also I will use [CLI interactive](https://docs.microsoft.com/en-us/cli/azure/interactive-azure-cli?view=azure-cli-latest) mode to have stuff like auto-complete and command descriptions. To enter interactive mode, simply type `az interactive` Most screenshots will be done from interactive mode.

To install IoT hub extension, run:

`az extension add --name azure-iot` 

### Create IoT hub

The core Azure resource we need \(at least in the cloud\) is Azure IoT hub. So, we need to create one. 

First, create a resource group to hold all resources in this master class. Southeast Asia region is picked because it has some Azure IoT hub preview features that we _may_ use like distributed tracing. 

`az group create --name iotedge-rg --location southeastasia`

![](../.gitbook/assets/image%20%2873%29.png)

Create a standard tier IoT hub. Standard tier is mainly required to use IoT Edge. The general advice is to use standard tier unless your solution is basically around telemetry collection. You will notice that in interactive mode we can skip typing the **az** command altogether. **Also IoT Hub name should be globally unique**. So try replacing any future `master-hub` reference with `<your-first-name>-master-hub` .

`iot hub create --resource-group iotedge-rg --sku S1 --hub-name master-hub`

![](../.gitbook/assets/image%20%28187%29.png)

Open Azure portal in the browser and verify that IoT hub is created in the right resource group/region.

### Configure VS Code to connect to IoT hub

Open VS Code and make sure to sign in to your Azure Account and select the correct subscription if you have access to more than one. Use Ctrl+Shift+P to open command palette.

![](../.gitbook/assets/image%20%28139%29.png)

From command palette, search for "select iot" to pick the command for wiring VS Code with our IoT hub.

![](../.gitbook/assets/image%20%28117%29.png)

Select the command above then select the subscription and IoT hub under this subscription. Once done, our `master-hub` will be available in VS Code Explorer view.



![](../.gitbook/assets/image%20%2828%29.png)



