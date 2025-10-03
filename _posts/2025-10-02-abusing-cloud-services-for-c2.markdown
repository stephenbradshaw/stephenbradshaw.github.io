---
layout: post
title:  "Abusing cloud services for command and control - AWS, Azure, GCP summary"
date:   2025-10-02 14:04:00 +1100
author: Stephen Bradshaw
tags:
- Serverless
- C2
- domain fronting
- forwarding
- sliver
- Azure service fronting
- AWS
- Azure
- GCP
- App Runner
- API Gateway
- Azure API Management Service
- Lambda
- CloudFront
- Amplify
- AppRunner
- Elastic Beanstalk
- Function Apps
- Function URL
- Cloud Run Functions
- Azure Service 
- Azure Front Door
- Azure Container Apps
- App Engine
- redirector
---

<p align="center">
  <img width="500" height="500" src="/assets/img/cloud_puppet_master.png" alt="Cloud Puppet Master">
</p>

# Introduction

Last weekend, I presented at [BSides Canberra](https://www.bsidesau.com.au/) on the topic of abusing native cloud services for Command and Control (C2). The talk discussed the concept of the high reputation redirector, why these are used in Command and Control designs, and listed a number of ways to create these redirectors using native cloud services from AWS, Azure and GCP. 

The slides from the presesntation are shared [here](https://github.com/stephenbradshaw/pentesting_stuff/blob/master/presentations/Abusing_Cloud_Services_for_C2_slides.pdf), and at some point in the near future the recording of the talk will be released on the [BSides Canberra YouTube channel](https://www.youtube.com/@bsidescanberra9688). I will update the post with a direct link to the talk once its available.

Given that I have been working on this subject since late 2023 and the various content I have written is scattered across multiple posts, I thought I would do a single summary post to organise everything and make specific information easier to find.

This post will explain the concept of using a high reputation redirector for C2, will list the various applicable cloud services that can be used, and will provide direct links to detailed posts where applicable. Some of the summary information below will be reproduced here from the original posts, so be aware you might see some repeated information from individual entries in the series.


# Why do we use high reputation redirectors in Command and Control designs?

To my knowledge, the idea for C2 fronting was first discussed back in 2017 in a post from the Cobalt Strike blog which can be found [here](https://www.cobaltstrike.com/blog/high-reputation-redirectors-and-domain-fronting). This post talks about two different concepts, domain fronting and high reputation redirectors. As discussed in the following section, domain fronting no longer works well in many modern CDNs, but the high reputation redirector concept still has value.

So, what do we mean by high reputation redirectors and why would we want to use them?

Consider this basic architecture diagram of a C2 system which uses HTTP/S for implant communications.

<p align="center">
  <img src="/assets/img/c2_architecture_basic.png" alt="Basic C2 Architecture">
</p>

In this case we are using a single C2 host accessible on the Internet. We as an attacker compromise victim systems (shown on the left) and install our implant software. This software connects back to our C2 server via HTTP/S to request tasking. This tasking could involve running other executables, uploading files or data from the victim system and more. Operators of the C2 system (shown on the right), connect to the C2 via a seperate admin communication channel to provide tasking through the C2 for the victim systems.

In this simple type of design the implant endpoint of the C2 is usually given a DNS name that victim hosts will connect to. In this case the DNS name is `www.evil.com`. Given that implant traffic will need to bypass any network security controls used by the victim system, choosing the right endpoint address can impact the liklihood of getting a successful connection. 

In the case of the high reputation redirectors concept, we are trying to take advantage of the good reputation of the redirector services domain name to avoid our implant traffic being blocked due to poor domain reputation. Instead of some newly registered dodgy looking domain like `www.evil.com`, we instead connect to the domain used by the forwarding service. This forwarding services are widely used for legitimate traffic, and are often trusted by security solutions.

Heres an example of a redesigned C2 system using a high reputation cloud service for fronting.

<p align="center">
  <img src="/assets/img/c2_architecture_basic_fronted.png" alt="Basic C2 Architecture with Cloud Service Fronting">
</p>

Here, the basic design has been modified to host the C2 service in a private VPC. The C2 server is no longer directly exposed to the Internet, and the implant traffic is directed to the C2 service by the use of a [AWS Cloudfront](https://aws.amazon.com/cloudfront/) distribution, which has not had a custom domain associated with it. So, instead of the implants having to connect to the suspicious looking address of `www.evil.com`, the implants instead connect to the much more trustworthy address `abcdefghijklm.cloudfront.net` that the CloudFront service auto generates for us when we create a new distribution. We even get a SSL certificate for HTTPS traffic included!


# What about domain fronting? An aside....

Back in 2023 when I first started looking into the high reputation redirector concept, I also investigated domain fronting given it was always referenced when anyone talked about using CloudFront with C2.  What follows in this section is reproduced from what I [wrote at the time](/2023/08/30/aws-service-C2-forwarding.html#a-note-on-domain-fronting-and-cloudfront) on the subject, specific to AWS services. Keep in mind I have not done any testing to update this information for GCP or Azure services or to account for changes since 2023. Consequently, while the details below about how domain fronting works are still accurate, specifics about whether or how it works with particular cloud services may have changed.

Domain fronting is a technique that attempts to hide the true destination of a HTTP request or redirect traffic to possibly restricted locations by abusing the HTTP routing capabilities of CDNs or certain other complex network environments. For version 1.1 of the protocol, HTTP involves a TCP connection being made to a destination server on a given IP address (normally associated with a domain name) and port, with additional TLS/SSL encryption support for the connection in HTTPS. Over this connection a structured plain text message is sent that requests a given resource and references a server in the `Host` header. Under normal circumstances the domain name associated with the TCP connection and the `Host` header in the HTTP message match.  In domain fronting, the destination for the TCP connection domain name is set to a site that you want to appear to be visiting, and the `Host` header in the HTTP request is set to the location you actually want to visit. Both locations must be served by the same CDN. 

The following curl command demonstrates in the simplest possible way how the approach is performed in suited environments. In the example, `http://fakesite.cloudfront.net/` is what you want to **appear** to be visiting, and `http://actualsite.cloudfront.net` is where you actually want to go:

```
curl -H 'Host: actualsite.cloudfront.net'  http://fakesite.cloudfront.net/
```

In this example, any DNS requests resolved on the client site are resolving the "fake" address, and packet captures will show the TCP traffic going to that fake systems IP address. If HTTPS is supported, and you use a `https://` URL, the actual destination you are visiting located in the HTTP `Host` header will also be hidden in the encrypted tunnel.

While this **is** a great way of hiding C2 traffic, due to a widespread practice of domain fronting being used to evade censorship restrictions, various CDNs did crack down on the approach a few years ago. Some changes were rolled back in some cases, but as at the time of my testing in 2023 this simple approach to domain fronting **does not work** in CloudFront for HTTPS. If the DNS hostname that you connect to does not match any of the certificates you have associated with your CloudFront distribution, you will get the following error:

```
The distribution does not match the certificate for which the HTTPS connection was established with.
```

This applies only to HTTPS - HTTP still works using the approach shown in the example above. However, given the fact that HTTP has the `Host` header value exposed in the clear in network traffic this leaves something to be desired when the purpose is hiding where you're going. Depending on the capability of inspection devices, it might be good enough for certain purposes however.

It is possible to make HTTPS domain fronting work on CloudFront via use of Server Name Indication [(SNI)](https://www.cloudflare.com/en-gb/learning/ssl/what-is-sni/) to specify a Server Name value during the TLS negotiation that matches a certificate in your Cloudfront distribution. In other words, you TCP connect via HTTPS to a fake site on the CDN and set the SNI servername for the TLS negotation **AND** the HTTP `Host` to your actual intended host.  

Heres how this connection looks using openssl.
```
openssl s_client  -quiet -connect fakesite.cloudfront.net:443 -servername actualsite.cloudfront.net < request.txt
depth=2 C = US, O = Amazon, CN = Amazon Root CA 1
verify return:1
depth=1 C = US, O = Amazon, CN = Amazon RSA 2048 M01
verify return:1
depth=0 CN = *.cloudfront.net
verify return:1
```

Where file `request.txt` contains something like the following:
```
GET / HTTP/1.1
Host: actualsite.cloudfront.net


```

Unfortunately, I'm not aware of any C2 implant that supports specifying the TLS servername in a manner similar to what is shown above, so C2 HTTPS domain fronting using CloudFront is not a viable approach until this time. However, this does not mean that CloudFront is completely unusable for C2. As already mentioned, you can do domain fronting via HTTP. Its also possible to access the distribution via HTTPS using the `<name>.cloudfront.net` name that is created randomly for you when you setup your distribution. This domain does have a good trust profile in some URL categorisation databases.

# Protection/filtering for the C2 servers HTTP endpoint

When I setup C2, I usually also add in a seperate HTTP proxy service before the C2s HTTP listener. This provides some additional protection to the C2 server from malicious access or attempts to fingerprint the service. The ideal design would have this proxy sitting on its own restricted dedicated bastion host. However, for the POC C2 architectures I have discussed in the majority of the blog posts in this series, I run this proxy on the same host running the C2 server to minimise cost and complexity. The proxy I use is an Apache instance listening on port 80, which will in turn forward _certain_ incoming HTTP traffic on to the C2 HTTP endpoint, which is listening on localhost port 8888.

A script for Ubuntu Linux that will set this up for you using the [Sliver](https://github.com/BishopFox/sliver) C2 server can be found [here](https://github.com/stephenbradshaw/pentesting_stuff/blob/master/setup_scripts/c2_setup.sh). This is the same script that is used in the [deployment templates that I created for the Azure entries in this series](/2025/06/04/azure_c&c_poc_infra_deployment.html).

The filtering in this design is pretty simple, configured in `/var/www/html/.htaccess` and works in the following manner:
* Any requests for any file that exists in the Apache web root (`/var/www/html`) will be served directly by Apache - there is a default index page, a robots.txt and a randomly named php script that exists for troubleshooting
* Any requests for any URL that does not exist in the Apache web root will be forwarded to the C2 endpoint _if_ it comes from a non-bot/automated/command-line user agent, otherwise the request will get a generic 404 response

This is not perfect, and can be refined to filter more stringently based on the HTTP communication profile of a particular C2, but it does help prevent against casual identification of the C2 or _certain_ direct attacks against its implant endpoint, as the robust and tested Apache server receives the web traffic first and does some response header modifications. The use of Apache also gives us an easy and well tested way of setting up a HTTPS listener for the C2 as well if we want this.

# What characteristics make a service suitable to be used as a high reputation redirector?

So what qualities do we look for in a cloud service that makes it a good candidate for use as a high reputation redirector for C2?

Lets define the goal as achieving functional HTTP/S communication for unmodified C2 software. While there are C2 solutions that support various custom communication channels such as [Slack](https://www.praetorian.com/blog/using-slack-as-c2-channel-mitre-attack-web-service-t1102/), [Discord](https://securityintelligence.com/x-force/self-checkout-discord-c2/), [GitHub](https://github.com/MythicC2Profiles/github) and more, I was interested in HTTP/S because its commonly used and very well supported by C2 software solutions. 

More specifically, the cloud service must:
* Provide the ability to proxy, redirect or forward HTTP/1.1 traffic to a configurable backend. This backend could be a service in a private VPC hosted in the cloud providers network, or a service accessible more generally over the Internet.
* Provide a trusted DNS name from the cloud provider. We dont want to have to provide our own custom domain name that we need to age and build trust for.
* Ideally, provide a trusted certificate for the cloud provider DNS domain, that will enable HTTPS for you without having to set it up yourself and have the domain appear in certificate transparency logs as a result of creating that certificate yourself using a service such as LetsEncrypt.
* Support at least the GET and POST HTTP methods.
* Support binary content in the HTTP message body for requests and responses.
* Support the ability to send requests and responses uncached.
* Retain any cookies set by the C2 HTTP server (through the Set-Cookie header).
* Maintain the URL path and query parameters for all forwarded requests as-is. Some consistent content included at the start of the URL path is fine (e.g. `/api/`), as most C2 servers can deal with this, as long as everything after this section of the path is sent with the same value as sent by the client.


# Previous work on using cloud services as high reputation redirectors

This section will list a few posts from others.

## AWS 

I found the following approaches discussed online for AWS services:
* [This post](https://blog.xpnsec.com/aws-lambda-redirector/) by Adam Chester which talks about using the [Serverless framework](https://serverless.com/) to create an [AWS Lambda](https://aws.amazon.com/lambda/) that will receive traffic from the [AWS API Gateway](https://aws.amazon.com/api-gateway/) and then forward it to another arbitrary destination. In Adams example, he forwards the traffic using ngrok to a local web server, but this approach could be used point to an EC2 instance in the same AWS account or other server on the Internet.
* [This post](https://scottctaylor12.github.io/lambda-function-urls.html) from Scott Taylor which again uses a Lambda to perform traffic redirection, but this time the entry point is via [Lambda function URLs](https://docs.aws.amazon.com/lambda/latest/dg/lambda-urls.html) instead of the API Gateway. There is associated code/instructions to deploy this Lambda to AWS as well as to create an EC2 instance to forward the traffic to.
* Many posts on using a [CloudFront distribution](https://aws.amazon.com/cloudfront/) and domain fronting to receive traffic from CloudFront sites and send them to a C2 server somewhere else. Some examples are [this](https://www.cobaltstrike.com/blog/high-reputation-redirectors-and-domain-fronting), [this](https://digi.ninja/blog/cloudfront_example.php), [this](https://www.mdsec.co.uk/2017/02/domain-fronting-via-cloudfront-alternate-domains/) and [this](https://www.blackhillsinfosec.com/using-cloudfront-to-relay-cobalt-strike-traffic/). 



## Azure

I found the following approaches discussed online for Azure services:
* A number of posts discuss using [Azure Function Apps](https://azure.microsoft.com/en-us/products/functions), including [this](https://web.archive.org/web/20201107235933/https://vincentyiu.com/red-team/attack-infrastructure/azure-apps-for-command-and-control) and [this](https://0xdarkvortex.dev/c2-infra-on-azure/)
*  [This previously mentioned post](https://0xdarkvortex.dev/c2-infra-on-azure/) also discusses using the (no longer operational) Azure Edge CDN (replaced by [Azure Front Door](https://azure.microsoft.com/en-us/products/cdn)) and the [Azure API Management Service](https://azure.microsoft.com/en-au/products/api-management)
* Finally we have [this post](https://rosesecurity.gitbook.io/red-teaming-ttps/guides/azure-static-web-application-c2-redirectors) which talks about using an [Azure Static Web App](https://azure.microsoft.com/en-us/products/app-service/static). The discussed approach however, involves a configuration which performs a HTTP 301 client side redirect to the backend C2 server. This unfortunately allows leakage of the C2 origin, something we dont want. The discussed Static Web App service also allows the ability to proxy directly to a backend API service, an idea that is not discussed in the post. This would ordinarily be a good way to provide a high reputation redirector, _however_ this API proxying approach does remove cookies set using the `Set-Cookie` header from the backend API service, meaning it will not work with certain C2 servers. This is why Azure Static Web Apps are not in my list of services for creating high reputation redirectors for C2.

## GCP

I wasn't able to find very many examples of other people abusing GCP for fronting C2 in this way. Perhaps its bad to my poor searching, but the only example I found was [this post from 2017 talking about using Google App Engine for C2 fronting](https://www.securityartwork.es/2017/01/31/simple-domain-fronting-poc-with-gae-c2-server/). Even this post does not provide a working example configuration that works for forwarding to generic C2 servers, and instead used a custom C2 backend - but the App Engine service itself WILL work . 

# Abusable cloud services by category

Now we will start to look at the cloud services (AWS, Azure and GCP) that I've personally verified for use as high reputation redirectors for C2 in the last 6 months. This section will group the services by service category, and the following section will group them by provider.

## VM

When you create a virtual machine instance in Azure or AWS, and give that instance a public IP address, you can get a cloud provider domain name associated with that public IP. You can create your own proxy service on that instance that can forward incoming traffic reaching the address to a backend of your choice. This requires that you install and configure the proxying software yourself on the instance, using Apache, Nginx or similar. If you want HTTPS, you need to set this up yourself.  One no cost way you can do this is by getting a certificate from LetsEncrypt for the cloud provider domain name and configure this with your proxying software. This will result in your domain name ending up in the certificate transparency logs and receiving some amount of attention in the form of scanning of the domain, so be aware of this consequence any time you need to setup HTTPS yourself.

**Note**: I only have a blog post discussing a POC implementation of the Azure VM approach, but the discussed approach is largely transferrable to AWS as well. In addition, the POC (simplified for cost and ease of use) involves running the C2 server on the same VM, with an Apache server running proxying traffic to the C2 HTTP interface listening on `localhost:8888`. This configuration is discussed in some more detail in the section [above](#protectionfiltering-for-the-c2-servers-http-endpoint). I dont recommend running the C2 on the same host that is receiving public traffic for any production use, and would suggest instead running just the proxy on the public VM, and having the C2 on its own seperated host. The Apache config in the design forwards communication to `http://backend:8888` (with the address of `backend` defined in the `/etc/hosts` file), and this can be modified to a new destination of your choice.


**AWS EC2**:
* **Domain name**: `ec2-<IP-ADDRESS>.<REGION>.compute.amazonaws.com`
* **HTTPS?**: BYO (configure using LetsEncrypt or similar)


**[Azure VM](/2025/07/10/azure-service-c2-forwarding-part3.html)**:
* **Domain name**: `<YOUR-LABEL>.<REGION>.cloudapp.azure.com`
* **HTTPS?**: BYO (configure using LetsEncrypt or similar)


## Content Delivery Networks (CDN)

Content Delivery Networks make your content more globally available using points of presence around the world. These services are designed to make generic websites more available, so configuring these services for C2 redirection just involves setting the backend C2 service in the CDN configuration. Be aware that these CDN services are also biased towards caching content, which will cause a problem for C2 HTTP communication, so you need to make sure you disable caching to work around this.


**[AWS CloudFront](/2023/08/30/aws-service-C2-forwarding.html#cloudfront-distribution-forwarding)**
* **Domain name**: `<RANDOM>.cloudfront.net`
* **HTTPS?**: Yes


**[Azure Front Door](/2025/06/25/azure-service-c2-forwarding-part2.html)**
* **Domain name**: `<YOUR-LABEL>-<RANDOM>.*.azurefd.net`
* **HTTPS?**: Yes


## API Services

API Services make it easy to run your HTTP based API and make it available on the Internet. The approach required to configure these services to forward arbitrary HTTP traffic differs depending on the service. For AWS, you have options to either send incoming traffic to backend serverless code which will forward the messages, or you can configure direct proxying. For Azure and GCP, you need to provide specifically structured API definitions that will forward traffic from arbitrary routes.

**Note**: As mentioned, the AWS API Gateway can use two different backend, both approaches of which are discussed in the [same post here](/2023/08/30/aws-service-C2-forwarding.html). The first approach is [forwarding to a Lambda](/2023/08/30/aws-service-C2-forwarding.html#api-gateway-to-lambda), and the second approach is [direct proxying to a backend HTTP service](/2023/08/30/aws-service-C2-forwarding.html#api-gateway-direct-proxying). Lambdas can also have web traffic directed into them using two different approaches, and are discussed below in the Serverless category.


**[AWS API Gateway forwarding to Lambda](/2023/08/30/aws-service-C2-forwarding.html#api-gateway-to-lambda)**
* **Domain name**: `<RANDOM>.execute-api.<REGION>.amazonaws.com`
* **HTTPS?**: Yes

**[AWS API Gateway direct proxying](/2023/08/30/aws-service-C2-forwarding.html#api-gateway-direct-proxying)**
* **Domain name**: `<RANDOM>.execute-api.<REGION>.amazonaws.com`
* **HTTPS?**: Yes


**[Azure API Management Service](/2025/07/30/azure-service-c2-forwarding-part4.html)**
* **Domain name**: `<YOUR-LABEL>.azure-api.net`
* **HTTPS?**: Yes


**[GCP API Gateway](/2025/06/25/azure-service-c2-forwarding-part2.html)**
* **Domain name**: `<YOUR-LABEL>-<RANDOM>.<REGIONID>.gateway.dev`
* **HTTPS?**: Yes


## Managed Website services

These services make it easier to launch a website on the Internet. In the case of AWS Amplify this website can be a direct proxy configuration to a backend site, for Elastic Beanstalk you can run a custom web application that can forward traffic to a chosen backend.

**[AWS Amplify](/2023/08/30/aws-service-C2-forwarding.html#aws-amplify-application)**
* **Domain name**: `<YOUR-LABEL>.<RANDOM>.amplifyapp.com`
* **HTTPS?**: Yes


**[AWS Elastic Beanstalk](/2025/08/27/aws-service-c2-forwarding-part3.html)**
* **Domain name**: `<YOUR-LABEL>.<RANDOM>.<REGION>.elasticbeanstalk.com`
* **HTTPS?**: BYO (configure using LetsEncrypt or similar)


## Container execution services

These services make it easier to run a container and make it accessible on the Internet for web traffic. To use this for C2 redirection, we can run a custom container similar to [this](https://github.com/stephenbradshaw/HTTPForwardContainer) that will forward traffic to a configured backend. The GCP Cloud Run service mentioned in this section also operates as a Serverless service, so will be listed in both sections - if you provide it a container it runs the container, if you provide it code it will convert that to a container for you and run it.

**Note**: The GCP Cloud Run [post](/2025/04/23/gcp-service-C2-forwarding-part-3.html) I have written only discusses implementation using the Serverless code approach.


**[AWS AppRunner](/2025/08/20/aws-service-c2-forwarding-part2.html)**
* **Domain name**: `<RANDOM>.<REGION>.awsapprunner.com`
* **HTTPS?**: Yes


**[Azure Container Applications](/2025/08/27/azure-service-c2-forwarding-part5.html)**
* **Domain name**: `<YOUR-LABEL>.<ENVIRONMENT-ID>.<REGION>.azurecontainerapps.io`
* **HTTPS?**: Yes


**[GCP Cloud Run](/2025/04/23/gcp-service-C2-forwarding-part-3.html)**
* **Domain name**: `<YOUR-LABEL>-<PROJECT>.<REGION>.run.app`
* **HTTPS?**: Yes


## Serverless code execution services

Serverless code execution services allow you to provide a bit of code that the cloud provider will wrap in a lightweight VM or container that will run without you having to maintain the host. This will then be made available to receive web traffic at a cloud provider domain name.  The code used in the POCs I discuss below are variants on a Python app that rewrites HTTP requests. [Flask](https://flask.palletsprojects.com/) is used for receiving web requests where this is not handled by the parent framework, and either [requests](https://requests.readthedocs.io/) or the Python standard [HTTP](https://docs.python.org/3/library/http.html) and [SSL](https://docs.python.org/3/library/ssl.html) libraries are used to replay the requests to the backend C2 service. 

**Note**: The Lambda function code referred to in the post below is an older version that requires the Python [requests](https://requests.readthedocs.io/)  module which used to be included in the Lambda Python runtime environment in older versions, but now needs to be added alongside your code in a layer. My updated version of the Lambda code available [here](https://github.com/stephenbradshaw/pentesting_stuff/blob/master/redirector_lambda/lambda_function_updated.py) uses the Python standard http and ssl libraries, and can be used with the latest available Python Lambda runtime version without having to add any additional modules.

**[AWS API Gateway forwarding to Lambda](/2023/08/30/aws-service-C2-forwarding.html#api-gateway-to-lambda)**
* **Domain name**: `<RANDOM>.execute-api.<REGION>.amazonaws.com`
* **HTTPS?**: Yes

**[AWS Lambda from Function URL](/2023/08/30/aws-service-C2-forwarding.html#function-url-to-lambda)**
* **Domain name**: `<RANDOM>.lambda-url.<REGION>.on.aws`
* **HTTPS?**: Yes

**[Azure Function App](/2025/05/07/azure-service-C2-forwarding.html)**
* **Domain name**: `<YOUR-LABEL>.azurewebsites.net`
* **HTTPS?**: Yes

**[GCP Cloud Run](/2025/03/26/gcp-service-C2-forwarding.html)**
* **Domain name**: `<YOUR-LABEL>-<PROJECT>.<REGION>.run.app`
* **HTTPS?**: Yes

**[GCP Cloud Run Functions](/2025/04/23/gcp-service-C2-forwarding-part-3.html)**
* **Domain name**: `<REGION>-<PROJECT>.cloudfunctions.net/<YOUR-LABEL>/`
* **HTTPS?**: Yes

**[GCP App Engine](/2025/03/26/gcp-service-C2-forwarding.html)**
* **Domain name**: `<PROJECT>.<REGION>.*.appspot.com`
* **HTTPS?**: Yes


# Abusable cloud services by provider

This section lists the same services from the section above, but categorised by cloud provider.


## AWS Services

**[AWS CloudFront](/2023/08/30/aws-service-C2-forwarding.html#cloudfront-distribution-forwarding)**
* **Domain name**: `<RANDOM>.cloudfront.net`
* **HTTPS?**: Yes

**[AWS API Gateway forwarding to Lambda](/2023/08/30/aws-service-C2-forwarding.html#api-gateway-to-lambda)**
* **Domain name**: `<RANDOM>.execute-api.<REGION>.amazonaws.com`
* **HTTPS?**: Yes

**[AWS API Gateway direct proxying](/2023/08/30/aws-service-C2-forwarding.html#api-gateway-direct-proxying)**
* **Domain name**: `<RANDOM>.execute-api.<REGION>.amazonaws.com`
* **HTTPS?**: Yes

**[AWS Lambda from Function URL](/2023/08/30/aws-service-C2-forwarding.html#function-url-to-lambda)**
* **Domain name**: `<RANDOM>.lambda-url.<REGION>.on.aws`
* **HTTPS?**: Yes

**[AWS Amplify](/2023/08/30/aws-service-C2-forwarding.html#aws-amplify-application)**
* **Domain name**: `<YOUR-LABEL>.<RANDOM>.amplifyapp.com`
* **HTTPS?**: Yes

**[AWS Elastic Beanstalk](/2025/08/27/aws-service-c2-forwarding-part3.html)**
* **Domain name**: `<YOUR-LABEL>.<RANDOM>.<REGION>.elasticbeanstalk.com`
* **HTTPS?**: BYO

**[AWS AppRunner](/2025/08/20/aws-service-c2-forwarding-part2.html)**
* **Domain name**: `<RANDOM>.<REGION>.awsapprunner.com`
* **HTTPS?**: Yes

**AWS EC2** (follow the approach discussed in the post on [abusing Azure VMs](/2025/07/10/azure-service-c2-forwarding-part3.html) )
* **Domain name**: `ec2-<IP-ADDRESS>.<REGION>.compute.amazonaws.com`
* **HTTPS?**: BYO (configure using LetsEncrypt or similar)


## Azure services

All of the following Azure services can be easily deployed in POC form using deployment templates as discussed in the post [here](/2025/06/04/azure_c&c_poc_infra_deployment.html). These templates are available on GitHub [here](https://github.com/stephenbradshaw/AzureC2PocDeployment).

**[Azure Function App](/2025/05/07/azure-service-C2-forwarding.html)**
* **Domain name**: `<YOUR-LABEL>.azurewebsites.net`
* **HTTPS?**: Yes

**[Azure Front Door](/2025/06/25/azure-service-c2-forwarding-part2.html)**
* **Domain name**: `<YOUR-LABEL>-<RANDOM>.*.azurefd.net`
* **HTTPS?**: Yes

**[Azure VM](/2025/07/10/azure-service-c2-forwarding-part3.html)**:
* **Domain name**: `<YOUR-LABEL>.<REGION>.cloudapp.azure.com`
* **HTTPS?**: BYO (configure using LetsEncrypt or similar)

**[Azure API Management Service](/2025/07/30/azure-service-c2-forwarding-part4.html)**
* **Domain name**: `<YOUR-LABEL>.azure-api.net`
* **HTTPS?**: Yes

**[Azure Container Applications](/2025/08/27/azure-service-c2-forwarding-part5.html)**
* **Domain name**: `<YOUR-LABEL>.<ENVIRONMENT-ID>.<REGION>.azurecontainerapps.io`
* **HTTPS?**: Yes


## GCP services


**[GCP Cloud Run](/2025/03/26/gcp-service-C2-forwarding.html)**
* **Domain name**: `<YOUR-LABEL>-<PROJECT>.<REGION>.run.app`
* **HTTPS?**: Yes

**[GCP Cloud Run Functions](/2025/04/23/gcp-service-C2-forwarding-part-3.html)**
* **Domain name**: `<REGION>-<PROJECT>.cloudfunctions.net/<YOUR-LABEL>/`
* **HTTPS?**: Yes

**[GCP App Engine](/2025/03/26/gcp-service-C2-forwarding.html)**
* **Domain name**: `<PROJECT>.<REGION>.*.appspot.com`
* **HTTPS?**: Yes


**[GCP API Gateway](/2025/06/25/azure-service-c2-forwarding-part2.html)**
* **Domain name**: `<YOUR-LABEL>-<RANDOM>.<REGIONID>.gateway.dev`
* **HTTPS?**: Yes


