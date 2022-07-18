# AiiDA on Azure

This is a step-by-step guide for setting up a computational environment using the [AiiDA workflow manager](https://www.aiida.net/) on the [Microsoft Azure](https://azure.microsoft.com/en-us/) cloud.

The following guide will be made with the following architecture in mind:

* A always on VM which will host the AiiDA installation and other services needed. This will be called the **aiida-machine**.
* A [CycleCloud](https://docs.microsoft.com/en-us/azure/cyclecloud/overview?view=cyclecloud-8) cluster where the calculations will be performed. This will be called the **compute-cluster**.

_Disclaimer: this guide is under active development and offered without warranty of any kind. Use at your own risk._

## Features

- Azure CycleCloud with autoscaling cluster for compute
- NFS for sharing file sharing
- Azure blobstorage for backup
- AiiDA on standalone virtual machine, connected to the CycleCloud

## Steps

Broadly speaking there are four steps that need to be performed:
1. [Configuration of Azure account/services](#configuration-of-azure-accountservices).
2. [Deployment/configuration of the **aiida-machine** with CycleCloud](#deploymentconfiguration-of-the-aiida-machine).
3. [Installation of AiiDA](#installation-of-aiida).
4. [Configuration/deployment of the **compute-cluster**](#configurationdeployment-of-the-compute-cluster).

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

### Installation of `AiiDA`
[Installing `AiiDA`](https://aiida.readthedocs.io/projects/aiida-core/en/latest/intro/install_system.html#intro-get-started-system-wide-install) in an Azure VM is performed in the same way as one would in a local machine. One should install it using a virtual environment to ensure that there are no conflicts with dependencies. It can be either via `virtualenv` or `conda`.

First one should install the prerequisites 
```
sudo apt install git python3-dev python3-pip postgresql postgresql-server-dev-all postgresql-client rabbitmq-server
```
**Important:** CycleCloud will use port `5672` the same default port from RabbitMQ, thus when RabbitMQ is installed it will instead probably use port `5673`. To check which exact port is being used one can run the command
```
sudo lsof -i -P -n | grep 'rabbitmq'
```

Once the pre-requisites have been installed one can install aiida-core using `pip`  in a virtual environment via
```
python -m venv ~/envs/aiida
source ~/envs/aiida/bin/activate
pip install aiida-core
```

Lastly, one needs to configure an aiida profile. To ensure that all the configuration is performed properly (mostly due to the possible issues with RabbitMQ) [`verdi setup`](https://aiida.readthedocs.io/projects/aiida-core/en/latest/intro/installation.html) is preferred over `verdi quicksetup`.

When setting up the RabbitMQ configuration via `verdi setup` one must ensure that the proper port (not the default one) is passed to the [configuration](https://aiida.readthedocs.io/projects/aiida-core/en/latest/intro/installation.html#rabbitmq-configuration).

With `blobfuse` being ready and `aiida` being installed one can setup the [backup](https://aiida.readthedocs.io/projects/aiida-core/en/latest/howto/installation.html#backing-up-your-installation) so that the data is stored in the blob. Of this way one would make sure that in the case of the failure of the VM the data is stored in as redundant manner as possible.

### Configuration/deployment of the **compute-cluster**

When creating a **compute-cluster** in CycleCloud the first thing that one needs to select is which kind of scheduler one wishes to use. CycleCloud supports a variety of schedulers such as [PBSPro](https://github.com/Azure/cyclecloud-pbspro), [GridEngine](https://github.com/Azure/cyclecloud-gridengine) and [SLURM](https://github.com/Azure/cyclecloud-slurm). Any of these defaults clusters provided with CycleCloud can be used to provision a cluster that can be used in combination with AiiDA with just small modifications. The possible options and defaults for a given cluster can be defined via a `template` file. The best way to generate a customized cluster is to take one of the templates found in the Azure github, modify it and upload it to the CycleCloud server so that it can be easily deployed.

For using the **compute-clusters** with AiiDA, one can provision one of the default clusters as they are and afterwards set a [static public IP](https://docs.microsoft.com/en-us/azure/virtual-network/ip-services/virtual-networks-static-private-ip-arm-pportal#change-private-ip-address-to-static) and a [DNS label](https://docs.microsoft.com/en-us/azure/virtual-machines/create-fqdn), these are necessary, if one does not does this the IP address of the head node will vary, which means that one would have to define a different computer every time the IP changes.

If one uses the pre-defined clusters they come with one of the Azure [HPC images](https://docs.microsoft.com/en-us/azure/virtual-machines/workloads/hpc/configure) which come with several compilers and libraries pre-installed which would allow the user to install most of the simulation software required. They also come with the [modules](https://modules.readthedocs.io/en/stable/index.html) package to handle the environment variables that different software packages might require.

The pre-defined clusters allow the users the capability of [adding NFS](https://docs.microsoft.com/en-us/azure/cyclecloud/how-to/mount-fileserver?view=cyclecloud-8) disks to the cluster. This is the place where the users `${HOME}` folders will be mounted, it is important to notice that this disk will be persistent both for the head node and the compute nodes, hence it is also a good place to store the simulation code that will be used. If one has found a particular NFS configuration that will be used for any cluster one can define it in the cluster `template` file. For example the size of the default `shared` filesystem can be defined as
```
    [[[volume shared]]]
    Size = 1024
    SSD = True
    Mount = builtinshared
    Persistent = ${NFSType == "Builtin"}

```

A similar modification can be performed to ensure that the **compute-cluster** has a pre-defined static IP address and DNS label. First one needs to create a static public IP address using the [Azure CLI](https://docs.microsoft.com/en-us/azure/virtual-network/ip-services/create-public-ip-cli?tabs=create-public-ip-standard%2Ccreate-public-ip-zonal%2Crouting-preference#create-a-resource-group)
```
az network public-ip create \
    --resource-group <resource_group_name> \
    --name <public_static_ip_name> \
    --version IPv4 \
    --sku Standard \
    --zone 1 2 3
```

Once the IP address has been created one can add it to the configuration of the cluster, of this way the cluster is ready to be used in conjunction with AiiDA

```
[[node scheduler]]
[[[network-interface eth0]]]
    PublicIp = /subscriptions/${subscription_id/resourceGroups/${resource_group_name}/providers/Microsoft.Network/publicIPAddresses/${public_static_ip_name}
    AssociatePublicIpAddress = true
    PublicDnsLabel = myuniquename
```

One can also define different types of calculation nodes, of that way one can handle jobs with different resource requirements, for this one can define different [nodearrays](https://docs.microsoft.com/en-us/azure/cyclecloud/cluster-references/node-nodearray-reference?view=cyclecloud-8), with different configurations
```
    [[node scheduler]]
    MachineType = $SchedulerMachineType
    ImageName = $SchedulerImageName
    IsReturnProxy = $ReturnProxy
    AdditionalClusterInitSpecs = $SchedulerClusterInitSpecs

    [[nodearray hpc]]
    MachineType = $HPCMachineType
    ImageName = $HPCImageName
    MaxCoreCount = $MaxHPCCoreCount

    Azure.MaxScalesetSize = $MaxScalesetSize
    AdditionalClusterInitSpecs = $HPCClusterInitSpecs

```

Where one can then define the default values of the different types of nodes
```
    [[parameters Virtual Machines ]]
    Description = "The cluster, in this case, has two roles: the scheduler with shared filer and the execute hosts. Configure which VM types to use based on the requirements of your application."
    Order = 20

        [[[parameter Region]]]
        Label = Region
        Description = Deployment Location
        ParameterType = Cloud.Region

        [[[parameter SchedulerMachineType]]]
        Label = Scheduler VM Type
        Description = The VM type for scheduler and shared filer.
        ParameterType = Cloud.MachineType
        DefaultValue = Standard_B4ms

        [[[parameter HPCMachineType]]]
        Label = HPC VM Type
        Description = The VM type for HPC nodes
        ParameterType = Cloud.MachineType
        DefaultValue = Standard_H16r
        Config.Multiselect = true
```
one can add as many different types of nodes as desired, and these can then be accessed by the different schedulers when submitting a calculation.

After one has modified the `template` file one can upload it to the CycleCloud application via the CycleCloud CLI. **Important** be sure to give this template its own name to avoid conflicts with previous templates that can have the same name.
```
cyclecloud import_template -c path_to_template/template.txt
```

After this the cluster can be provisioned via the CycleCloud application.

## Contribute

Suggestions for improvements and bug fixes are highly welcome.
Just open an issue or a pull request.
