# AiiDA on Azure

This is a step-by-step guide for setting up a computational environment using the [AiiDA workflow manager](https://www.aiida.net/) on the [Microsoft Azure](https://azure.microsoft.com/en-us/) cloud.

The following guide will be made with the following architecture in mind:

* A always on VM which will host the AiiDA installation and other services needed. This will be called the **aiida-machine**.
* A [CycleCloud](https://docs.microsoft.com/en-us/azure/cyclecloud/overview?view=cyclecloud-8) cluster where the calculations will be performed. This will be called the **compute-cluster**.

_Disclaimer: this guide is under active development and offered without warranty of any kind. Use at your own risk._

## Features

- Azure CycleCloud with autoscaling SLURM cluster for compute
- NFS for sharing file sharing  
- Azure blobstorage for backup
- AiiDA on standalone virtual machine, connected to the CycleCloud
- ...

## Steps

Broadly speaking there are four steps that need to be performed:
1. Configuration of Azure account/services.
2. Deployment/configuration of the **aiida-machine** with CycleCloud.
3. Configuration/deployment of the **compute-cluster**.
4. Installation of AiiDA and configuration to work with the **compute-cluster**.

All of these steps assume that the user has an Azure account, with the appropriate permissions to [create resource groups](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal), and a [service principal](https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals#service-principal-object). The examples will show how many of the operations are performed making use of the Azure CLI, however, it is important to notice that all these operations can be also performed by the Azure portal or the Azure PowerShell.


### Configuration of Azure account/services
To try to keep everything organized it is best to create a [resource group](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal) where all the following services/resources will be placed. Such resource group can be created via the Azure CLI by
```
az group create --name <resource_group_name> --location <location_name>
```

Once that the resource group is created one will want to create a [storage account](https://docs.microsoft.com/en-us/azure/storage/common/storage-account-overview), this will give the user a centralized place where data can be stored, specially by making use of the [blob storage](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blobs-introduction). The storage account is created by the following command:

```
az storage account create --name <storage_account_name> --resource-group <resource_group_name> --location <location_name> --sku Standard_ZRS  --encryption-services blob
```
for simplicity the location of all the resources will be the same, however this is not necessary. **Important** not all resources are available in all regions, this is specially important for the **compute-cluster**, thus, check which [VM sizes](https://azure.microsoft.com/en-us/pricing/details/virtual-machines/linux/) one will want to use and choose the region accordingly.

With the storage account created one can then create a container


### Deployment/configuration of the **aiida-machine**



Azure CycleCloud is a service for the provisioning, management and auto-scaling of HPC clusters. It allows one to 

1. Get an Azure account
2. Create Azure CycleCloud from ARM template
3. Connect AiiDA VM
4. Run your first workflow

## Contribute

Suggestions for improvements and bug fixes are highly welcome.
Just open an issue or a pull request.
