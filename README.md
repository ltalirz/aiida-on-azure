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

With the storage account created one can then create a container, a container is the place where the data will actually be stored, this is what will be connected to the VM using the [blobfuse](https://github.com/Azure/azure-storage-fuse) application. **Important** when mounting `blobfuse` one can either give permission to the files only to the user mounting the `blobfuse` or to all the users, i.e all users would have access to **all** folders/files present in this container, if one wants to keep data strictly separated for each user, it is advisable to create a container for each user. 

A container is then created via
```
az ad signed-in-user show --query objectId -o tsv | az role assignment create \
    --role "Storage Blob Data Contributor" \
    --assignee @- \
    --scope "/subscriptions/<subscription>/resourceGroups/<resource_group_name>/providers/Microsoft.Storage/storageAccounts/<storage_account_name>"

az storage container create \
    --account-name <storage_account_name> \
    --name <container_name> \
    --auth-mode login
```

Where the first command gives the user the capability of generating the container and the second creates the container itself.


### Deployment/configuration of the **aiida-machine**
This machine will host the CycleCloud server which will be used to spawn the **compute-cluster**. This can be done in several ways, either using the [Azure marketplace](https://docs.microsoft.com/en-us/azure/cyclecloud/qs-install-marketplace?view=cyclecloud-8), an [ARM template](https://docs.microsoft.com/en-us/azure/cyclecloud/how-to/install-arm?view=cyclecloud-8), using a [container](https://docs.microsoft.com/en-us/azure/cyclecloud/how-to/run-in-container?view=cyclecloud-8) or via [manual installation](https://docs.microsoft.com/en-us/azure/cyclecloud/how-to/install-manual?view=cyclecloud-8). In this guide the manual installation si followed.

The first thing to do is to [deploy a VM](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-portal) in the resource group that has been created. The configuration of the machine will depend on the number of users and applications that will be deployed. It is important to consider that this machine will be always on, so it is important to strike the right balance of performance and cost. The VM should have a Linux OS, for simplicity Ubutnu will be used for this example, though the procedure is similar for RHEL.

One can [create the VM using an SSH key](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-portal#create-virtual-machine), of this way one can connect using a RSA key. One can improve the security by requiring that all users connect [only using RSA keys](https://askubuntu.com/questions/346857/how-do-i-force-ssh-to-only-allow-users-with-a-key-to-log-in). It is also recommended to setup [fail2ban](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-20-04), to try to protect against brute force attacks as much as possible.

The public IP address of the machine can be [static](https://docs.microsoft.com/en-us/azure/virtual-network/ip-services/virtual-networks-static-private-ip-arm-pportal#change-private-ip-address-to-static) and a [DNS label](https://docs.microsoft.com/en-us/azure/virtual-machines/create-fqdn) can be given. Though not strictly necessary (in contrast to the head node of the **compute-cluster**) it might be useful to ease the connection to the VM.

Once the machine is created, and the connectivity is solved (SSH keys, fail2ban, DNS label, etc.) one should update the VM `sudo apt-get update && sudo apt-get upgrade` to ensure that the latest security patches and updates are applied.

#### Install CycleCloud
The next step is to [install CycleCloud](https://docs.microsoft.com/en-us/azure/cyclecloud/how-to/install-manual?view=cyclecloud-8), for which one first needs to install a couple of dependencies:
```
sudo apt update && sudo apt -y install wget gnupg2
```
These will allow one to download the necessary trusted keys from MS to install CycleCloud using `apt-get`
```
wget -qO - https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
sudo echo 'deb https://packages.microsoft.com/repos/cyclecloud bionic main' > /etc/apt/sources.list.d/cyclecloud.list
sudo apt update
sudo apt -y install cyclecloud8
```

Once that is done all the pieces that comprise CycleCloud (sans the CLI) will be installed in the VM, however, they need to be configured so that they work properly. This must be done via a web browser. If the CycleCloud server can be accessed from the public IP, one can use the browser from the local machine, otherwise one needs to install a browser in the VM, in the following `firefox` will be used:
```
sudo apt-get install firefox
```
after that one can access the browser in the local via
```
firefox -url=http://cycle_coud_domain_name:8080 -no-remote
```
after that one must [configure the server](https://docs.microsoft.com/en-us/azure/cyclecloud/how-to/install-manual?view=cyclecloud-8#configuration). This is done by setting an administrator user account, with a password and ssh-key.

Inside the CycleCloud web application one can [configure](https://docs.microsoft.com/en-us/azure/cyclecloud/concepts/user-management?view=cyclecloud-8) which users have access and level of access to the CycleCloud clusters. It is recommended that one setups the ssh keys of each user here, since of that way once the cluster is deployed the users will be able to login to the head node using ssh.

One should also configure the CycleCloud application to use a [service principal](https://docs.microsoft.com/en-us/azure/cyclecloud/how-to/service-principals?view=cyclecloud-8#permissions), this will allow the CycleCloud application to create nodes inside the resource group (or other resource groups).

Lastly one can [install the CycleCloud CLI](https://docs.microsoft.com/en-us/azure/cyclecloud/how-to/install-cyclecloud-cli?view=cyclecloud-8), this can be done either via the web application, or by downloading the installer from the terminal

```
wget https://<cycle_coud_domain_name>/static/tools/cyclecloud-cli.zip
```
with the installer one can then extract the installer to a temporary folder and run the provided installation script

```
cd /tmp
unzip /opt/cycle_server/tools/cyclecloud-cli.zip
cd /tmp/cyclecloud-cli-installer
./install.sh
```

After this CycleCloud should be configured so that one can start deploying clusters.

#### Setup `blobfuse`

Setting up `blobfuse` can be useful as it will help to easily transfer data between the **aiida-machine** and the **compute-cluster**. Depending on how one setups the `blob` storage it can also be one of the most cost-effective long-term storage solutions for data generated from simulations.
```
wget https://packages.microsoft.com/config/ubuntu/<ubuntu version 16.04 or 18.04 or 20.04>/packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt-get update
sudo apt-get install blobfuse fuse
```

**Important** As of the 17th of July 2022 a deb package is not provided for Ubuntu 22.04, however, one can build blobfuse [from source](https://github.com/Azure/azure-storage-fuse/wiki/1.-Installation#option-2---build-from-source), and it will work. However, use at your own risk as it is not officially supported.

Once `blobfuse` is installed one must **mount** a container. Bear in mind, that `blobfuse` is a process, that can go down for many reasons, such as connectivity issues, lack of resources, etc. To mount the container one must first create a folder where the cache will be stored, it is recommended to always use the fastest drive possible to obtain better performance
```
sudo mkdir /mnt/blobfusetmp
sudo chown <youruser> /mnt/blobfusetmp
```

After this one can mount the container via

```
blobfuse path_where_to_mount --tmp-path=/mnt/blobfusetmp --use-attr-cache=true -o attr_timeout=240 -o entry_timeout=240 -o negative_timeout=120 --config-file=path_to_config/connection.cfg
```

where the `connection.cfg` is a file with the configuration for the connection, i.e. this is where the Azure secrets, container name, etc. are located **keep this file secret**.

```
accountName <account-name-here>
# Please provide either an account key or a SAS token, and delete the other line.
accountKey <account-key-here-delete-next-line>
#change authType to specify only 1
sasToken <shared-access-token-here-delete-previous-line>
authType <MSI/SAS/SPN/Key/empty>
containerName <insert-container-name-here>
#if you are using a proxy server and https, https is the default protocol set the caCertFile below
caCertFile <insert the certfile name with full path>
#if you are using a proxy server and https protocol which is the default protocol set the httpsProxy below 
httpsProxy <insert the https proxy server Eg: http://10.1.0.23:8080>
#if you are using proxy server and have turned https off using --use-https=false, set the httpProxy below
httpProxy <insert the http proxy server if any if you have turned https off using --use-https=true>
```

If `blobfuse` were to fail, or if one wants to stop the process, one can use `fusermount -u path_where_to_mount` to unmount the disk.

#### Install `AiiDA`

## Contribute

Suggestions for improvements and bug fixes are highly welcome.
Just open an issue or a pull request.
