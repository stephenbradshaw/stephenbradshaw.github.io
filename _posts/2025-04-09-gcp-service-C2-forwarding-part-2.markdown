---
layout: post
title:  "GCP Service Command and Control HTTP traffic forwarding part 2"
date:   2025-04-09 18:40:00 +1100
author: Stephen Bradshaw
tags:
- GCP
- Cloud Run
- API Gateway
- Serverless
- C2
- domain fronting
- forwarding
- container
- sliver
- redirector
---

<p align="center">
  <img width="500" height="500" src="/assets/img/google-cloud-hacker.png" alt="Google Cloud Hacker">
</p>

This is part two in my series on GCP C2 implant traffic forwarding, where I will be talking about abusing the GCP API Gateway service for C2 implant connections.

The underlying concepts of C2 implant forwarding, and setup of the supporting GCP services and infrastructure (including C2 server) are covered in [part 1 of the series](/2025/03/26/gcp-service-C2-forwarding.html), so make sure you have read that first.

I also explored [abusing AWS services in a similar way in an older post](/2023/08/30/aws-service-C2-forwarding.html), so check that out if you are interested in doing something similar in AWS.


# GCP API Gateway

We can use the GCP API Gateway to provide an Internet reachable endpoint for certain other types of GCP services. What we will be doing in this example is using the API Gateway to forward traffic through to an already existing Cloud Run service, similar to the one we created as per the instructions in [part one here](/2025/03/26/gcp-service-C2-forwarding.html#google-cloud-run).

This uses files from a GitHub repository [here](https://github.com/stephenbradshaw/GCPCloudRunC2Forwarder), which includes files for both the Cloud Run and API Gateway configuration.

We will essentially be using the process for connectinv API Gateway to Cloud Run services as described in the GCP documentation [here](https://cloud.google.com/api-gateway/docs/get-started-cloud-run).

Following these instructions, use the Create Gateway wizard [available here](https://console.cloud.google.com/api-gateway/api) to create a new API, API Config and Gateway combination, the form will look like the following.


<p align="center">
  <img src="/assets/img/gateway_wizard.png" alt="GCP API Gateway Creation Form">
</p>


The **API** section is essentially a structure that groups the API Config and Gateway together. You need to configure two values in the above form, the display name and the API ID. I set both of these in my example to the value of `testapi`.

The **API Config** specifies the configuration of the API. We need to provide an API Spec file, a display name, and a service account. I used `gateway` as the display name, and the default service account.

For the API Spec file I used a template file `api_forward.yaml` I have added to the Github repository [here](https://github.com/stephenbradshaw/GCPCloudRunC2Forwarder/blob/main/api_forward.yaml). We need to modify two values in the template before uploading it, as follows:

* `API_ID`- change to the `API ID` of the API you create, e.g. `testapi` in my case.
* `APP_URL` - change to the URL of the Cloud Run service already created that we want to forward traffic to, e.g. in my case `https://toteslegit-983481224768.us-central1.run.app` was the address of the Cloud Run app I created in [part 1](https://thegreycorner.com/2025/03/26/gcp-service-C2-forwarding.html)

For the **Gateway** section, we need to set a display name (that will appear in the final URL reachable by implants), I used `gateway`, and a region in which to run the service, I chose `us-central1`.

After you hit **Create Gateway** after configuring these details, the associated services will be created after a few minutes, with a URL for the service in the form of `https://<GATEWAY_NAME>-<8_DIGIT_STRING>.<REGION_ID>.gateway.dev`.

The final URL will be accesible from within the Gateways section of your new API once its done deploying, mine was (similar to) `https://gateway-abcdefgh.uc.gateway.dev`.

The architecture of this approach looks like the following.

<p align="center">
  <img src="/assets/img/c2_architecture_basic_cr_apigw.png" alt="C2 with Cloud Run Fronting">
</p>


# Conclusion

I intend to keep investigating GCP to see if there are any more ways to use their cloud services to forward for C2, and if I find more I'll continue to do follow up posts. If you aware of any that I haven't already discussed, please let me know about them.

I will also mention at this point, its your responsibility to make sure your use of the GCP platform is in line with the [AUP](https://cloud.google.com/terms/aup?hl=en) and [TOS](https://cloud.google.com/terms/)

