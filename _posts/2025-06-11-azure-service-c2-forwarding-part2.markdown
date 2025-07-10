---
layout: post
title:  "Azure Service Command and Control HTTP traffic forwarding part 2"
date:   2025-06-25 16:55:00 +1100
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
- Azure Front Door
- Azure service fronting
---

<p align="center">
  <img src="/assets/img/azure_front_door.webp" alt="Azure Front Door">
</p>

Here is another entry in my ongoing series of posts where I find ways to abuse cloud native services for fronting C2 implant traffic. 

This time I will be discussing the second Azure specific approach using [Azure Front Door](https://learn.microsoft.com/en-us/azure/frontdoor/front-door-overview).

For the first Azure specific post, you can go [here](/2025/05/07/azure-service-C2-forwarding.html), and for all of the other posts, also covering AWS and GCP, you can go [here](/tags.html#domain-fronting).


# Overview

Azure Front Door is essentially a Content Delivery Network for other services, so it does need some sort of backend system to forward traffic to. It is possible to have this forward traffic to a public IP address of an Azure VM instance, but given I had already setup a public endpoint for receiving C2 implant traffic in the form of an [Azure Function App](/2025/05/07/azure-service-C2-forwarding.html), I decided to continue to use this for this POC to avoid having to publicly expose the HTTP port on this instance to the Internet and worrying about how to [secure the origin service](https://learn.microsoft.com/en-us/azure/frontdoor/origin-security?tabs=app-service-functions&pivots=front-door-standard-premium).

This means that in my POC, I am using the Front Door service to provide an additional/alternate way for implant traffic to reach my C2. It is possible to [secure this](https://learn.microsoft.com/en-us/azure/frontdoor/origin-security?tabs=app-service-functions&pivots=front-door-standard-premium) so that traffic can only come from the frontdoor but I selected to just allow traffic to both endpoints.

# Setup

The supporting infrastructure in the POC environment including an Azure VPC, Linux VM running the [Sliver C2](https://github.com/BishopFox/sliver) software and Azure Function App can be automatically deployed as discused in this [blog post](/2025/06/04/azure_c&c_poc_infra_deployment.html).

Then the Azure Front Door component can be deployed as per the instructions [here](https://github.com/stephenbradshaw/AzureC2PocDeployment?tab=readme-ov-file#creating-an-azure-front-door-instance-for-c2-fronting)


Heres an example of how to do the necessary deployment.


## Create the C2 VM and VPC resources

Create a ssh key to use when accessing the vm, if you dont already have one.

```
$ ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_azure
```

Make sure the [Azure CLI is installed and working](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-linux?pivots=apt).


Create the `C2VMRG` resource group.

```
$ az group create --name C2VMRG --location westus2
```

Clone the POC deployment templates locally
```
$ git clone https://github.com/stephenbradshaw/AzureC2PocDeployment.git
```


Modify the `base` deployment templates for your environment (which uses your Internet facing IP and public key data for access) and deploy the resources.
```
$ cd AzureC2PocDeployment/base
$ export PUBLIC_IP=$(curl ipinfo.io/ip)
$ sed -i "s/<YOUR_IP_ADDRESS_HERE>/$PUBLIC_IP/g" parameters.json
$ export PUBLIC_KEY=$(cat ~/.ssh/id_ed25519_azure.pub | sed 's/\//\\\//g')
$ sed -i "s/<YOUR_PUBLIC_KEY_HERE>/$PUBLIC_KEY/g" parameters.json
$ export DNS_LABEL=dnslabel
$ sed -i "s/<YOUR_DNS_LABEL_HERE>/$DNS_LABEL/g" parameters.json
$ az deployment group create --resource-group C2VMRG --template-file template.json --parameters @parameters.json
```

Once this step is done you should be able to view the resources in the Azure Portal in the [Resource Group section](https://portal.azure.com/#browse/resourcegroups) in the `C2VMRG` group.


You can also get the public IP address of your VM as follows:

```
$ az vm list-ip-addresses -g C2VMRG --query "[].virtualMachine[].network.publicIpAddresses[].ipAddress | [0]"
```

Add an ssh hosts file entry for your VM as follows, replacing `<PUBLIC_IP_ADDRESS>` with the output of the above.

```
Host azurevm
    hostname <PUBLIC_IP_ADDRESS>
    identityfile ~/.ssh/id_ed25519_azure
    user ubuntu
```


You should now be able to ssh into that instance once it has fully deployed - which will take a few minutes. Access will be limited to the Internet facing IP address of the machine you performed the deployment from using a Network Security Group deployed with the resources defined in the templates.

## Create the Function App for implant traffic forwarding

Now we can create the Function App resources. We need to pick a globally unique name the app, I will use `mytestfunctionxyz123`, you will need to use something else.

This is done as follows:

```
$ cd ../functionapp
$ export FUNCTION_APP_NAME=mytestfunctionxyz123
$ sed -i "s/<FUNCTION_APP_NAME_HERE>/$FUNCTION_APP_NAME/g" parameters.json
$ az deployment group create --resource-group C2VMRG --template-file template.json --parameters @parameters.json
```

With the Azure resources created for the App we then need to deploy the code that executes in the Function App. This is done by cloning the example code repo ande deploying it using the az cli similar to the following:

```
$ cd ../../
$ git clone https://github.com/stephenbradshaw/AzureFunctionC2Forwarder
$ cd AzureFunctionC2Forwarder
$ zip -r /tmp/dep.zip ./function_app.py ./host.json ./requirements.txt
$ az functionapp deployment source config-zip -g C2VMRG -n $FUNCTION_APP_NAME --src /tmp/dep.zip
```

Once this is successfully deployed, we should be able to make the following curl request and see the default HTML context that sits on the Apache forwarder configured on the Linux C2 VM - this will respond to any requests to the endpoint with any files locally present in the Apache web root and send any other requests to the Sliver C2 listener. 

```
$ curl https://$FUNCTION_APP_NAME.azurewebsites.net/
<html>
</html>
```

## Create the Azure Front Door as a secondary/alternate forwarding endpoint

Now we can create the Azure Front Door resources by changing back to our deployment template repo in the `azurefd` subdirectory and replacing some variables for our Function App name (`mytestfunctionxyz123` in my case) and our Front Door name (I am using `testfd` below, this value will form part of the URL with which you can access the  ). This looks similar to the following. 

```
$ cd ../AzureC2PocDeployment/azurefd
$ export FUNCTION_APP_NAME=mytestfunctionxyz123
$ sed -i "s/<FUNCTION_APP_NAME_HERE>/$FUNCTION_APP_NAME/g" parameters.json
$ export ENDPOINT_NAME=testfd
$ sed -i "s/<FRONT_DOOR_ENDPOINT_NAME_HERE>/$ENDPOINT_NAME/g" parameters.json
$ az deployment group create --resource-group C2VMRG --template-file template.json --parameters @parameters.json
```

Once done, the Azure resource explorer will show the resources in the `C2VMRG` resource group looking similar to the following.


<p align="center">
  <img src="/assets/img/C2VMRG_azurefd.png" alt="Azure infrastructure after ">
</p>


Although the output from the `az deployment` command above will include the hostname that is auto created for our Front Door instance, it is buried in a lot of other text, so its also useful to know that it can be queried as follows: 

```
az afd endpoint list -g C2VMRG --profile-name MyFrontDoor --query '[].hostName | [0]'
```

In my case, the address was `testfd-a5g3ahdybnd6a9cz.z01.azurefd.net`, and making the following curl request to the endpoint as shown below should retrieve the default contents from the Apache forwarding service as shown below.

```
$ curl https://testfd-a5g3ahdybnd6a9cz.z01.azurefd.net/
<html>
</html>
```

The network diagram of our POC C2 setup now looks as follows, with two listening endpoints provided by the Function App and the Front Door that can be used by implants to talk to the C2.

<p align="center">
  <img src="/assets/img/c2_architecture_basic_azure_fd.png" alt="C2 with Azure Function App and Front Door Fronting">
</p>


When you are done with the POC environment, you can easily delete the resource group and all associated resources as follows:

```
az group delete --name C2VMRG
```


# Conclusion

I intend to keep investigating Azure to see if there are any more ways to use their cloud services to forward for C2, and if I find more I'll do a follow up post to discuss them. Are you aware of any? Let me know!


