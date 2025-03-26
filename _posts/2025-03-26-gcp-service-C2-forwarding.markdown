---
layout: post
title:  "GCP Service Command and Control HTTP traffic forwarding"
date:   2025-03-26 20:06:00 +1100
author: Stephen Bradshaw
tags:
- GCP
- Cloud Run
- Google App Engine
- Serverless
- C2
- domain fronting
- forwarding
- container
- sliver
---


<p align="center">
  <img src="/assets/img/gcp_hacker.png" alt="GCP Hacker">
</p>


Back in 2023 I wrote a [post on (ab)using various AWS services](/2023/08/30/aws-service-C2-forwarding.html) to provide fronting for C2 services. Ive been planning for a while to try and do the same thing in other well known cloud providers, and Ive finally gotten around to looking at this in GCP. This post will cover how we can front C2 servers using the [Google App Engine](https://cloud.google.com/appengine) and [Cloud Run](https://cloud.google.com/run) Google Cloud services.

# What is the purpose of C2 fronting?

When I wrote my [AWS post on C2 fronting](/2023/08/30/aws-service-C2-forwarding.html) I didnt really provide a good explanation of the value of this idea, so I thought it might be worthwhile to do it now.

The idea for this was first (to my knowledge) discussed back in 2017 in a post from the Cobalt Strike blog which can be found [here](https://www.cobaltstrike.com/blog/high-reputation-redirectors-and-domain-fronting). This post talks talks about two different concepts, domain fronting and high reputation redirectors. As discussed in [my AWS post](/2023/08/30/aws-service-C2-forwarding.html#a-note-on-domain-fronting-and-cloudfront), domain fronting no longer works well in practice in many modern CDNs, but the high reputation redirector concept still has value.

What do we mean by high reputation redirectors and why would we want to use them?

Consider this basic architecture diagram of a C2 system which uses HTTP for implant communications.

<p align="center">
  <img src="/assets/img/c2_architecture_basic.png" alt="Basic C2 Architecture">
</p>

In this case we are using a single C2 Internet accessible host. We as an attacker compromise victim systems (shown on the left) and install our implant software, and these connect back to our C2 server via HTTP/S to request tasking. This tasking could involve running other executables, uploading files or data from the victim system or more. Operators of the C2 system (shown on the right), connect to the C2 via a seperate admin communication channel, and provide tasking through the C2 for the victim systems via this channel.

In this simple type of design the implant endpoint of the C2 is usually given a DNS name that victim hosts will connect to. In this case the DNS name is `www.evil.com`. Given that implant traffic will need to bypass any network security controls used by the victim system, choosing the right endpoint address can impact on the liklihood of getting a successful connection. 

In the case of the high reputation redirectors concept, what we are looking to do is take advantage of the good reputation of the domain name of the redirector service to avoid our implant traffic being blocked by services which check the reputation of the DNS domain to which we are connecting. Instead of some newly registered dodgy looking domain like `www.evil.com` we instead connect to the domain used by the forwarding service. This forwarding service is widely used with the vast majority of traffic being legitimate, and are often highly trusted by security solutions.

Heres a more defensively designed C2 system design, using a high reputation cloud service for fronting.

<p align="center">
  <img src="/assets/img/c2_architecture_basic_fronted.png" alt="Basic C2 Architecture with Cloud Service Fronting">
</p>

Here, the basic design has been modified by instead hosting the C2 service in a private VPC, no longer directly exposed to the Internet, and the implant traffic instead can be directed to the C2 service by the use of an [AWS Cloudfront](https://aws.amazon.com/cloudfront/) distribution, which has not had a custom domain associated with it. So, instead of the implants having to connect to the suspicious looking custom address of `www.evil.com`, the implants instead connect to the much more trustworthy address `abcdefghijklm.cloudfront.net` that the CloudFront service auto generates for us when we create a new distribution. We even get a SSL certificate for HTTPS traffic included!

Now lets look at how we can achieve the same, but with GCP services.

# Previous work

As with the previous AWS post, I wanted to see what other people had done in this area. 

My goal was to find native GCP services that allowed forwarding of generic HTTP/S C2 implant traffic. While there are C2 implants that support all sorts of different communications methods such as [Slack](https://www.praetorian.com/blog/using-slack-as-c2-channel-mitre-attack-web-service-t1102/), [Discord](https://securityintelligence.com/x-force/self-checkout-discord-c2/), [GitHub](https://github.com/MythicC2Profiles/github) and more, I was interested only in services that would allow support for the greatest breadth of C2 implants. This means the service had to allow more or less direct proxying/forwarding of HTTP 1.1 requests, with support for at least GET and POST requests.

Unlike with AWS however, I wasn't able to find very many examples of other people abusing GCP for fronting C2 in this way. Maybe Im bad at searching, but the only example I found was [this post from 2017 talking about using Google App Engine for C2 fronting](https://www.securityartwork.es/2017/01/31/simple-domain-fronting-poc-with-gae-c2-server/). Even this post does not provide a fully working example configuration that works for forwarding to generic C2 servers. It did provide enough of a starting point/hint however that I was able to create a working Python POC application that allows generic C2 forwarding using either [Google App Engine](https://cloud.google.com/appengine?hl=en) or [Cloud Run](https://cloud.google.com/run?hl=en).


# GCP C2 POC environment 

Common to both of the fronting configurations was a shared environment comprised of a GCP project `LegitApp` (project id `legitapp-24002`) that had a compute engine VM `instance-20250312-010048` running in region `us-central1-c` with the [Sliver](https://github.com/BishopFox/sliver) C2 server installed and a HTTP listener configured on port 80. The `default` VPC and subnets were used for the instance. Internal IP address of the instance was `10.128.0.2`.

The steps below involve extensive use of the gcloud CLI, which was [installed and configured as per these instructions](https://cloud.google.com/sdk/docs/install).

Access to the C2 instance was via gloud initiated ssh, via a command similar to the following. This is the approach I used to both install the Sliver C2 software and connect to Sliver as an operator.

```
gcloud compute ssh --zone "us-central1-c" "instance-20250312-010048" --project "legitapp-24002"
```

Setting the default project in the gcloud cli can also be necessary. This is done like so:

```
gcloud config set project legitapp-24002
```

Now we can look at the specifics of setting up each type of forwarder.


# Google App Engine

Theres a bit more clickops required to configure this POC. Read through the whole thing before attempting.

The code you will need is [here](https://github.com/stephenbradshaw/GCPAppEngineC2Forwarder).

Setup a [Serverless VPC connector](https://cloud.google.com/vpc/docs/serverless-vpc-access) in your VPC with an unused IP address range in the same region as your App Engine App and VM instance. This is done in the console [here](https://console.cloud.google.com/networking/connectors/list). This is required to allow traffic from the App Engine App to the private IP address of your C2 instance - so you dont need to directly expose the HTTP interface of the C2/proxy to the Internet. Take note of the name of the connector as we need it when we configure the app before deployment - I called mine `testconnector`. Be aware there is an ongoing cost associated with running this connector - I deleted mine immediately after my POC testing was validated.  


Then, setup a [firewall rule](https://console.cloud.google.com/net-security/firewall-manager/firewall-policies/list) for the VPC which allows traffic from the VPC connectors private IP address range to port 80 on our C2 host. I handled this by creating a rule to allow all internal traffic from  the range `10.0.0.0/8` (which included the IP range assigned to my `testconnector` VPC connector) to TCP port 80.


Follow the [instructions](https://cloud.google.com/appengine/docs/standard/python3/building-app/creating-gcp-project) to create a Python GCP project. I created a `Standard` app in the `us-central` region. This will also involve enabling the required APIs using the web console if you have not already done this.


Now you need to configure a few settings in `app.yml` before deploying the app.

First, set the private IP address of the GCP C2 compute instance target VM (`10.128.0.2`) in the following location. Add a port designation to the address as well (e.g. 10.128.0.2:8080) if you have changed the listener  port to something different.

```
env_variables:
  DESTINATION: "10.128.0.2"
```

Now we need to configure the app to use the Serverless VPC connector we created. Modify the following line, replacing the <PROJECT_ID>, <REGION> and <CONNECTOR_NAME> placeholders with the appropriate values.

```
vpc_access_connector:
  name: projects/legitapp-24002/locations/us-central1/connectors/testconnector
  egress_setting: private-ranges-only
```


Now we can deploy the app. Run the following from the root directory of the source files.

```
gcloud app deploy
```

After this point my app was listening at the address `https://legitapp-24002.uc.r.appspot.com`, ready to forward traffic to my C2.

The architecture looks like the following.

<p align="center">
  <img src="/assets/img/c2_architecture_basic_gae.png" alt="C2 with GAE Fronting">
</p>


Clean up instructions to get rid of GAE apps are [here](https://cloud.google.com/appengine/docs/standard/python3/building-app/cleaning-up), but be warned, its kind of a pain.


# Google Cloud Run

The code you will need for the Cloud Run POC is [here](https://github.com/stephenbradshaw/GCPCloudRunC2Forwarder). Its very similar to the previous code repository just with different supporting files for the main Python app.

The Cloud Run service is a newer GCP serverlesss computing than GAE and this is readily apparent in the difference in ease of use, flexibility and managability between the two services. Cloud Run essentially launches your app in a container that you can connect to an existing VPC subnet, which makes things much easier and much more flexible when it comes to routing traffic, you are also able to easily launch multiple Run apps in a single project, and the apps are much easier to delete the apps when you are done with them. All things considered, Cloud Run is much preferable to GAE for this use case, so its what I recommend.

We can use [Direct VPC egress](https://cloud.google.com/run/docs/configuring/vpc-direct-vpc) to allow traffic to flow direct from the Cloud Run container by running the container within the same (or an adjacent) subnet as your destination C2 GCP compute VM. You then need to create firewall rules to allow traffic to the port on the destination host. The [firewall rule](https://console.cloud.google.com/net-security/firewall-manager/firewall-policies/list) I mentioned in the GAE section that allows all internal traffic to port 80 from range `10.0.0.0/8` will work for this if you run the Cloud Run container in the same subnet as the C2 VM instance.


Heres how we deploy the service.  First enable the needed APIs.

```
gcloud services enable run.googleapis.com
gcloud services enable cloudbuild.googleapis.com
```


Now we can deploy the app. All the necessary configuration can be done at deploy time. In the following case, my app name is `toteslegit`, I configure the `DESTINATION` environment variable with our C2 instance IP address `10.128.0.2`, set the appropriate region `us-central1` and specify that the container should run in my instances subnet, `projects/legitapp-24002/regions/us-central1/subnetworks/default`.

Run the following from the root directory of the source files.

```
gcloud run deploy toteslegit --source . --set-env-vars='DESTINATION=10.128.0.2' --region=us-central1 --subnet='projects/legitapp-24002/regions/us-central1/subnetworks/default'
```

After deployment, the app is running at `https://toteslegit-983481224768.us-central1.run.app`

The architecture looks like the following.

<p align="center">
  <img src="/assets/img/c2_architecture_basic_cr.png" alt="C2 with Cloud Run Fronting">
</p>


# Conclusion

Im intending to keep investigating GCP to see if there are any more ways to use available services to forward for C2, and if I find more I'll do a follow up post to discuss them. Are you aware of any? Let me know!

Also, I will mention at this point, its your responsibility to make sure your use of the GCP platform is in line with the [AUP](https://cloud.google.com/terms/aup?hl=en) and [TOS](https://cloud.google.com/terms/)