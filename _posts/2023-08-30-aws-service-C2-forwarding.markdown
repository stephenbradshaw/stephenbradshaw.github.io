---
layout: post
title:  "AWS Service Command and Control HTTP traffic forwarding"
date:   2023-08-30 18:04:00 +1100
author: Stephen Bradshaw
tags:
- AWS
- Lambda
- Amplify
- API Gateway
- Function URL
- Serverless
- C2
- domain fronting
- forwarding
---

Recently I've been looking into options for abusing AWS services to forward HTTP Command and Control (C2) traffic. This post will talk about a number of approaches for this I found discussed on the Internet as well as a few options that I identified myself.

For those not familiar with how most modern C2 systems work, an overview of their operation might be helpful. Skip ahead a paragraph if you are already familiar with the way C2 systems are designed. 

A Command and Control system provides an interface by which commands can be run on already compromised computers in order for attackers to achieve their goals. Specific terms vary, but C2 architecture normally consists of implants that run on compromised devices and run commands, interfaces that attackers can use to issue commands to be run on compromised devices, and servers that coordinate communications between the two. The C2 server normally sits in a location reachable from the Internet, so victim systems with the C2 implant installed can communicate back to the server to ask for instructions. Depending on the C2 software in question, there will be one or more protocols supported for this purpose. The protocols chosen are repurposed, in that they were originally designed and used for some other benign purpose. This repurposing is done deliberitely in order to make the C2 implant communications blend in to normal network traffic. The most commonly supported protocol for implant to server communication in modern C2 systems is HTTP/S. 

