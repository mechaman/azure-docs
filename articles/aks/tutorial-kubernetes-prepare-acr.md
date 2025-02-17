---
title: Kubernetes on Azure tutorial - Create a container registry
description: In this Azure Kubernetes Service (AKS) tutorial, you create an Azure Container Registry instance and upload a sample application container image.
services: container-service
ms.topic: tutorial
ms.date: 12/20/2021

ms.custom: mvc, devx-track-azurecli, devx-track-azurepowershell

#Customer intent: As a developer, I want to learn how to create and use a container registry so that I can deploy my own applications to Azure Kubernetes Service.
---

# Tutorial: Deploy and use Azure Container Registry

Azure Container Registry (ACR) is a private registry for container images. A private container registry lets you securely build and deploy your applications and custom code. In this tutorial, part two of seven, you deploy an ACR instance and push a container image to it. You learn how to:

> [!div class="checklist"]
> * Create an Azure Container Registry (ACR) instance
> * Tag a container image for ACR
> * Upload the image to ACR
> * View images in your registry

In later tutorials, this ACR instance is integrated with a Kubernetes cluster in AKS, and an application is deployed from the image.

## Before you begin

In the [previous tutorial][aks-tutorial-prepare-app], a container image was created for a simple Azure Voting application. If you have not created the Azure Voting app image, return to [Tutorial 1 – Create container images][aks-tutorial-prepare-app].

### [Azure CLI](#tab/azure-cli)

This tutorial requires that you're running the Azure CLI version 2.0.53 or later. Run `az --version` to find the version. If you need to install or upgrade, see [Install Azure CLI][azure-cli-install].

### [Azure PowerShell](#tab/azure-powershell)

This tutorial requires that you're running Azure PowerShell version 5.9.0 or later. Run `Get-InstalledModule -Name Az` to find the version. If you need to install or upgrade, see [Install Azure PowerShell][azure-powershell-install].

---

## Create an Azure Container Registry

To create an Azure Container Registry, you first need a resource group. An Azure resource group is a logical container into which Azure resources are deployed and managed.

### [Azure CLI](#tab/azure-cli)

Create a resource group with the [az group create][az-group-create] command. In the following example, a resource group named *myResourceGroup* is created in the *eastus* region:

```azurecli
az group create --name myResourceGroup --location eastus
```

Create an Azure Container Registry instance with the [az acr create][az-acr-create] command and provide your own registry name. The registry name must be unique within Azure, and contain 5-50 alphanumeric characters. In the rest of this tutorial, `<acrName>` is used as a placeholder for the container registry name. Provide your own unique registry name. The *Basic* SKU is a cost-optimized entry point for development purposes that provides a balance of storage and throughput.

```azurecli
az acr create --resource-group myResourceGroup --name <acrName> --sku Basic
```

### [Azure PowerShell](#tab/azure-powershell)

Create a resource group with the [New-AzResourceGroup][new-azresourcegroup] cmdlet. In the following example, a resource group named *myResourceGroup* is created in the *eastus* region:

```azurepowershell
New-AzResourceGroup -Name myResourceGroup -Location eastus
```

Create an Azure Container Registry instance with the [New-AzContainerRegistry][new-azcontainerregistry] cmdlet and provide your own registry name. The registry name must be unique within Azure, and contain 5-50 alphanumeric characters. In the rest of this tutorial, `<acrName>` is used as a placeholder for the container registry name. Provide your own unique registry name. The *Basic* SKU is a cost-optimized entry point for development purposes that provides a balance of storage and throughput.

```azurepowershell
New-AzContainerRegistry -ResourceGroupName myResourceGroup -Name <acrname> -Sku Basic
```

---

## Log in to the container registry

### [Azure CLI](#tab/azure-cli)

To use the ACR instance, you must first log in. Use the [az acr login][az-acr-login] command and provide the unique name given to the container registry in the previous step.

```azurecli
az acr login --name <acrName>
```

### [Azure PowerShell](#tab/azure-powershell)

To use the ACR instance, you must first log in. Use the [Connect-AzContainerRegistry][connect-azcontainerregistry] cmdlet and provide the unique name given to the container registry in the previous step.

```azurepowershell
Connect-AzContainerRegistry -Name <acrName>
```

---

The command returns a *Login Succeeded* message once completed.

## Tag a container image

To see a list of your current local images, use the [docker images][docker-images] command:

```console
docker images
```
The above command's output shows list of your current local images:

```output
REPOSITORY                                     TAG                 IMAGE ID            CREATED             SIZE
mcr.microsoft.com/azuredocs/azure-vote-front   v1                  84b41c268ad9        7 minutes ago       944MB
mcr.microsoft.com/oss/bitnami/redis            6.0.8               3a54a920bb6c        2 days ago          103MB
tiangolo/uwsgi-nginx-flask                     python3.6           a16ce562e863        6 weeks ago         944MB
```

To use the *azure-vote-front* container image with ACR, the image needs to be tagged with the login server address of your registry. This tag is used for routing when pushing container images to an image registry.

### [Azure CLI](#tab/azure-cli)

To get the login server address, use the [az acr list][az-acr-list] command and query for the *loginServer* as follows:

```azurecli
az acr list --resource-group myResourceGroup --query "[].{acrLoginServer:loginServer}" --output table
```
### [Azure PowerShell](#tab/azure-powershell)

To get the login server address, use the [Get-AzContainerRegistry][get-azcontainerregistry] cmdlet and query for the *loginServer* as follows:

