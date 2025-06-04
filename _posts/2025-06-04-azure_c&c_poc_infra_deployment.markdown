---
layout: post
title:  "Azure Command and Control POC Infrastructure Deployment"
date:   2025-06-04 18:06:00 +1100
author: Stephen Bradshaw
tags:
- Azure
- Azure functions
- Serverless
- C2
- domain fronting
- forwarding
- container
- sliver
---

<p align="center">
  <img src="/assets/img/azure-resource-manager.png" alt="Azure Resource Manager">
</p>


As Ive been playing with Azure on and off over the last few weeks trying to find native services abusable for C2 fronting, I've developed a need to quickly setup and tear down POC C2 environments so I can avoid being billed for cloud resources during idle periods.

To this end, Ive created some simple Azure Resource Manager templates that allow me to quickly setup the resources mentioned in my earlier post on Azure Function Apps [here](/2025/05/07/azure-service-C2-forwarding.html). I figured it was worthwhile to share these in case anyone else find them useful.

The templates are available in this [repository on GitHub](https://github.com/stephenbradshaw/AzureC2PocDeployment).

There are currently two template sets in the repository, seperated by folder with names indicated below:
* `base` - Creates the virtual network, C2 VM, related resources and a network security group with rules that allow internal network HTTP traffic and SSH access from a single IP address over the Internet
* `functionapp` - Creates the Function App resource and necessary configuration to allow it to route traffic to the C2 VM created by the `base` template

Both template sets create resources in a resource group named `C2VMRG`, which must be created before deployment, and which can be deleted to remove all contained resources.


# The base template

The base template can be used to create a supporting C2 server that runs in Azure and allows operator connections from a single IP address specified at deploy time over the Internet. No implant connectivity is allowed at this stage, this must be provided by additional configuration, such as by deploying a Function App using the `functionapp` template.

The Azure resources deployed look like the following.

<p align="center">
  <img src="/assets/img/C2VMRG.png" alt="Azure Resource Manager">
</p>


The network diagram is at this point very simple - only an operator can connect to the C2 server. This is performed over the Internet using SSH to the instances associated public IP address (the `C2VM-ip` resource). It is allowed from a single IP address belonging to the operator which must be defined at deploy time. 


<p align="center">
  <img src="/assets/img/c2_architecture_basic_azure_base.png" alt="Partial Network Diagram">
</p>


The C2VM is configured with an Apache forwarder listening on TCP port 80 that performs some simple filtering and forwards relevant traffic to port 8888 where a Sliver HTTP listener job is ready to receive incoming implant connections. The network security group however only allows TCP 80 traffic from the private network however, so we need to perform further steps for a working test C2 environment.


# The functionapp template

The functionapp template deploys additional Azure resources to the `C2VMRG` resource group, providing a way for implant traffic to enter the environment from the Internet. It depends on the resources created by the `base` template having been deployed first.

The Azure resources look like the following after deployment (assuming a chosen function app name of `mytestfunctionxyz123` - these names must be globally unique within Azure, pick a different name if you try this yourself).


<p align="center">
  <img src="/assets/img/C2VMRG_more.png" alt="Azure Resource Manager">
</p>


Now the network diagram looks as follows - compromised hosts that we want to control with our C2 now have a way to communicate to the C2 by the Function Apps Internet facing endpoint of `https://mytestfunctionxyz123.azurewebsites.net/` - this app will receive traffic and forward it to port 80 on the C2 VM, where it will be forwarded to the Sliver service after some simple filtering.

Now the network diagram of our environment looks like the following:

<p align="center">
  <img src="/assets/img/c2_architecture_basic_azure_fr1.png" alt="Full Network Diagram">
</p>


# Conclusion

If you want to give these templates a try to create your own testing C2 environment in Azure, more detailed deploy documentation is included in [the repository on GitHub](https://github.com/stephenbradshaw/AzureC2PocDeployment). Keep in mind that this configuration is only suitable for testing and development and would need to be modified and made more robust for any sort of real operational use, but even in that case would serve as a decent starting point. 

When you are done with testing, the resources created by the templates can all easily be removed by deleting the entire `C2VMRG` resource group in the [Azure Portal](https://portal.azure.com/#browse/resourcegroups).