The use of HTTP/S by C2 is not the only way this protocol is abused for malicious activity, so defenders are paying attention. One approach to try and identify abuse of the protocol is to check the "reputation" of HTTP/S traffic destinations using a source like [this](https://urlfiltering.paloaltonetworks.com/query/) (other sources are available). By making use of AWS services that can proxy HTTP traffic, the operator of a C2 server can take advantage of the comparitively "good" reputation of the URLs associated with those AWS services to try and avoid detection. So, my question was, what AWS services can be used in this manner?


# Previous work

I started off this exercise by looking for existing writeups on the topic. I considered out of scope anything that required custom C2 implant communications (e.g. [External C2](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics/listener-infrastructure_external-c2.htm?cshid=1043)). I wanted forwarding of plain old HTTP/S to give me the widest possible range of options in C2 servers that could be put behind this without requiring code changes.

After a few hours of research, I found the following:
* [This post](https://blog.xpnsec.com/aws-lambda-redirector/) by Adam Chester which talks about using the [Serverless framework](https://serverless.com/) to create an [AWS Lambda](https://aws.amazon.com/lambda/) that will receive traffic from the [AWS API Gateway](https://aws.amazon.com/api-gateway/) and then forward it to another arbitrary destination. In Adams example, he forwards the traffic using ngrok to a local web server, but this approach could be used point to an EC2 instance in the same AWS account or other server on the Internet.
* [This post](https://scottctaylor12.github.io/lambda-function-urls.html) from Scott Taylor which again uses a Lambda to perform traffic redirection, but this time the entry point is via [Lambda function URLs](https://docs.aws.amazon.com/lambda/latest/dg/lambda-urls.html) instead of the API Gateway. There is associated code/instructions to deploy this Lambda to AWS as well as to create an EC2 instance to forward the traffic to.
* Many posts on using a [CloudFront distribution](https://aws.amazon.com/cloudfront/) and domain fronting to receive traffic from CloudFront sites and send them to a C2 server somewhere else. Some examples are [this](https://www.cobaltstrike.com/blog/high-reputation-redirectors-and-domain-fronting), [this](https://digi.ninja/blog/cloudfront_example.php) and [this](https://www.mdsec.co.uk/2017/02/domain-fronting-via-cloudfront-alternate-domains/). There are a bunch more too. This topic of domain fronting in CloudFront deserves a deeper dive, because things have changed....



# A note on Domain fronting and CloudFront

Something that you will see repeated endlessly if you search on this topic is that you can use the CloudFront Content Delivery Network (CDN) to perform domain fronting for C2 services. 

Domain fronting is a technique that attempts to hide the true destination of a HTTP request or redirect traffic to possibly restricted locations by abusing the HTTP routing capabilities of CDNs or certain other complex network environments. For version 1.1 of the protocol, HTTP involves a TCP connection being made to a destination server on a given IP address (normally associated with a domain name) and port, with additional TLS/SSL encryption support for the connection in HTTPS. Over this connection a structured plain text message is sent that requests a given resource and references a server in the `Host` header. Under normal circumstances the domain name associated with the TCP connection and the `Host` header in the HTTP message match.  In domain fronting, the destination for the TCP connection domain name is set to a site that you want to appear to be visiting, and the `Host` header in the HTTP request is set to the location you actually want to visit. Both locations must be served by the same CDN. 

The following curl command demonstrates in the simplest possible way how the approach is performed in suited environments. In the example, `http://fakesite.cloudfront.net/` is what you want to **appear** to be visiting, and `http://actualsite.cloudfront.net` is where you actually want to go:

```
curl -H 'Host: actualsite.cloudfront.net'  http://fakesite.cloudfront.net/
```

In this example, any DNS requests resolved on the client site are resolving the "fake" address, and packet captures will show the TCP traffic going to that fake systems IP address. If HTTPS is supported, and you use a `https://` URL, the actual destination you are visiting located in the HTTP `Host` header will also be hidden in the encrypted tunnel.

While this **is** a great way of hiding C2 traffic, due to a widespread practice of domain fronting being used to evade censorship restrictions, various CDNs did crack down on the approach a few years ago. Some changes were rolled back in some cases, but as at the time of this writing this simple approach to domain fronting **does not work** in CloudFront for HTTPS. If the DNS hostname that you connect to does not match any of the certificates you have associated with your CloudFront distribution, you will get the following error:

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


# Summary of identified approaches

With that diversion out of the way, lets look at the complete list of options I identified for forwarding HTTP traffic using AWS services.

Heres the list, including the afore mentioned approaches I found discussed elsewhere on the Internet, and a few more I identified myself:
* **Function URLs execute an AWS Lambda that forwards HTTP requests and responses**.  In this approach, requests enter the AWS account via the Function URL HTTP endpoint and are handled by an AWS Lambda. This Lambda forwards requests and responses between the implant and a backend server of your choice. The backend server can be an EC2 instance in the same AWS account, or any other HTTP/S service that the Lambda can reach, including servers accessible on the Internet. This is the [approach from Scott Taylor](https://scottctaylor12.github.io/lambda-function-urls.html) as mentioned above.
* **The API gateway executing an AWS Lambda that forwards HTTP requests and responses**. In this approach, requests enter the AWS account via the API gateway HTTP endpoint, and are handled by an instance of the AWS Lambda. [This approach](https://blog.xpnsec.com/aws-lambda-redirector/) originally from Adam Chester is functionally very similar to the afore mentioned Function URL approach, with the only differences being the entry point into the AWS account and the setup. Even though Adam and Scott had different Lambda functions in each of their blog posts I found it was possible to use [Scott's function](https://github.com/scottctaylor12/Red-Lambda/blob/main/lambda.py) for both approaches. It is even possible to configure the same Lambda function instance to handle incoming requests from both a Function URL endpoint and one or more API Gateway endpoints at the same time. There are some differences in how the entrypoints have to be used depending on what type of API gateway is used and how its configured, that will be discussed below. 
* **CloudFront distribution forwarding to a back end service**. In this approach, requests enter the AWS account via CloudFront, and can be forwarded to various backends, such as an AWS load balancer or Internet accessible URL. Theres lots of online resources talking about how to set this up so I wont be going into a great deal of detail about it here, but be aware that some references are out of date when it comes to the domain fronting point that I've discussed above.
* **API gateway direct proxying**. In this approach, instead of taking requests from the API Gateway and sending them to a Lambda, you instead proxy them directly to a private resource or other URL. The private resource can be a [Cloudmap](https://aws.amazon.com/cloud-map/) service (that you can point to an EC2 instance and port), an AWS Application Load Balancer [(ALB)](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html) or a Network Load Balancer [(NLB)](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html) and allows the traffic to be forwarded within the AWS account without transiting the Internet. The HTTP URI option allows forwarding to another Internet accessible URL.
* **AWS Amplify application**. In this approach, an Amplify application acts as a proxy to forward traffic to another Internet accessible location - such as an existing CloudFront distribution. An [Amplify app](https://aws.amazon.com/amplify/) provides an easy way to quickly generate a web application that will be hosted on an auto generated domain name delivered through CloudFront at the `*.amplifyapp.com` domain. 

Out of these, my favorite approach is the API Gateway proxying method. In a coming section, I'll talk at a high level about how to implement each of these approaches, as well as some of the more relevant details for C2 forwarding that apply. First however, given that its referenced in two of the above options I want to go over the relevant differences between the two API Gateway types for C2 forwarding.


# API Gateway: REST vs HTTP

The API gateway offers two main types of API - REST and HTTP. The API documentation provides a lot of information on choosing between these two types of gateway starting [here](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html), but from our perspective of fronting C2 traffic the important points are as follows:
* The entry point URLs look the same for REST and HTTP types and fit the pattern of `https://<random_10_chr_str>.execute-api.<region>.amazonaws.com/<path>`. 
* It is possible to perform domain fronting with HTTPS with another API gateway instance in the same region and of the same type (e.g. HTTP or REST).
* Both types can be used to forward to a Lambda or to a HTTP URI or various other AWS services, although the integration of API stage names in entry point URLs can complicate how well this works for C2 as discussed in the next point.
* The HTTP type is the only type that I have been able to configure to proxy services to a bare URI. The REST type requires that the `stage` be included at the start of the URI path. For example a REST API entrypoint would look like this `https://<rest-api>.execute-api.<region>.amazonaws.com/stage_name/`, whereas a HTTP one could look like this `https://<http-api>.execute-api.<region>.amazonaws.com/`. Some C2 servers can deal with additional path information in the URL without a problem although this does make certain proxying configurations more complex for REST types. 

Due to last point alone my preference is to use HTTP API gateway types instead of REST ones for C2 forwarding, whether via Lambda or direct proxying, and this is the implementation approach I recommend below in cases where the API gateway is used. This also means that my suggested method for implementing the API Gateway<->Lambda forwarder is different than the Serverless approach discussed in [Adam's post](https://blog.xpnsec.com/aws-lambda-redirector/). I think this difference in approach here is largely due to the rapid increase in functionality of AWS services over time - I don't believe the HTTP API Gateway type was available back when Adam originally wrote his post.

# Overview and implementation

The following are instructions on how to implement each of the afore mentioned C2 forwarding approaches and a summary of some of their relevant distinguishing features. The assumption I have made with the instructions is that the destination C2 server sits within the same AWS account as the AWS forwarding service being configured, and that you are following a minimal access permission model in your account. I havent made any specific assumptions about the rest of the C2 design in your network, although my design involved an additional reverse HTTP proxy that handled all implant HTTP traffic destined for the C2 box. In cases where I refer to en EC2 instance receiving HTTP/S from the AWS service forwarder - this was the box being referred to. If theres interest, I can do a seperate post on this design, but for the purpose of this post Ive tried to keep the instructions generic. 

The instructions are fairly bare bones, listing the minimal configuration settings you need to set to make the service functional, and assume you have fairly decent knowldge of how AWS networking, IAM and security groups work. You might need to refer to the AWS documentation for specific services to find where a particular referenced setting is set. These are the manual click-ops steps, but if you want to rapidly deploy and tear down your infrastructure you will obviously want to implement these steps in an Infrastructure as Code format.


## Function URL to Lambda

**Summary**
* **_URL format_**: `https://<random_32_char_value>.lambda-url.<aws_region>.on.aws`
* **_HTTPS certificate_**: Valid certificate automatically provided on Function URL creation
* **_Palo Alto URL categories_**: `Computer and Internet Info`, `Low Risk`
* **_Domain fronting_**: Works via Host header manipulation for HTTP and HTTPS for arbitrary values within the same region matching pattern `*.lambda-url.<aws_region>.on.aws`


**Setup**

[Scott's blog post](https://scottctaylor12.github.io/lambda-function-urls.html) and associated [Red Lambda Github repository](https://github.com/scottctaylor12/Red-Lambda) provide some instructions and CloudFormation code to implement a Lambda/Function URL forwarder and C2 system, but if your design is different its helpful to know know to do the Function URL and Lambda setup manually.

1. Take the [Lambda code from Scotts repository](https://github.com/scottctaylor12/Red-Lambda/blob/main/lambda.py). Depending on when you follow these instructions, you can also use [my fork instead]((https://github.com/stephenbradshaw/Red-Lambda/blob/main/lambda.py), thats awaiting PR acceptance into Scott's repository and fixes an issue with proper forwarding of binary responses from the backend C2 server.

2. Create a new Lambda using the Python 3.7 runtime (later versions will not work due to issues with the Python requests module).

3. The handler for the Lambda should be set to `lambda_function.redirector` assuming a code filename of `lambda_function.py`.

4. Set an environment variable of `TEAMSERVER` to point to the private IP address or name of the HTTPS capable service you want to redirect to.

5. Associate the appropriate VPC and subnet with the Lambda (these should match the VPC and subnet of the destination EC2 instance) and create a dedicated security group for the Lambda.

6. Add a security rule to the Security Group for the destination EC2 instance that allows HTTPS from the security group associated with the Lambda.

7. When creating the Lambda, choose to associate a Function URL with it.

8. The Lambda execution role should be a custom IAM role with the `AWSLambdaExecute` managed policy AND the following custom policy attached.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "LambdaRedir",
            "Effect": "Allow",
            "Action": [
                "ec2:CreateNetworkInterface",
                "ec2:DeleteNetworkInterface",
                "ec2:DescribeInstances",
                "ec2:AttachNetworkInterface",
                "ec2:DescribeNetworkInterfaces"
            ],
            "Resource": "*"
        }
    ]
}
```


An auto generated Function URL address will be provided in the console once configuration is complete.



## API Gateway to Lambda

**Summary**
* **_URL format_**: `https://<random_10_char_value>.execute-api.<region_name>.amazonaws.com/`
* **_HTTPS certificate_**: Valid certificate automatically provided on API Gateway creation
* **_Palo Alto URL categories_**: `Computer and Internet Info`, `Low Risk`
* **_Domain fronting_**: Works via Host header manipulation for HTTP and HTTPS for valid API Gateway domains in the same region and of the same type (REST or HTTP)


**Setup**

As mentioned in the [API Gateway](#api-gateway-rest-vs-http) section above, I prefer using the HTTP API Gateway type as opposed to the REST type, so my setup approach is different to the way that API Gateway/Lambda forwarding was setup in [Adam's blog post](https://blog.xpnsec.com/aws-lambda-redirector/). 

The setup is pretty straightforward.

1. Setup a forwarder Lambda as described in the section above. This Lambda code works with both Function URL and API Gateway triggers. Given you are using API Gateway forwarding you can skip the step where you enable a Function URL if you like or leave it if you want both entrypoints enabled.

2. Create a `HTTP` API Gateway instance.

3. Create a `/{proxy+}` resource for the `ANY` method.

4. Add an AWS Lambda integration, pick the region where your Lambda resides and the Lambda name. There will be an option `Grant API Gateway permission to invoke your Lambda function` which you can leave selected. No authorizer is required.

5. Create a `$default` stage and enable automatic deployment.

The invoke URL will be provided on the summary page for the gateway.


## CloudFront distribution forwarding

**Summary**
* **_URL format_**: `https://<13_character_random_value>.cloudfront.net/`
* **_HTTPS certificate_**: Valid certificate automatically provided on API Gateway creation
* **_Palo Alto URL categories_**: `Content Delivery Networks`, `Low Risk` for *cloudfront.net URLS, or domain dependant for custom domains
* **_Domain fronting_**: Works via Host header for HTTP, requires using SNI as part of the TLS session initiation set to the same value as the host header for HTTPS for other valid sites delivered by the CDN


**Setup**

There are dozens of resources on the Internet that describe how to use CloudFront as a forwarder for C2, so I wont go into detail on how to configure it here, you can check one of the many other resources for detail instructions. Amongst those linked above I also used [this](https://www.blackhillsinfosec.com/using-cloudfront-to-relay-cobalt-strike-traffic/) as a reference when setting up my POC.

I will provide some general notes and tips I had about the creation process.

Using a CloudFront distribution for C2 fronting requires:
* An AWS load balancer of some type (ALB, NLB or classic) setup to forward traffic to the destination HTTP/S service (e.g. EC2 instance)
* A custom domain and AWS hosted SSL certificate **IF** use of a custom domain is desired

Each of the different load balancer options has different characteristics that might influence the right option to choose depending on the environment.
* The ALB requires that you specify at least two subnets in different availability zones as traffic destinations to be selected during setup, so may not be suitable for single AZ setups.
* The NLB does a direct pass through of the client IP address to the destination EC2 instance, so does require that Internet access rules be applied in the security group associated with the EC2 instance. To restrict access to only CloudFront origin endpoints (as opposed to the entire Internet) requires in the neighbourhood of 55 rules to be added to the security group. This may require a limit increase in the associated AWS account if many existing rules are already present, as the default maximum rules in a security group is 60.
* The Classic Load balancer is considered to be "out of date" technology but does not have any current end of life set and does work reliably. It does not have either of the afore mentioned restrictions, having its own dedicated security group and no requirement for multiple availability zone routing. The main downside is the mandatory health check requests generating unnecessary log entries on the destination HTTP/S service.

To setup a classic load balancer add one in the same VPC as the destination EC2 instance, create a dedicated security group and add a rule to allow traffic to port 80 from the managed prefix list `com.amazonaws.global.cloudfront.origin-facing` - this allows only traffic from the CloudFront servers to reach the load balancer. The id of the list can be looked up in the Managed prefix lists section of the VPC console in order to add it to the security group - this was `pl-b8a742d1` at the time this post was written. The origin in the CloudFront distribution should then be set to forward traffic using HTTP.

If access via a custom domain is required, the domain needs to have a SSL certificate added with all desired names (e.g. domain.com, www.domain.com)for the domain in the us-east-1 region (N.Virginia) region. That certificate can then be selected in the Custom SSL Certificate section of the Distributions settings. The names in the certificate should include all of the Alternate Domain Name (CNAME) entries in the distribution. A link to the appropriate section of the AWS console to create the certificate is shown in the wizard for creating a CloudFront distribution.

Other than the two previously mentioned options, the other important values to set in the distribution relate to forwarding behavior - specifically the caching and allowable HTTP methods. Allow all HTTP methods and use Legacy cache settings selecting All for Headers, Query strings and Cookies.

The distribution domain name will be provided in the settings.


## API Gateway direct proxying

**Summary**
* **_URL format_**: `https://<random_10_char_value>.execute-api.<region_name>.amazonaws.com/`
* **_HTTPS certificate_**: Valid certificate automatically provided on API Gateway creation
* **_Palo Alto URL categories_**: `Computer and Internet Info`, `Low Risk`
* **_Domain fronting_**: Works via Host header manipulation for HTTP and HTTPS for valid API Gateway domains in the same region and of the same type (REST or HTTP)


**Setup**

The API Gateway direct proxying approach allows you to forward to an Internet accessible URI or a private resource. For cases where the C2 sits in the same AWS account as the forwarding service, a private resource is preferrable as it does not require that you expose your C2 service directly on the Internet and you can save on AWS network traffic transit costs. For a private resource you can forward to a load balancer or a Cloud Map service that points to one or more services running on cloud resources (e.g. web servers on EC2 instances) that you want to receive your forwarded traffic. I chose to use a Cloud Map service pointing to port 80 on an EC2 instance as it was more cost effective than a load balancer. Seeing as the forwarded traffic is internal to my AWS account I was forwarding it as HTTP not HTTPS.

The following instructions explain how to setup proxying to a Cloud Map service that will point to a target EC2 instance. Take note of the target EC2 instances private IP address, VPC and subnet before starting as these details will be required:


1. Setup a AWS Cloud Map namespace supporting `API calls and DNS queries in VPCs`. You can put it in the same VPC as your destination EC2 instance.

2. Create a cloud map `API and DNS` service within the namespace crated in step 1. 

3. Create a service instance within the service created in step 2. This service instance should point to the private IP address of the target EC2 instance and TCP port 80.

4. Create a security group in the VPC/subnet where the target EC2 instance resides that can be used to associate with a VPC link. This will be used to allow the VPC link and hence the API gateway to talk **TO** the destination EC2 instance.

5. Add a rule to the target EC2 instances security group that allows connections **FROM** the security group created in step 4 to the same service port configured in the Cloud Map service instance (e.g. 80/HTTP).

6. Create an API Gateway VPC link for HTTP APIs to provide a path for the API Gateway to communicate with the VPC and subnet where the destination EC2 instance resides. 

7. Associate the VPC link created in step 6 with the VPC and subnet that the target EC2 instance resides in and the security group created in step 4. 

7. Create a API Gateway `HTTP` API instance, without creating an initial integration. Keep the default `$default` deployment stage with automated deployment.

8. In the new API gateway, create a route with pattern `/{proxy+}` resource for the `ANY` method.

9. In the new API gateway, create a private resource integration that points to the Cloud Map service created in step 2. Associate the VPC link created in step 7 with the integration.

10. Attach the integration created in step 9 to the route created in step 8.

The invoke URL will be shown in the stage configuration page.


## AWS Amplify application


**Summary**
* **_URL format_**: `https://<stage_name>.<14_character_random_value>.amplifyapp.com/`
* **_HTTPS certificate_**: Valid certificate automatically provided on API Gateway creation
* **_Palo Alto URL categories_**: `Business and Economy`, `Low Risk`
* **_Domain fronting_**: Delivered through the CloudFront CDN, the same domain fronting conditions for CloudFlare as discussed above apply

**Setup**

The AWS Amplify fronting method requires an Internet accessible URI to forward traffic to. When you have this you can create an Amplify application with an empty code deployment and then configure rewriting to redirect all requests to the application to your desired site. The site is delivered via CloudFront and visitors to the site wont be able to tell they are being redirected. Setup like so:


1. Create a new AWS Amplify hosted environment. Choose a manual file upload based distribution and upload an empty zip file. Something like the following can be used to create a zip file test1.zip in the PWD from empty folder /tmp/test.

```
python -c 'import shutil; shutil.make_archive("test1", "zip", "/tmp/test")'
```

2. Then in the Rewrites and Redirects section of the settings, add a redirect from source `/<*>` to target `https://web.site/<*>` (or your custom destination) of type `200 (Rewrite)`

Once configured the URL for the app will be available in a few locations throughout the Amplify interface.