```azurepowershell
(Get-AzContainerRegistry -ResourceGroupName myResourceGroup -Name <acrName>).LoginServer
```

---

Now, tag your local *azure-vote-front* image with the *acrLoginServer* address of the container registry. To indicate the image version, add *:v1* to the end of the image name:

```console
docker tag mcr.microsoft.com/azuredocs/azure-vote-front:v1 <acrLoginServer>/azure-vote-front:v1
```

To verify the tags are applied, run [docker images][docker-images] again.

```console
docker images
```

An image is tagged with the ACR instance address and a version number.

```
REPOSITORY                                      TAG                 IMAGE ID            CREATED             SIZE
mcr.microsoft.com/azuredocs/azure-vote-front    v1                  84b41c268ad9        16 minutes ago      944MB
mycontainerregistry.azurecr.io/azure-vote-front v1                  84b41c268ad9        16 minutes ago      944MB
mcr.microsoft.com/oss/bitnami/redis             6.0.8               3a54a920bb6c        2 days ago          103MB
tiangolo/uwsgi-nginx-flask                      python3.6           a16ce562e863        6 weeks ago         944MB
```

## Push images to registry

With your image built and tagged, push the *azure-vote-front* image to your ACR instance. Use [docker push][docker-push] and provide your own *acrLoginServer* address for the image name as follows:

```console
docker push <acrLoginServer>/azure-vote-front:v1
```

It may take a few minutes to complete the image push to ACR.

## List images in registry

### [Azure CLI](#tab/azure-cli)

To return a list of images that have been pushed to your ACR instance, use the [az acr repository list][az-acr-repository-list] command. Provide your own `<acrName>` as follows:

```azurecli
az acr repository list --name <acrName> --output table
```

The following example output lists the *azure-vote-front* image as available in the registry:

```output
Result
----------------
azure-vote-front
```

To see the tags for a specific image, use the [az acr repository show-tags][az-acr-repository-show-tags] command as follows:

```azurecli
az acr repository show-tags --name <acrName> --repository azure-vote-front --output table
```

The following example output shows the *v1* image tagged in a previous step:

```output
Result
--------
v1
```

### [Azure PowerShell](#tab/azure-powershell)

To return a list of images that have been pushed to your ACR instance, use the [Get-AzContainerRegistryManifest][get-azcontainerregistrymanifest] cmdlet. Provide your own `<acrName>` as follows:

```azurepowershell
Get-AzContainerRegistryManifest -RegistryName <acrName> -RepositoryName azure-vote-front
```

The following example output lists the *azure-vote-front* image as available in the registry:

```output
Registry  ImageName        ManifestsAttributes
--------  ---------        -------------------
<acrName> azure-vote-front {Microsoft.Azure.Commands.ContainerRegistry.Models.PSManifestAttributeBase}
```

To see the tags for a specific image, use the [Get-AzContainerRegistryTag][get-azcontainerregistrytag] cmdlet as follows:

```azurepowershell
Get-AzContainerRegistryTag -RegistryName <acrName> -RepositoryName azure-vote-front
```

The following example output shows the *v1* image tagged in a previous step:

```output
Registry  ImageName        Tags
--------  ---------        ----
<acrName> azure-vote-front {v1}
```

---

You now have a container image that is stored in a private Azure Container Registry instance. This image is deployed from ACR to a Kubernetes cluster in the next tutorial.

## Next steps

In this tutorial, you created an Azure Container Registry and pushed an image for use in an AKS cluster. You learned how to:

> [!div class="checklist"]
> * Create an Azure Container Registry (ACR) instance
> * Tag a container image for ACR
> * Upload the image to ACR
> * View images in your registry

Advance to the next tutorial to learn how to deploy a Kubernetes cluster in Azure.

> [!div class="nextstepaction"]
> [Deploy Kubernetes cluster][aks-tutorial-deploy-cluster]

<!-- LINKS - external -->
[docker-images]: https://docs.docker.com/engine/reference/commandline/images/
[docker-push]: https://docs.docker.com/engine/reference/commandline/push/

<!-- LINKS - internal -->
[az-acr-create]: /cli/azure/acr#az_acr_create
[az-acr-list]: /cli/azure/acr#az_acr_list
[az-acr-login]: /cli/azure/acr#az_acr_login
[az-acr-repository-list]: /cli/azure/acr/repository#az_acr_repository_list
[az-acr-repository-show-tags]: /cli/azure/acr/repository#az_acr_repository_show_tags
[az-group-create]: /cli/azure/group#az_group_create
[azure-cli-install]: /cli/azure/install-azure-cli
[aks-tutorial-deploy-cluster]: ./tutorial-kubernetes-deploy-cluster.md
[aks-tutorial-prepare-app]: ./tutorial-kubernetes-prepare-app.md
[azure-powershell-install]: /powershell/azure/install-az-ps
[new-azresourcegroup]: /powershell/module/az.resources/new-azresourcegroup
[new-azcontainerregistry]: /powershell/module/az.containerregistry/new-azcontainerregistry
[connect-azcontainerregistry]: /powershell/module/az.containerregistry/connect-azcontainerregistry
[get-azcontainerregistry]: /powershell/module/az.containerregistry/get-azcontainerregistry
[get-azcontainerregistrymanifest]: /powershell/module/az.containerregistry/get-azcontainerregistrymanifest
[get-azcontainerregistrytag]: /powershell/module/az.containerregistry/get-azcontainerregistrytag
