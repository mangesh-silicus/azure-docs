﻿---
title: Create an Azure Service Fabric container application | Microsoft Docs
description: Create your first Windows container application on Azure Service Fabric.  Build a Docker image with a Python application, push the image to a container registry, build and deploy a Service Fabric container application.
services: service-fabric
documentationcenter: .net
author: rwike77
manager: timlt
editor: 'vturecek'

ms.assetid:
ms.service: service-fabric
ms.devlang: dotNet
ms.topic: get-started-article
ms.tgt_pltfrm: NA
ms.workload: NA
ms.date: 11/03/2017
ms.author: ryanwi

---

# Create your first Service Fabric container application on Windows
> [!div class="op_single_selector"]
> * [Windows](service-fabric-get-started-containers.md)
> * [Linux](service-fabric-get-started-containers-linux.md)

Running an existing application in a Windows container on a Service Fabric cluster doesn't require any changes to your application. This article walks you through creating a Docker image containing a Python [Flask](http://flask.pocoo.org/) web application and deploying it to a Service Fabric cluster.  You will also share your containerized application through [Azure Container Registry](/azure/container-registry/).  This article assumes a basic understanding of Docker. You can learn about Docker by reading the [Docker Overview](https://docs.docker.com/engine/understanding-docker/).

## Prerequisites
A development computer running:
* Visual Studio 2015 or Visual Studio 2017.
* [Service Fabric SDK and tools](service-fabric-get-started.md).
*  Docker for Windows.  [Get Docker CE for Windows (stable)](https://store.docker.com/editions/community/docker-ce-desktop-windows?tab=description). After installing and starting Docker, right-click on the tray icon and select **Switch to Windows containers**. This is required to run Docker images based on Windows.

A Windows cluster with three or more nodes running on Windows Server 2016 with Containers- [Create a cluster](service-fabric-cluster-creation-via-portal.md) or [try Service Fabric for free](https://aka.ms/tryservicefabric).

A registry in Azure Container Registry - [Create a container registry](../container-registry/container-registry-get-started-portal.md) in your Azure subscription.

## Define the Docker container
Build an image based on the [Python image](https://hub.docker.com/_/python/) located on Docker Hub.

Define your Docker container in a Dockerfile. The Dockerfile contains instructions for setting up the environment inside your container, loading the application you want to run, and mapping ports. The Dockerfile is the input to the `docker build` command, which creates the image.

Create an empty directory and create the file *Dockerfile* (with no file extension). Add the following to *Dockerfile* and save your changes:

```
# Use an official Python runtime as a base image
FROM python:2.7-windowsservercore

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
ADD . /app

# Install any needed packages specified in requirements.txt
RUN pip install -r requirements.txt

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]
```

Read the [Dockerfile reference](https://docs.docker.com/engine/reference/builder/) for more information.

## Create a simple web application
Create a Flask web application listening on port 80 that returns "Hello World!".  In the same directory, create the file *requirements.txt*.  Add the following and save your changes:
```
Flask
```

Also create the *app.py* file and add the following:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():

    return 'Hello World!'

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```

<a id="Build-Containers"></a>
## Build the image
Run the `docker build` command to create the image that runs your web application. Open a PowerShell window and navigate to the directory containing the Dockerfile. Run the following command:

```
docker build -t helloworldapp .
```

This command builds the new image using the instructions in your Dockerfile, naming (-t tagging) the image "helloworldapp". Building an image pulls the base image down from Docker Hub and creates a new image that adds your application on top of the base image.  

Once the build command completes, run the `docker images` command to see information on the new image:

```
$ docker images

REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
helloworldapp                 latest              8ce25f5d6a79        2 minutes ago       10.4 GB
```

## Run the application locally
Verify your image locally before pushing it the container registry.  

Run the application:

```
docker run -d --name my-web-site helloworldapp
```

*name* gives a name to the running container (instead of the container ID).

Once the container starts, find its IP address so that you can connect to your running container from a browser:
```
docker inspect -f "{{ .NetworkSettings.Networks.nat.IPAddress }}" my-web-site
```

Connect to the running container.  Open a web browser pointing to the IP address returned, for example "http://172.31.194.61". You should see the heading "Hello World!" display in the browser.

To stop your container, run:

```
docker stop my-web-site
```

Delete the container from your development machine:

```
docker rm my-web-site
```

<a id="Push-Containers"></a>
## Push the image to the container registry
After you verify that the container runs on your development machine, push the image to your registry in Azure Container Registry.

Run ``docker login`` to log in to your container registry with your [registry credentials](../container-registry/container-registry-authentication.md).

The following example passes the ID and password of an Azure Active Directory [service principal](../active-directory/active-directory-application-objects.md). For example, you might have assigned a service principal to your registry for an automation scenario. Or, you could login using your registry username and password.

```
docker login myregistry.azurecr.io -u xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx -p myPassword
```

The following command creates a tag, or alias, of the image, with a fully qualified path to your registry. This example places the image in the ```samples``` namespace to avoid clutter in the root of the registry.

```
docker tag helloworldapp myregistry.azurecr.io/samples/helloworldapp
```

Push the image to your container registry:

```
docker push myregistry.azurecr.io/samples/helloworldapp
```

## Create the containerized service in Visual Studio
The Service Fabric SDK and tools provide a service template to help you create a containerized application.

1. Start Visual Studio.  Select **File** > **New** > **Project**.
2. Select **Service Fabric application**, name it "MyFirstContainer", and click **OK**.
3. Select **Container** from the list of **service templates**.
4. In **Image Name** enter "myregistry.azurecr.io/samples/helloworldapp", the image you pushed to your container repository.
5. Give your service a name, and click **OK**.

## Configure communication
The containerized service needs an endpoint for communication. Add an `Endpoint` element with the protocol, port, and type to the ServiceManifest.xml file. For this article, the containerized service listens on port 8081.  In this example, a fixed port 8081 is used.  If no port is specified, a random port from the application port range is chosen.  

```xml
<Resources>
  <Endpoints>
    <Endpoint Name="Guest1TypeEndpoint" UriScheme="http" Port="8081" Protocol="http"/>
  </Endpoints>
</Resources>
```

By defining an endpoint, Service Fabric publishes the endpoint to the Naming service.  Other services running in the cluster can resolve this container.  You can also perform container-to-container communication using the [reverse proxy](service-fabric-reverseproxy.md).  Communication is performed by providing the reverse proxy HTTP listening port and the name of the services that you want to communicate with as environment variables.

## Configure and set environment variables
Environment variables can be specified for each code package in the service manifest. This feature is available for all services irrespective of whether they are deployed as containers or processes or guest executables. You can override environment variable values in the application manifest or specify them during deployment as application parameters.

The following service manifest XML snippet shows an example of how to specify environment variables for a code package:
```xml
<CodePackage Name="Code" Version="1.0.0">
  ...
  <EnvironmentVariables>
    <EnvironmentVariable Name="HttpGatewayPort" Value=""/>    
  </EnvironmentVariables>
</CodePackage>
```

These environment variables can be overridden in the application manifest:

```xml
<ServiceManifestImport>
  <ServiceManifestRef ServiceManifestName="Guest1Pkg" ServiceManifestVersion="1.0.0" />
    <EnvironmentVariable Name="HttpGatewayPort" Value="19080"/>
  </EnvironmentOverrides>
  ...
</ServiceManifestImport>
```

## Configure container port-to-host port mapping and container-to-container discovery
Configure a host port used to communicate  with the container. The port binding maps the port on which the service is listening inside the container to a port on the host. Add a `PortBinding` element in `ContainerHostPolicies` element of the ApplicationManifest.xml file.  For this article, `ContainerPort` is 80 (the container exposes port 80, as specified in the Dockerfile) and `EndpointRef` is "Guest1TypeEndpoint" (the endpoint previously defined in the service manifest).  Incoming requests to the service on port 8081 are mapped to port 80 on the container.

```xml
<Policies>
  <ContainerHostPolicies CodePackageRef="Code">
    <PortBinding ContainerPort="80" EndpointRef="Guest1TypeEndpoint"/>
  </ContainerHostPolicies>
</Policies>
```

## Configure container registry authentication
Configure container registry authentication by adding `RepositoryCredentials` to `ContainerHostPolicies` of the ApplicationManifest.xml file. Add the account and password for the myregistry.azurecr.io container registry, which allows the service to download the container image from the repository.

```xml
<Policies>
    <ContainerHostPolicies CodePackageRef="Code">
        <RepositoryCredentials AccountName="myregistry" Password="=P==/==/=8=/=+u4lyOB=+=nWzEeRfF=" PasswordEncrypted="false"/>
        <PortBinding ContainerPort="80" EndpointRef="Guest1TypeEndpoint"/>
    </ContainerHostPolicies>
</Policies>
```

We recommend that you encrypt the repository password by using an encipherment certificate that's deployed to all nodes of the cluster. When Service Fabric deploys the service package to the cluster, the encipherment certificate is used to decrypt the cipher text.  The Invoke-ServiceFabricEncryptText cmdlet is used to create the cipher text for the password, which is added to the ApplicationManifest.xml file.

The following script creates a new self-signed certificate and exports it to a PFX file.  The certificate is imported into an existing key vault and then deployed to the Service Fabric cluster.

```powershell
# Variables.
$certpwd = ConvertTo-SecureString -String "Pa$$word321!" -Force -AsPlainText
$filepath = "C:\MyCertificates\dataenciphermentcert.pfx"
$subjectname = "dataencipherment"
$vaultname = "mykeyvault"
$certificateName = "dataenciphermentcert"
$groupname="myclustergroup"
$clustername = "mycluster"

$subscriptionId = "subscription ID"

Login-AzureRmAccount

Select-AzureRmSubscription -SubscriptionId $subscriptionId

# Create a self signed cert, export to PFX file.
New-SelfSignedCertificate -Type DocumentEncryptionCert -KeyUsage DataEncipherment -Subject $subjectname -Provider 'Microsoft Enhanced Cryptographic Provider v1.0' `
| Export-PfxCertificate -FilePath $filepath -Password $certpwd

# Import the certificate to an existing key vault.  The key vault must be enabled for deployment.
$cer = Import-AzureKeyVaultCertificate -VaultName $vaultName -Name $certificateName -FilePath $filepath -Password $certpwd

Set-AzureRmKeyVaultAccessPolicy -VaultName $vaultName -ResourceGroupName $groupname -EnabledForDeployment

# Add the certificate to all the VMs in the cluster.
Add-AzureRmServiceFabricApplicationCertificate -ResourceGroupName $groupname -Name $clustername -SecretIdentifier $cer.SecretId
```
Encrypt the password using the [Invoke-ServiceFabricEncryptText](/powershell/module/servicefabric/Invoke-ServiceFabricEncryptText?view=azureservicefabricps) cmdlet.

```powershell
$text = "=P==/==/=8=/=+u4lyOB=+=nWzEeRfF="
Invoke-ServiceFabricEncryptText -CertStore -CertThumbprint $cer.Thumbprint -Text $text -StoreLocation Local -StoreName My
```

Replace the password with the cipher text returned by the [Invoke-ServiceFabricEncryptText](/powershell/module/servicefabric/Invoke-ServiceFabricEncryptText?view=azureservicefabricps) cmdlet and set `PasswordEncrypted` to "true".

```xml
<Policies>
  <ContainerHostPolicies CodePackageRef="Code">
    <RepositoryCredentials AccountName="myregistry" Password="MIIB6QYJKoZIhvcNAQcDoIIB2jCCAdYCAQAxggFRMIIBTQIBADA1MCExHzAdBgNVBAMMFnJ5YW53aWRhdGFlbmNpcGhlcm1lbnQCEFfyjOX/17S6RIoSjA6UZ1QwDQYJKoZIhvcNAQEHMAAEg
gEAS7oqxvoz8i6+8zULhDzFpBpOTLU+c2mhBdqXpkLwVfcmWUNA82rEWG57Vl1jZXe7J9BkW9ly4xhU8BbARkZHLEuKqg0saTrTHsMBQ6KMQDotSdU8m8Y2BR5Y100wRjvVx3y5+iNYuy/JmM
gSrNyyMQ/45HfMuVb5B4rwnuP8PAkXNT9VLbPeqAfxsMkYg+vGCDEtd8m+bX/7Xgp/kfwxymOuUCrq/YmSwe9QTG3pBri7Hq1K3zEpX4FH/7W2Zb4o3fBAQ+FuxH4nFjFNoYG29inL0bKEcTX
yNZNKrvhdM3n1Uk/8W2Hr62FQ33HgeFR1yxQjLsUu800PrYcR5tLfyTB8BgkqhkiG9w0BBwEwHQYJYIZIAWUDBAEqBBBybgM5NUV8BeetUbMR8mJhgFBrVSUsnp9B8RyebmtgU36dZiSObDsI
NtTvlzhk11LIlae/5kjPv95r3lw6DHmV4kXLwiCNlcWPYIWBGIuspwyG+28EWSrHmN7Dt2WqEWqeNQ==" PasswordEncrypted="true"/>
    <PortBinding ContainerPort="80" EndpointRef="Guest1TypeEndpoint"/>
  </ContainerHostPolicies>
</Policies>
```

## Configure isolation mode
Windows supports two isolation modes for containers: process and Hyper-V. With the process isolation mode, all the containers running on the same host machine share the kernel with the host. With the Hyper-V isolation mode, the kernels are isolated between each Hyper-V container and the container host. The isolation mode is specified in the `ContainerHostPolicies` element in the application manifest file. The isolation modes that can be specified are `process`, `hyperv`, and `default`. The default isolation mode defaults to `process` on Windows Server hosts, and defaults to `hyperv` on Windows 10 hosts. The following snippet shows how the isolation mode is specified in the application manifest file.

```xml
<ContainerHostPolicies CodePackageRef="Code" Isolation="hyperv">
```
   > [!NOTE]
   > The hyperv isolation mode is available on Ev3 and Dv3 Azure SKUs which have nested virtualization support. 
   > Ensure the hyperv role is installed on the hosts. Verify this by connecting to the hosts.
   >

## Configure resource governance
[Resource governance](service-fabric-resource-governance.md) restricts the resources that the container can use on the host. The `ResourceGovernancePolicy` element, which is specified in the application manifest, is used to declare resource limits for a service code package. Resource limits can be set for the following resources: Memory, MemorySwap, CpuShares (CPU relative weight), MemoryReservationInMB, BlkioWeight (BlockIO relative weight).  In this example, service package Guest1Pkg gets one core on the cluster nodes where it is placed.  Memory limits are absolute, so the code package is limited to 1024 MB of memory (and a soft-guarantee reservation of the same). Code packages (containers or processes) are not able to allocate more memory than this limit, and attempting to do so results in an out-of-memory exception. For resource limit enforcement to work, all code packages within a service package should have memory limits specified.

```xml
<ServiceManifestImport>
  <ServiceManifestRef ServiceManifestName="Guest1Pkg" ServiceManifestVersion="1.0.0" />
  <Policies>
    <ServicePackageResourceGovernancePolicy CpuCores="1"/>
    <ResourceGovernancePolicy CodePackageRef="Code" MemoryInMB="1024"  />
  </Policies>
</ServiceManifestImport>
```

## Deploy the container application
Save all your changes and build the application. To publish your application, right-click on **MyFirstContainer** in Solution Explorer and select **Publish**.

In **Connection Endpoint**, enter the management endpoint for the cluster.  For example, "containercluster.westus2.cloudapp.azure.com:19000". You can find the client connection endpoint in the Overview blade for your cluster in the [Azure portal](https://portal.azure.com).

Click **Publish**.

[Service Fabric Explorer](service-fabric-visualizing-your-cluster.md) is a web-based tool for inspecting and managing applications and nodes in a Service Fabric cluster. Open a browser and navigate to http://containercluster.westus2.cloudapp.azure.com:19080/Explorer/ and follow the application deployment.  The application deploys but is in an error state until the image is downloaded on the cluster nodes (which can take some time, depending on the image size):
![Error][1]

The application is ready when it's in ```Ready``` state:
![Ready][2]

Open a browser and navigate to http://containercluster.westus2.cloudapp.azure.com:8081. You should see the heading "Hello World!" display in the browser.

## Clean up
You continue to incur charges while the cluster is running, consider [deleting your cluster](service-fabric-get-started-azure-cluster.md#remove-the-cluster).  [Party clusters](https://try.servicefabric.azure.com/) are automatically deleted after a few hours.

After you push the image to the container registry you can delete the local image from your development computer:

```
docker rmi helloworldapp
docker rmi myregistry.azurecr.io/samples/helloworldapp
```

## Complete example Service Fabric application and service manifests
Here are the complete service and application manifests used in this article.

### ServiceManifest.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<ServiceManifest Name="Guest1Pkg"
                 Version="1.0.0"
                 xmlns="http://schemas.microsoft.com/2011/01/fabric"
                 xmlns:xsd="http://www.w3.org/2001/XMLSchema"
                 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <ServiceTypes>
    <!-- This is the name of your ServiceType.
         The UseImplicitHost attribute indicates this is a guest service. -->
    <StatelessServiceType ServiceTypeName="Guest1Type" UseImplicitHost="true" />
  </ServiceTypes>

  <!-- Code package is your service executable. -->
  <CodePackage Name="Code" Version="1.0.0">
    <EntryPoint>
      <!-- Follow this link for more information about deploying Windows containers to Service Fabric: https://aka.ms/sfguestcontainers -->
      <ContainerHost>
        <ImageName>myregistry.azurecr.io/samples/helloworldapp</ImageName>
      </ContainerHost>
    </EntryPoint>
    <!-- Pass environment variables to your container: -->    
    <EnvironmentVariables>
      <EnvironmentVariable Name="HttpGatewayPort" Value=""/>
      <EnvironmentVariable Name="BackendServiceName" Value=""/>
    </EnvironmentVariables>

  </CodePackage>

  <!-- Config package is the contents of the Config directoy under PackageRoot that contains an
       independently-updateable and versioned set of custom configuration settings for your service. -->
  <ConfigPackage Name="Config" Version="1.0.0" />

  <Resources>
    <Endpoints>
      <!-- This endpoint is used by the communication listener to obtain the port on which to
           listen. Please note that if your service is partitioned, this port is shared with
           replicas of different partitions that are placed in your code. -->
      <Endpoint Name="Guest1TypeEndpoint" UriScheme="http" Port="8081" Protocol="http"/>
    </Endpoints>
  </Resources>
</ServiceManifest>
```
### ApplicationManifest.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<ApplicationManifest ApplicationTypeName="MyFirstContainerType"
                     ApplicationTypeVersion="1.0.0"
                     xmlns="http://schemas.microsoft.com/2011/01/fabric"
                     xmlns:xsd="http://www.w3.org/2001/XMLSchema"
                     xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <Parameters>
    <Parameter Name="Guest1_InstanceCount" DefaultValue="-1" />
  </Parameters>
  <!-- Import the ServiceManifest from the ServicePackage. The ServiceManifestName and ServiceManifestVersion
       should match the Name and Version attributes of the ServiceManifest element defined in the
       ServiceManifest.xml file. -->
  <ServiceManifestImport>
    <ServiceManifestRef ServiceManifestName="Guest1Pkg" ServiceManifestVersion="1.0.0" />
    <EnvironmentOverrides CodePackageRef="FrontendService.Code">
      <EnvironmentVariable Name="HttpGatewayPort" Value="19080"/>
    </EnvironmentOverrides>
    <ConfigOverrides />
    <Policies>
      <ContainerHostPolicies CodePackageRef="Code">
        <RepositoryCredentials AccountName="myregistry" Password="MIIB6QYJKoZIhvcNAQcDoIIB2jCCAdYCAQAxggFRMIIBTQIBADA1MCExHzAdBgNVBAMMFnJ5YW53aWRhdGFlbmNpcGhlcm1lbnQCEFfyjOX/17S6RIoSjA6UZ1QwDQYJKoZIhvcNAQEHMAAEg
gEAS7oqxvoz8i6+8zULhDzFpBpOTLU+c2mhBdqXpkLwVfcmWUNA82rEWG57Vl1jZXe7J9BkW9ly4xhU8BbARkZHLEuKqg0saTrTHsMBQ6KMQDotSdU8m8Y2BR5Y100wRjvVx3y5+iNYuy/JmM
gSrNyyMQ/45HfMuVb5B4rwnuP8PAkXNT9VLbPeqAfxsMkYg+vGCDEtd8m+bX/7Xgp/kfwxymOuUCrq/YmSwe9QTG3pBri7Hq1K3zEpX4FH/7W2Zb4o3fBAQ+FuxH4nFjFNoYG29inL0bKEcTX
yNZNKrvhdM3n1Uk/8W2Hr62FQ33HgeFR1yxQjLsUu800PrYcR5tLfyTB8BgkqhkiG9w0BBwEwHQYJYIZIAWUDBAEqBBBybgM5NUV8BeetUbMR8mJhgFBrVSUsnp9B8RyebmtgU36dZiSObDsI
NtTvlzhk11LIlae/5kjPv95r3lw6DHmV4kXLwiCNlcWPYIWBGIuspwyG+28EWSrHmN7Dt2WqEWqeNQ==" PasswordEncrypted="true"/>
        <PortBinding ContainerPort="80" EndpointRef="Guest1TypeEndpoint"/>
      </ContainerHostPolicies>
      <ServicePackageResourceGovernancePolicy CpuCores="1"/>
      <ResourceGovernancePolicy CodePackageRef="Code" MemoryInMB="1024"  />
    </Policies>
  </ServiceManifestImport>
  <DefaultServices>
    <!-- The section below creates instances of service types, when an instance of this
         application type is created. You can also create one or more instances of service type using the
         ServiceFabric PowerShell module.

         The attribute ServiceTypeName below must match the name defined in the imported ServiceManifest.xml file. -->
    <Service Name="Guest1">
      <StatelessService ServiceTypeName="Guest1Type" InstanceCount="[Guest1_InstanceCount]">
        <SingletonPartition />
      </StatelessService>
    </Service>
  </DefaultServices>
</ApplicationManifest>
```

## Configure time interval before container is force terminated

You can configure a time interval for the runtime to wait before the container is removed after the service deletion (or a move to another node) has started. Configuring the time interval sends the `docker stop <time in seconds>` command to the container.   For more detail, see [docker stop](https://docs.docker.com/engine/reference/commandline/stop/). The time interval to wait is specified under the `Hosting` section. The following cluster manifest snippet shows how to set the wait interval:

```xml
{
        "name": "Hosting",
        "parameters": [
          {
            "ContainerDeactivationTimeout": "10",
	      ...
          }
        ]
}
```
The default time interval is set to 10 seconds. Since this configuration is dynamic, a config only upgrade on the cluster updates the timeout. 


## Configure the runtime to remove unused container images

You can configure the Service Fabric cluster to remove unused container images from the node. This configuration allows disk space to be recaptured if too many container images are present on the node.  To enable this feature, update the `Hosting` section in the cluster manifest as shown in the following snippet: 


```xml
{
        "name": "Hosting",
        "parameters": [
          {
	        "PruneContainerImages": “True”,
            "ContainerImagesToSkip": "microsoft/windowsservercore|microsoft/nanoserver|…",
	      ...
          }
        ]
} 
```

For images that should not be deleted, you can specify them under the `ContainerImagesToSkip` parameter. 



## Next steps
* Learn more about running [containers on Service Fabric](service-fabric-containers-overview.md).
* Read the [Deploy a .NET application in a container](service-fabric-host-app-in-a-container.md) tutorial.
* Learn about the Service Fabric [application life-cycle](service-fabric-application-lifecycle.md).
* Checkout the [Service Fabric container code samples](https://github.com/Azure-Samples/service-fabric-containers) on GitHub.

[1]: ./media/service-fabric-get-started-containers/MyFirstContainerError.png
[2]: ./media/service-fabric-get-started-containers/MyFirstContainerReady.png
