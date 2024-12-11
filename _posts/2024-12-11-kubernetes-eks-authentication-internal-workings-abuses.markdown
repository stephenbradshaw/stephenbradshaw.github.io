---
layout: post
title: "Kubernetes EKS Authentication internal workings and abuses"
date: 2024-12-11 17:20:00 +1100
author: Stephen Bradshaw
tags:
- kubernetes
- pentesting
- pentest
- red team
- persistence
- iam
- aws
- eks
- container
- authentication
- ConfigMap
- aws-auth
- kube-system
- privilege escalation
- lateral movement
---

Wanted to do a quick follow up/addition to this previous post [here](/2023/11/15/kubernetes-auth-deep-dive.html) where I talked about Kubernetes authentication. This time around, I'm going to talk a little about the workings of the AWS EKS authentication extension for Kubernetes, which allows you to make calls to the Kubernetes API by authenticating using AWS credentials, so you can better understand how it can be abused. 

Theres a reasonable amount of documentation out there on this authentication method, including from an offensive perspective, with some good sources including [this](https://www.panoptica.app/research/eks-authentication-part-1) and [this](https://www.panoptica.app/research/exploiting-authentication-in-aws-iam-authenticator-for-kubernetes). Consequently, for this post I won't do a full overview here but will instead just cover off on a few aspects I haven't seen covered in the specific detail that I was after when I initially statred looking into this. If after you read this you want more details, check the afore-mentioned posts, as well as the [official documentation](https://docs.aws.amazon.com/eks/latest/userguide/grant-k8s-access.html).

The EKS authentication approach involves creating custom Bearer tokens passed in the "Authorization" header in requests to the Kubernetes API, similar to how native service account tokens are used. The tokens are comprised of base64 encoded, presigned URLs for the [AWS STS API](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetCallerIdentity.html) (the API used when you run `aws sts get-caller-identity`), which have been customised by including the cluster-id or cluster-name of the relevant EKS cluster in an additional header as part of the signature calculation. These tokens can be directly used for authentication of individual `kubectl` command executions via the `--token` switch, again in the same way as for service account tokens. 

To get these tokens using native tools, you can use the `aws eks get-token` CLI command to get an individual token (valid for ~ 15 minutes), or the `aws eks update-kubeconfig` command to write a kubectl config file that will provide a regularly refreshed version of the token for each execution of the tool. Both commands require either the cluster name or id to be specified using the appropriate switches for the command to run.

A curl-ified version of using one of these tokens to make an API call might look something like this:
```
curl -k -H "Authorization: Bearer $(aws eks get-token --cluster-name <CLUSTER_NAME> | jq -r .status.token)" "https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT/api/v1/"
```

If you want to create the EKS tokens directly yourself, you can do so via wrapping/packaging presigned URLs created using the boto3 library, although you do need to modify the URL creation process slightly. After spending some time trying to figure out exactly how these tokens were crafted, I finally figured it out after reading the AWS CLI source code and wrote/borrowed a simple bit of Python code that implements this which is available [here](https://github.com/stephenbradshaw/pentesting_stuff/blob/master/utilities/create_eks_auth_token.py) so you can see how it works if you're interested. The code allows you to directly specify the credentials to use or have the code pull them from the environment as per the usual process boto uses where it looks for them in credential files, environment variables, metadata sources, etc.

Identities in AWS IAM are mapped to Kubernetes specific roles by use of either mappings specified in the `aws-auth` [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) in the `kube-system` namespace, or, more recently, with [EKS access entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html). You can identify which of these approches are being used by checking the `cluster.accessConfig.authenticationMode` value that you can query by running `aws eks describe-cluster` for a given cluster, which will be set to possible values of `CONFIG_MAP`, `API` or `API_AND_CONFIG_MAP`. These values indicate that either the ConfigMap, EKS access entries or both approaches simultaneously are being used. If the ConfigMap approach is in effect, this gives you the ability to view access mappings or give persistent access to the cluster to arbitrary AWS identities if you have the ability to read/write ConfigMaps in the `kube-system` namespace in a privilege escalation/lateral movement scenario after compromising a container in the cluster. More detail on how this is done [here](https://securitylabs.datadoghq.com/articles/amazon-eks-attacking-securing-cloud-identities/).