# AiiDA on Azure

This is a step-by-step guide for setting up a computational environment using the [AiiDA workflow manager](https://www.aiida.net/) on the [Microsoft Azure](https://azure.microsoft.com/en-us/) cloud.

_Disclaimer: this guide is under active development and offered without warranty of any kind. Use at your own risk._

## Features

- Azure CycleCloud with autoscaling SLURM cluster for compute
- NFS for sharing file sharing  
- Azure blobstorage for backup
- AiiDA on standalone virtual machine, connected to the CycleCloud
- ...

## Steps

1. Get an Azure account
2. Create Azure CycleCloud from ARM template
3. Connect AiiDA VM
4. Run your first workflow

## Contribute

Suggestions for improvements and bug fixes are highly welcome.
Just open an issue or a pull request.
