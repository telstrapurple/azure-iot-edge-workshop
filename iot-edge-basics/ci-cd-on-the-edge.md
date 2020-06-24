# CI/CD on the edge

![https://unsplash.com/photos/Dph00R2SwFo](../.gitbook/assets/image%20%2879%29.png)

In this article, you will see how to manage SDLC process for IoT Edge projects similar to the common practices used in other project types. We will continue from the solution developed in [last page](local-development-and-debugging.md). But first make sure you have an Azure DevOps organisation with permissions to create projects. 

### Create DevOps project and push code

Login to your DevOps organisation and create a new project.

![](../.gitbook/assets/image%20%28159%29.png)

Switch to VS Code and open Source Control view and click the button to initialise repository.

![](../.gitbook/assets/image%20%2882%29.png)

This will simply do a git init in the solution folder.

![](../.gitbook/assets/image%20%28194%29.png)

Push the local code to the remote but replace the remote URL with your own URL.

```text
git remote add origin https://[YOUR-ACCOUNT]@dev.azure.com/[YOUR-ORGANISATION]/EdgeOps/_git/EdgeOps
git add .
git commit -m "Initial commit"
git push -u origin --all
```

Make sure the solution files appear in DevOps project now.

![](../.gitbook/assets/image%20%28127%29.png)

Now before proceeding with DevOps pipelines, we will set the stage of what we are going to do. First assume the edge device we have \(the virtual machine\) is not a development device but rather a CI environment device. It has to be cleared of any non-system running modules. The build pipeline is supposed to build our code changes then a release pipeline will deploy the solution to this device.

Open the edge device details screen in Azure portal. You may find modules from the debugging article shown in the list of modules on the device. They are not running though, they are just to help with local development and debugging.

![](../.gitbook/assets/image%20%2813%29.png)

Click **Set Modules** button and follow the steps to clear all modules and routes on the device.

![](../.gitbook/assets/image%20%2889%29.png)

### Setup build pipeline

In Azure DevOps create a new build pipeline and according to the following steps:

* Use the classic editor
* In **Select Repository** page, proceed with default settings.
* In **Choose a template** page, select start with an empty job.
* Change Agent Pool to Ubuntu 18.

![](../.gitbook/assets/image%20%2857%29.png)

* Add the following tasks to Agent job 1
  *  Azure IoT Edge
  * Azure IoT Edge \(yes, once again\)
  * Copy Files
  * Publish Build Artifacts
* Edit the properties of the first IoT Edge task as follows
  * Keep the default **name** and **action** as is, this task will build images for any custom modules in the solution so the default action of build module images is fine.
  * Browse and select the location of `deployment.template.json` file: in the repository.
  * Leave the **default platform** as **amd64.** If we are targeting a different device we might have to change it.
  * Expand **output variables** section and fill something memorable for **Reference name**. This reference will be used in later steps of the pipeline. It's a prefix to that will be applied to a variable name holding the location of generated deployment manifest file.
* Let's switch to variables tab and add a few variables. Because `.env` file is git ignored, Azure container registry details are not included with files pushed to Azure DevOps repo. Add the following variables:

  * Copy the variable names and values from solution `.env` files and create them in variables tab
  * Add another variable in `.env` file locally to hold the registry name. Currently, registry name is hardcoded in `module.json` file of each custom module. So if we want  to have it also configurable \(such that build pipeline could point to another registry other than developer setup\), we need to define another variable.
  * If you define this registry address variable, you have to update `module.json` repository element to look like `"repository": "${CONTAINER_REGISTRY_ADDRESS}/temperatureprocessingmodule",` and also update `template.json` to look like this:

  ```text
  "registryCredentials": {
      "yousrycr": {
        "username": "$CONTAINER_REGISTRY_USERNAME_yousrycr",
        "password": "$CONTAINER_REGISTRY_PASSWORD_yousrycr",
        "address": "$CONTAINER_REGISTRY_ADDRESS"
      }
    }
  ```

  * Git commit and push your changes.

* Edit the second Azure IoT Edge task which pushes all module images to the container registry that you select.
  * Change **action** to **Push module images** and this will also change the description.
  * For **Container registry type**, select **Azure Container Registry**
  * Select your **Azure subscription** and the **container registry** used. You might need to authorize DevOps to that subscription.
  * What we are doing here is granting DevOps access to push any custom modules images to the target registry. **Note:** _****The variables we have defined before are merely to create the correct content inside the deployment manifest file but they are not used by the pipeline to access the registry._
  * **.template.json** file and **Default Platform** should have right default values but you can tweak them if you have a different template file location or target platform.
* Edit Copy Files task as follows:
  * Change **Display name** to **Copy Files to: Drop folder**.
  * Replace the **contents** field value with two lines containing  `deployment.template.json` and `**/module.json`
  * Set  **Target Folder** to `$(Build.ArtifactStagingDirectory)`
* The last task of publishing artifacts should be fine with its default values. It will pull any files copied to `$(Build.ArtifactStagingDirectory)` and publish them as an artefact called **drop.**
* Save the the build pipeline

### Versioning

Have  a look on Azure Portal or using the CLI on the image built from the previous build run.

`acr repository show-tags --name [YOUR-CR-NAME] --repository temperatureprocessingmodule` 

![](../.gitbook/assets/image%20%28161%29.png)

Well, currently for any code change in the module source code we would always get the single tag of 0.0.1. Let's fix that. 

* Open `module.json` inside the C\# module folder and change the version element value from `0.0.1` to `$BUILD_TAG_VERSION`  and save the file. Don't git push now, we will do that shortly.
* Edit your `.env` file to add this new variable `BUILD_TAG_VERSION=local-dev-0.0.1` 
* For local development you can usually increment the version if you usually develop and deploy to a virtual/real device. It's assumed that a single device wouldn't be used for both local development and CI environment. Also, a developer may have his own container registry different than the one used for CI/CD pipeline.
* For Azure DevOps, we will bind this variable to a version string based off build number/sequence.
* Build and push this IoT Edge solution \(from command palette or right clicking deployment template file\). If you inspect images built, you will find a new tag for the local development image.

![](../.gitbook/assets/image%20%2878%29.png)

To do the same for the build pipeline, switch to ADO \(Azure DevOps\) and edit the build pipeline as follows:

* Change build number format to `1.0$(Rev:.r)` 

![](../.gitbook/assets/image%20%28205%29.png)

* Add build variable named `BUILD_TAG_VERSION` with value `$(Build.BuildNumber)` 

![](../.gitbook/assets/image%20%2834%29.png)

* Save build definition
* Git push your local changes
* Trigger the build manually or enable continuous integration if you want

![](../.gitbook/assets/image%20%2852%29.png)

* Now if you inspect container registry repository tags, you will see the tag for the new build

![](../.gitbook/assets/image%20%28105%29.png)

* Artefacts generated with the build should look like below

![](../.gitbook/assets/image%20%28188%29.png)

### Release pipeline

Now we are done with the build, it's time to configure release pipeline.

* Create a new release pipeline in the same project
* Choose an **Empty job** as the template for the new pipeline
* Rename **Stage 1** to **Dev**
* Click **Add an artifact** button and link the release with artefact created from the build pipeline in this project. Change the source alias to **Source**

![](../.gitbook/assets/image%20%28189%29.png)

* You can also enable the continuous deployment trigger such that a new release will be created each time a new build is available.
* Click the link inside the **Dev** stage card to edit its tasks
* Change **Agent Job** to have a specification of **ubuntu-18.04**
* Add two **Azure IoT Edge** tasks
* Edit the first task as follows:
  * Change action to **Generate deployment manifest**.
  * Change template.json file path to `$(System.DefaultWorkingDirectory)/Source/drop/deployment.template.json` or use the browse button to pick it.
  * Make sure default platform is **amd64**
  * Change output path to `$(System.DefaultWorkingDirectory)/Source/drop/configs/deployment.json` 
* Edit the second task as follows:

  * Change action to **Deploy to IoT Edge devices**
  * Change deployment file to `$(System.DefaultWorkingDirectory)/Drop/drop/configs/deployment.json` which is the output file from previous task.
  * Select subscription, hub name and choose the option to deploy to single device and provide your device name \(in our case **edge-device**\).
  * Update advanced section to have the deployment Id variable to have a value like `$(System.TeamProject)-$(Release.EnvironmentName)-$(Release.ReleaseName)` . This is a unique name that will appear in IoT hub showing what deployment is currently deployed on the device. 

![](../.gitbook/assets/image%20%28135%29.png)

* Edit the release variables to include container registry details and build version. Please note that the variables in the build pipeline are there to allow tagging and pushing the image to the container registry. Variables in release pipeline are to pull image and deploy to the device. Please note the value of build tag version variable `$(Release.Artifacts.Source.BuildNumber)` 

![](../.gitbook/assets/image%20%2856%29.png)

* Save release definition
* Create a new release for the **Dev** stage and link it to the latest build.
* The deployment should work fine assuming all configuration is good.

![](../.gitbook/assets/image%20%2841%29.png)

* Inspecting modules running on the device \(VM\), we can see the expected modules deployed and the custom module with a version matching the release/build from ADO.

![](../.gitbook/assets/image%20%28185%29.png)

* In Azure Portal, there will be a deployment entry in the edge device details.

![](../.gitbook/assets/image%20%28169%29.png)

### Nice to have

Enable auto triggering of the build and release and do a slight change in the code. For example, change the text of the console message when temperature is below threshold.

![](../.gitbook/assets/image%20%28209%29.png)

Git push the change and watch the pipeline triggered and deploy to the device directly.

{% hint style="info" %}
Release pipeline can be extended with a shell task to inspect modules running on the device and make sure the deployment is successful.
{% endhint %}





