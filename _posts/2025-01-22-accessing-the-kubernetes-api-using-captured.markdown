---
layout: post
title: "Accessing the Kubernetes API using captured credentials and HTTP clients"
date: 2025-01-22 18:00:00 +1100
author: Stephen Bradshaw
tags:
- kubernetes
- pentesting
- pentest
- container
- pod
- authentication
- privilege escalation
- lateral movement
- wget
- curl
- bad pods
- can-they
---

When attacking a Kubernetes cluster it is common to run into scenarios where you can obtain access to leaked Kubernetes credentials. This blog post will talk in detail about a number of different ways you can actually use those credentials to interact with the Kubernetes API, especially in scenarios where you might only have access to simple tools like wget or curl with which to make the connection.

Although the intent of this interaction will often be to move laterally or escalate privileges when done in the context of a pentest or security assessment, I wont be going into too much specific detail about the techniques for doing this, as this subject has been covered extensively before. Some good references are [here](https://www.schutzwerk.com/en/blog/kubernetes-privilege-escalation-01/), [here](https://www.upwind.io/feed/understanding-kubernetes-identities-part-2-escalation-paths) and [here](https://bishopfox.com/blog/kubernetes-pod-privilege-escalation).

Instead, what Im aiming to achieve with this post is to explain enough about how the Kubernetes API works to allow you to do the following:
* How you can identify a Kubernetes API credential and test that it works
* How you can use a captured credential with various different tools for authenticating to the Kubernetes API
* How you can explore the Kubernetes API, any extensions that may be present and your current permissions with simple tools
* Understand the difference between accessing namespaced and non-namespaced resources in the API when using simple tools

Having this knowledge makes it easier to use leaked credentials to make API calls to achieve any of your offensive goals.

I will start off this post by discussing some locations where Kubernetes credentials can be found and how to recognise them.

# Some sources of Kubernetes API credentials and how to recognise Kubernetes credentials

The "leakable" authentication credentials for Kubernetes generally fall into one of two high level categories:
* Authentication Bearer tokens, or;
* X509 Certificates and the associated private key

Some common locations in which you might find credentials include, (but are not limited to):
* In the filesystem of a running container/pod, including the service account token mounted by default (in file `/run/secrets/kubernetes.io/serviceaccount/token`)
* In files or other locations (e.g. URLs) referenced by environment variables in containers
* From the filesystem of the Kubernetes node host that runs containers/pods, in locations such as:
  - Kubectl config files (or backups) in user home directories e.g. `/home/user/.kube/config`
  - Kubectl config files in `/etc/kubernetes/` on control nodes
  - Service account tokens from all the running pods under `/var/lib/kubelet/pods/` (the [Bishop Fox can-they.sh script](https://github.com/BishopFox/badPods/blob/main/scripts/can-they.sh) helps in finding/testing these)
* Secret storage in the Kubernetes API - the secrets of type `kubernetes.io/service-account-token` are service account tokens
* Other locations where secrets can be leaked, e.g. emails, messaging systems like Slack, source code repos, configuration files, etc


Depending on the source, the credentials can be encoded or stored in various different ways. For example, the Kubernetes secrets system uses base64 to encode values, so when you extract secrets from this source you need to base64 decode to get things into their original format. Kubectl config files will also encode certificates using base64. For secrets that might be in source code or other random locations, any encoding scheme could be in use.

The first step when assessing whether you have a valid leaked credential is to convert the credentials into their standard format - either a token or a X509 cert and key.

The default Kubernetes service account tokens are [JWTs](https://jwt.io/). Heres an example one in its standard form:

```
eyJhbGciOiJSUzI1NiIsImtpZCI6ImVEMnJyekx4LVpFdm9BZnZ0Rmp4VmNoZExYNHBjaldfZjRJWEs4WU9HTlkifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzY5MDQyMjAxLCJpYXQiOjE3Mzc1MDYyMDEsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJ0aGVpYS03NDdjZGY1ZDRjLTJuY3dqIiwidWlkIjoiNjE5NjFiOTQtZTcyNS00N2VlLTg3NjUtMDFmYTk3OWY0Y2RjIn0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJkZWZhdWx0IiwidWlkIjoiYzRkYmFlZjEtZTk1My00NTg1LTk5MGQtN2M0NTA5MDhhYTQ2In0sIndhcm5hZnRlciI6MTczNzUwOTgwOH0sIm5iZiI6MTczNzUwNjIwMSwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6ZGVmYXVsdCJ9.rrYvp0yJaXW5pP0_4kNCwETcBoBW1pjIRwbDxazwO_dCM4TfWmgpuul0WCwWziRnWq-SbjH2rFUbjLMzlPGdM3EwtwcQ7__YyEv68_2GsnbBCgxwrUrxEi3rHCHRWmpEShfT_Lont2tHqzTurK1OW6lRKxBt_6qz0mP4GoxUQ3G7-74gsWuZ4yN0tjDfmPdx06RdrC_33TWMRmFAySBn69klqvbWY8aSwJUe3lLBxf1n9mahMn4YW6fZxtGELi5UUeR4CEmKaBlPjb6MHqgfxLdRpW0odN0g6Bt0oqAMby4z-Q7JvFPT8EGozgJQx66RuOm99adYDYzzgD9FtFjJIA
```

When decoded, the JWT content looks something like the following:

```
$ cat /run/secrets/kubernetes.io/serviceaccount/token | jwt -show -
Header:
{
    "alg": "RS256",
    "kid": "eD2rrzLx-ZEvoAfvtFjxVchdLX4pcjW_f4IXK8YOGNY"
}
Claims:
{
    "aud": [
        "https://kubernetes.default.svc.cluster.local"
    ],
    "exp": 1769042201,
    "iat": 1737506201,
    "iss": "https://kubernetes.default.svc.cluster.local",
    "kubernetes.io": {
        "namespace": "default",
        "pod": {
            "name": "theia-747cdf5d4c-2ncwj",
            "uid": "61961b94-e725-47ee-8765-01fa979f4cdc"
        },
        "serviceaccount": {
            "name": "default",
            "uid": "c4dbaef1-e953-4585-990d-7c450908aa46"
        },
        "warnafter": 1737509808
    },
    "nbf": 1737506201,
    "sub": "system:serviceaccount:default:default"
}
```

The `aud`, `iss` and `kubernetes.io` field in the claims of the token above indicate the token is for Kubernetes, and we have other fields such as `sub` to indicate the name of the associated service account identity.

Another token type you might run across is the [AWS authentication tokens](/2024/12/11/kubernetes-eks-authentication-internal-workings-abuses.html), unique to AWS EKS. These start with the value `k8s-aws-v1.` followed by a base64 encoded AWS STS `GetCallerIdentity` signed request as discussed [here](https://github.com/kubernetes-sigs/aws-iam-authenticator). Leaked versions of these are often less useful for reuse however, as they have a very limited token lifetime, meaning you need to be able to get to them very shortly after generation to make them usable.

For certificates and keys, these are most commonly found in PEM format, sometimes with additional base64 encoding when stored in kubectl config files (although the use of other encoding formats for storage could be possible).

The base64 encoded PEM files will start with characters like the following:

```
LS0tLS1CRUd
```

The unencoded PEM formats of the two files will generally start with first lines like the following:

For keys:

```
-----BEGIN RSA PRIVATE KEY-----
```


For certificates:
```
-----BEGIN CERTIFICATE-----
```


The keys for Kubernetes client authentication dont have any unique identifying characteristics that would distinguish them from keys used for other purposes, however the certificates do. Lets parse a Kubernetes client authentication X509 certificate to see which elements allow us to determine its used with Kubernetes.


```
$ openssl x509 -in client.pem -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1026249671020933914 (0xe3df8a37410731a)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = kubernetes
        Validity
            Not Before: Oct  4 01:27:56 2023 GMT
            Not After : Oct 14 00:24:21 2025 GMT
        Subject: O = system:masters, CN = kubernetes-admin
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:b4:fe:d2:38:42:81:75:f4:f9:37:19:54:40:a7:
                    d7:5d:5d:4f:d4:c9:1d:ac:66:ad:89:eb:06:6d:41:
                    92:21:b4:1c:a0:19:0e:f4:52:fa:33:ea:8e:a6:4f:
                    87:dc:da:71:75:8e:bc:a9:cb:4e:68:02:89:9e:87:
                    84:ad:37:74:30:09:18:cb:4f:da:21:85:2e:7c:75:
                    47:e8:50:b7:4e:dd:65:b2:55:55:c2:ea:80:62:85:
                    54:01:1f:f1:31:cb:1a:26:2b:c2:b8:c8:95:1d:7e:
                    21:f8:ac:1d:3a:92:9f:03:07:e7:c3:ae:20:17:ca:
                    6d:6d:3d:89:15:8b:21:f3:26:64:aa:6b:af:03:e5:
                    c5:d9:e8:d1:08:30:b2:6e:42:8d:f5:d4:6e:16:0d:
                    98:6d:3c:60:40:a9:f6:83:7c:d0:22:c1:23:a0:71:
                    9c:df:68:98:45:0e:cd:85:07:8b:07:4c:16:ba:21:
                    49:16:48:de:ab:ea:05:37:29:c3:c2:ff:07:ee:68:
                    ce:b6:25:bb:8d:3b:23:e8:0a:17:c2:70:0e:0b:ef:
                    a6:c6:b2:87:82:a7:90:07:e2:c4:dc:7d:7e:09:93:
                    94:91:b9:e3:d9:53:39:37:1f:0a:63:32:ed:81:b9:
                    16:54:87:a9:2a:b8:55:46:f6:6a:4f:d7:54:6e:d3:
                    43:0b
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Authority Key Identifier:
                38:89:CA:D4:D2:FC:D9:20:54:0F:69:DE:10:C0:85:07:9E:50:EA:13
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        1c:9d:08:70:a8:cc:2f:dd:a6:80:e9:91:50:d6:3e:7b:5e:89:
        f2:03:5e:47:aa:0d:c4:9e:61:1a:4e:ee:06:14:eb:a2:77:1c:
        66:0c:c0:6f:a0:b7:48:8c:c9:f9:b4:54:be:94:bb:ea:e0:54:
        86:05:ee:a3:2d:03:dc:68:b5:2b:e2:44:ad:a8:26:a0:b4:da:
        23:3d:db:c0:66:cc:25:9c:f9:e3:fc:b0:2c:70:59:1b:67:10:
        2f:60:1e:fd:8d:12:03:14:56:16:6b:db:11:39:b1:06:50:d7:
        77:88:6c:5d:46:24:25:fd:ae:a9:99:af:74:88:15:72:29:3b:
        f7:77:96:ad:72:a4:42:3e:e1:0e:56:8c:4d:3a:cd:b1:52:57:
        e9:49:0c:56:5f:d0:f3:cc:4b:07:a4:5f:7e:23:f0:e7:20:4f:
        67:03:bb:5e:7b:6b:45:48:24:ab:c3:81:bc:02:73:18:f1:07:
        24:11:84:87:27:dd:65:e8:4c:1e:dc:87:e4:eb:e5:bd:66:48:
        dd:96:70:ba:db:c4:12:7b:0e:20:c9:85:ed:77:f8:d5:e2:0d:
        74:2e:bc:32:99:51:78:8c:c9:43:ce:6e:29:a4:3e:f2:17:df:
        36:41:da:bf:a7:f7:8a:04:88:16:e6:15:a2:82:fd:ef:49:eb:
        26:b2:03:25
```


The fields here that identify this as a Kubernetes authentication certificate are the `Issuer` and `Subject` fields. The `Issuer` field shows that it was created by a Kubernetes server (or at least a CA whose identifier is `CN = kubernetes`) and the `Subject` field shows the user identity `CN = kubernetes-admin` and permissions `O = system:masters` granted by this certificate. For a deep dive into this subject you can go [here](/2023/11/15/kubernetes-auth-deep-dive.html).

Once you have verified that a certificate is used for Kubernetes, you can confirm that a given key file is associated with it by comparing the modulus values for both the key and certificate to make sure they match. For a key file `client.key` and certificate file `client.pem` commands such as the following will allow you to confirm this.

```
$ openssl rsa -in client.key -noout -check
RSA key ok
$ openssl rsa -in client.key -noout -modulus | openssl sha256
SHA256(stdin)= b004a1b8004444823bc72a6389310ec7823f8082770cb2cc63af124803c7d6a0
$ openssl x509 -in client.pem -noout -modulus | openssl sha256
SHA256(stdin)= b004a1b8004444823bc72a6389310ec7823f8082770cb2cc63af124803c7d6a0
```

Now lets talk about how to actually use the credentials, and how to figure out if they work by making API requests.

# The Kubernetes API and authentication

Given we are talking about abusing Kubernetes credentials its worthwhile giving a quick refresher on how we can communicate and authenitcate to the Kubernetes API. If you want more detail on these topics, I have two other posts on the topic [here discussing user and service account authentication](/2023/11/15/kubernetes-auth-deep-dive.html) and [here talking about EKS](/2024/12/11/kubernetes-eks-authentication-internal-workings-abuses.html).

The API server is a REST HTTPS server, so you can communicate with it using any regular HTTP client.  There are a few different ways to identify where the API server is from a pod inside a cluster, including the `KUBERNETES_SERVICE_HOST` and `KUBERNETES_SERVICE_PORT_HTTPS` environment variables and the `kubernetes.default.svc.cluster.local` default internal service (more [here](/2023/12/13/kubernetes-internal-service-discovery.html)). From outside a pod, you will have to rely on other discovery approaches to identify the Kubernetes API server.  In a number of the example commands below I will be communicating with the Kubernetes API server for my test cluster from its control node - in this particular case the server is accessible via HTTPS at the address `kubernetes:6443`.

In terms of authentication, generally speaking, you authenticate to it as a client either using:
* **TLS X509 client certificates** and the associated private key (normally for "user" credentials), or;
* **Bearer tokens** provided using the "Authorization" HTTP header (for service account credentials and other types of credentials like EKS tokens), 

**Note:** You also have the option to authenticate to Kubernetes using OIDC or webhooks, however exploitation of these authentication mechanisms requires a different approach and wont be discussed in this post. 

The server itself is authenticated to the clients (to prevent server impersonation or MITM attacks) via an X509 TLS server certificate. As a client you specify the CA certificate you have chosen to trust for that Kubernetes instance when connecting. This server CA certificate is usually the root CA in the same chain of trust used to verify the user certificates. 

Thus, authenticating to the API server means specifying the user cert and key or bearer token with each HTTPS request to the server, and optionally the CA certificate to verify the server identity. Lets look at how this is done using a few common tools.


# API access and authentication using kubectl 

Most examples of interaction with the API you will see discussed online are done using the `kubectl` command line tool. Generally, this tool will identify the location of the API server and the credentials to use either via a config file located at `~/.kube/config` or from the environment including the aforementioned environment variables or default services. If running from within a pod, there is also a service account token mounted at `/run/secrets/kubernetes.io/serviceaccount/token` that will be used for authentication as a bearer token if no other credentials are provided. The server certicate is also available to pods at the location `/run/secrets/kubernetes.io/serviceaccount/ca.crt`.

## kubectl authentication using command line parameters  

You can also directly specify these authentication details using command line options for kubectl, which can make it easier to do one-off command executions with specific credentials.

#### Certificate authentication

Heres an example of specifying TLS certificate authentication and the server location manually on the command line. In the following case, `ca.pem` is the PEM encoded version of the clusters CA certificate, `client.pem` is the PEM client certificate and `client.key` is the PEM client private key.

```
kubectl --certificate-authority=ca.pem --client-certificate=client.pem --client-key=client.key -s https://kubernetes:6443 auth whoami
```

If you dont care about verifying the server, you can ignore the server cert checking by specifying the `--insecure-skip-tls-verify=true` option instead of `--certificate-authority=ca.pem`.

I will also note here that if your existing kubectl config file already includes user certificate credentials, these command line options will NOT override the existing credentials, and you will see a message like the following. In this case, you can temporarily move the existing config file to a new location and make sure to specify the server address using `-s` to temporarily try the new creds.

```
Error in configuration:
* client-cert-data and client-cert are both specified for kubernetes-admin. client-cert-data will override.
* client-key-data and client-key are both specified for kubernetes-admin; client-key-data will override
```

We are using the `auth whoami` command here with `kubectl` as (under default configuration) its one that will always work if the credential is valid, giving us a way to test the credential is valid regardless of the permissions it may be assigned.


#### Bearer tokens

To manually specify a bearer token instead (like for a service account), you can use the `--token` switch, as in the following example:

```
kubectl --certificate-authority=ca.pem --token=<TOKEN_CONTENT> -s https://kubernetes:6443 auth whoami
```

The above involves specifying the actual token content on the command line. If you want to read the token from a file instead, you can do this via a subshell like so: 

```
kubectl --certificate-authority=ca.pem --token=$(cat /run/secrets/kubernetes.io/serviceaccount/token) -s https://kubernetes:6443 auth whoami
```

## kubectl authentication using captured tokens 

If you want to authenticate kubectl using a captured service account token **_without_** having to specify it on the command line each time as discussed above, you can create a config file (`~/.kube/config`) that defines it for you. This can be helpful when running tools that rely on kubectl under the hood, such as the handy [Bishop Fox can-they.sh script](https://github.com/BishopFox/badPods/blob/main/scripts/can-they.sh).

It took me a while to figure out how to do this, so I wanted to share the method here so I had a reference.

You can use the basic template below to create such a file, replacing the `<BASE64_ENCODED_CERT_DATA>` with an non-linewrapped base64 encoded version of the CA certificate, `<SERVER>` with the server address, and `<TOKEN_DATA>` with the service account token.

```
---
apiVersion: v1
kind: Config
clusters:
  - name: kubernetes
    cluster:
      certificate-authority-data: <BASE64_ENCODED_CERT_DATA>
      server: <SERVER>
contexts:
  - name: service_account@kubernetes
    context:
      cluster: kubernetes
      namespace: default
      user: service-account
users:
  - name: service-account
    user:
      token: <TOKEN_DATA>
current-context: service-account@kubernetes
```

Another option for creating these config files involves using [this script](https://github.com/stephenbradshaw/pentesting_stuff/blob/master/utilities/create_kubeconfig.sh) that will take a service account credential mount folder (with `ca.crt`, `namespace` and `token` files) as a parameter and create a config file for you. This script is intended to be used alongside the afore mentioned `can-they.sh` script to make it easier to abuse privileged service account tokens accessed from a node root volume.

Take this example use of the `can-they.sh` script to identify tokens that can read secrets in the `kube-system` namespace as reproduced from [the Bad Pods guide here](https://github.com/BishopFox/badPods/tree/main/manifests/everything-allowed):


```
root@aks-agentpool-76920337-vmss000000:/# ./can-they.sh -i "list secrets -n kube-system"
--------------------------------------------------------
Token Location: /host/var/lib/kubelet/pods/c888d3a8-743e-41dd-8464-91b3e6628174/volumes/kubernetes.io~secret/gatekeeper-admin-token-jmw8z/token
Command: kubectl auth can-i list secrets -n kube-system
yes

--------------------------------------------------------
Token Location: /host/var/lib/kubelet/pods/d13e311b-affa-4fad-b1c4-ec4f7817fd98/volumes/kubernetes.io~secret/metrics-server-token-ftxxd/token
Command: kubectl auth can-i list secrets -n kube-system
no

...omitted for brevity...
```


In this case, we take the base path from the token that has access (the one with the "yes" response), and run the script like so to generate a kubectl config file for you that you can then write to `~/.kube/config`:

```
create_kubeconfig.sh /host/var/lib/kubelet/pods/c888d3a8-743e-41dd-8464-91b3e6628174/volumes/kubernetes.io~secret/gatekeeper-admin-token-jmw8z/
```

Now that we have discussed how to use `kubectl` with captured credentials, lets look at how to use the API in situations where you dont have access to this tool.

# API access and authentication using alternate clients

While `kubectl` is generally the best way to make Kubernetes API calls from the command line, you may find yourself in situations where the tool is not available (such as in a compromised pod), but you do have access to other HTTP client tools such as `curl` or `wget`. Lets look at how we can use tools such as these to access the Kubernetes API. 

We will first demonstrate the basics of how to do client certificate and bearer token authentication with `curl` and `wget`, and we will then examine how to usefully interact with the API with direct HTTP/REST calls as opposed to the more familiar kubectl commands.

In the examples below, the Kubernetes API server I will be accessing is my test cluster at the address `https://kubernetes:6443`, you will need to modify as required for your target environment.

## curl and API authentication

Heres an example of client certifiate authentication with curl. As with the `kubectl` example above, the files referenced are `ca.pem` which is the clusters CA certificate in PEM form, `client.pem` which is the client certificate in PEM format and `client.key` which is the associated private key in PEM form.
```
curl -s --cacert ca.pem --cert client.pem --key client.key  https://kubernetes:6443/api
```

In the above `-k` can be substituted for `--cacert ca.pem` if you dont want to verify the server.


Heres an example of authenticating using a bearer token, read from a file at `/run/secrets/kubernetes.io/serviceaccount/token`. You can substitute the subshell for the actual token content if you prefer.
```
curl -s --cacert ca.pem -H "Authorization: Bearer $(cat /run/secrets/kubernetes.io/serviceaccount/token)"  https://kubernetes:6443/api
```

Both of the above commands are making a REST API call to the base path of the main Kubernets api - `/api`. We are using this path for our first request as it is likely the simplest way to determine if a credential is valid or not, regardless of permissions. 

A valid credential will return a response looking something like the following:
```
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.10.214:6443"
    }
  ]
}
```

An invalid credential will return a response looking something like this:
```
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```


We will look at more specific API call examples later.


## wget and API authentication

Heres an example of client certifiate authentication with wget. As with the `kubectl` example above, the files referenced are `ca.pem` which is the clusters CA certificate in PEM form, `client.pem` which is the client certificate in PEM format and `client.key` which is the associated private key in PEM form.

```
wget -q --ca-cert=ca.pem --certificate=client.pem --private-key=client.key  https://kubernetes:6443/api -O -
```

In the above `--no-check-certificate` can be substituted for `--ca-cert=ca.pem` if you dont want to verify the server. The `-O -` at the end sends the response to STDOUT instead of to a file.

Heres an example of authenticating using a bearer token, read from a file at `/run/secrets/kubernetes.io/serviceaccount/token`. You can substitute the subshell for the actual token content if you prefer. Pretty similar to curl.

```
wget -q --ca-cert=ca.pem -H "Authorization: Bearer $(cat /run/secrets/kubernetes.io/serviceaccount/token)"  https://kubernetes:6443/api -O -
```


## Identifying the user and permissions

Now lets look at how to identify the user/permissions associated with a credential using regular web requests. In the kubectl tool, this is done using `kubectl auth whoami` (shown earlier) and `kubectl auth can-i <verb> <resource>`, which under the hood use `selfsubjectreviews` in the `authentication.k8s.io` apiGroup and `selfsubjectaccessreviews` in the `authorization.k8s.io` apiGroup respectively.

A `whoami` is done using a POST request to the API, and looks like this:
```
$ curl -s --cacert ca.pem --cert client.pem --key client.key https://kubernetes:6443/apis/authentication.k8s.io/v1/selfsubjectreviews -H "Content-Type: application/json" -d '{}'
{
  "kind": "SelfSubjectReview",
  "apiVersion": "authentication.k8s.io/v1",
  "metadata": {
    "creationTimestamp": "2024-12-18T06:42:02Z"
  },
  "status": {
    "userInfo": {
      "username": "kubernetes-admin",
      "groups": [
        "system:masters",
        "system:authenticated"
      ]
    }
  }
}
```

A `can-i get pods` also uses a POST request, and looks like this:

```
$ curl -s --cacert ca.pem --cert client.pem --key client.key https://kubernetes:6443/apis/authorization.k8s.io/v1/selfsubjectaccessreviews -H "Content-Type: application/json" -d '{"kind":"SelfSubjectAccessReview","apiVersion":"authorization.k8s.io/v1","spec":{"resourceAttributes":{"verb":"get", "resource":"pods"}}}'
{
  "kind": "SelfSubjectAccessReview",
  "apiVersion": "authorization.k8s.io/v1",
  "metadata": {
    "creationTimestamp": null,
    "managedFields": [
      {
        "manager": "curl",
        "operation": "Update",
        "apiVersion": "authorization.k8s.io/v1",
        "time": "2024-12-18T06:51:20Z",
        "fieldsType": "FieldsV1",
        "fieldsV1": {
          "f:spec": {
            "f:resourceAttributes": {
              ".": {},
              "f:namespace": {},
              "f:resource": {},
              "f:verb": {}
            }
          }
        }
      }
    ]
  },
  "spec": {
    "resourceAttributes": {
      "namespace": "default",
      "verb": "get",
      "resource": "pods"
    }
  },
  "status": {
    "allowed": true
  }
}
```

The above is technically querying for permissions across all namespaces (the `-A` switch in kubectl). To query for permissions within a given namespace, for example "default", you can add the `"namespace":"default"` key and value to the `resourceAttributes` object in the above POST request JSON data.

Fully exploring your available permissions requires that you also understand the resources available, which you can enumerate once you understand the API URL path structure.


## Kubernetes API paths

Without the kubectl command line tool to translate commands for us, we do need to understand a few things about how the REST paths in the API are structured in order to use other HTTP client tools to interact with Kubernetes.

Some of the basic points that are useful to understand here when it comes to manually exploring/using the API are:
* There are a number of components of the API that are by default accessible to all authenticated users as discussed [here](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#discovery-roles). This includes paths that allow you to identify details about your current user and their permissions, paths that allow you to identify API resources in use and a path that gives you some basic cluster information.
* The default output format for responses from the API is `application/json`. Default alternate output formats are `application/yaml` and `application/vnd.kubernetes.protobuf`, which can be accessed by adding an appropriate "Accept" header to your request e.g. `-H 'Accept: application/yaml'`
* The Kubernetes API is extensible, allowing new categories of resources to be added to the API. The "extended" resources usually sit under the path `/apis/` in their own categories. These will have paths that match their `apiGroup` and `apiVersion` as you might see specified in YAML resource definitions for that resource type. There are a lot of "extended" resources that come by default and form core parts of the Kubernetes API. We will look at examples later.
* Standard APIs, belonging to the `v1` `apiVersion` type are under the path `/api/v1/`

Theres some official documentation [here](https://kubernetes.io/docs/reference/using-api/api-concepts/) that describes the API structure that you can look at for more info, but I'll try and explain the most imporant aspects of this by example below.

In the following section Im going to explore the API via example using `curl`. I will be using `grep` (and sometimes `cut` for neater output layout) to help pull out the most relevant information from the responses where possible, although this does not always work well on the JSON data returned. Using something that designed to query JSON like `jq` is definitely preferrable, however this is normally not present on many pods so I wont use it in these examples either, so you can get an idea of how to do this yourself in a restricted environment.


### Exploring available API resources

One of the first things thats useful to do is to find out the resource types supported on the server. Heres how you can explore this:

This will list the standard resource types:
```
$ curl -s --cacert ca.pem --cert client.pem --key client.key  https://kubernetes:6443/api/v1/ | grep '"name"' | cut -d'"' -f 4
bindings
componentstatuses
configmaps
endpoints
events
limitranges
namespaces
namespaces/finalize
namespaces/status
nodes
nodes/proxy
nodes/status
persistentvolumeclaims
persistentvolumeclaims/status
persistentvolumes
persistentvolumes/status
pods
pods/attach
pods/binding
pods/ephemeralcontainers
pods/eviction
pods/exec
pods/log
pods/portforward
pods/proxy
pods/status
podtemplates
replicationcontrollers
replicationcontrollers/scale
replicationcontrollers/status
resourcequotas
resourcequotas/status
secrets
serviceaccounts
serviceaccounts/token
services
services/proxy
services/status
```

This will list the extended apiGroups
```
$ curl -s --cacert ca.pem --cert client.pem --key client.key  https://kubernetes:6443/apis/ | grep '"name"' | cut -d'"' -f 4
apiregistration.k8s.io
apps
events.k8s.io
authentication.k8s.io
authorization.k8s.io
autoscaling
batch
certificates.k8s.io
networking.k8s.io
policy
rbac.authorization.k8s.io
storage.k8s.io
admissionregistration.k8s.io
apiextensions.k8s.io
scheduling.k8s.io
coordination.k8s.io
node.k8s.io
discovery.k8s.io
flowcontrol.apiserver.k8s.io
crd.projectcalico.org
security.istio.io
argoproj.io
extensions.istio.io
install.istio.io
telemetry.istio.io
networking.istio.io
```

This will list the available apiVersions for a given apiGroup `authorization.k8s.io`

```
$ curl -s --cacert ca.pem --cert client.pem --key client.key  https://kubernetes:6443/apis/authorization.k8s.io/ | grep '"version"' | cut -d'"' -f 4
v1
v1
```


Given an apiVersion of `v1` for apiGoup `authorization.k8s.io`, this is how we get the resources for that version of the API:
```
$ curl -s --cacert ca.pem --cert client.pem --key client.key  https://kubernetes:6443/apis/authorization.k8s.io/v1/| grep '"name"' | cut -d'"' -f 4
localsubjectaccessreviews
selfsubjectaccessreviews
selfsubjectrulesreviews
subjectaccessreviews
```

If your want to list all the resoures of a particular type, for example pods, you can do like the following:

```
curl -s --cacert ca.pem --cert client.pem --key client.key  https://kubernetes:6443/api/v1/pods | grep '"name"'
        "name": "argocd-application-controller-0",
            "name": "argocd-application-controller",
            "name": "workload-socket",
[SNIP]
```

Our trick of simply grepping for the "name" field as used in previous examples doesnt work as well with more complex responses, but in the case above you can tell the pod name by picking the value thats indented furthest to the left.

The response will also include the full details of each resource if you want to look in detail - as long as you have the permissions to list that resource type in all namespaces. If that resource type is namespaced and you only have permissions to list that resource type in a particular namespace you can go to the following section to see how we can interact with resources in a namespaced context. This is also an important concept to understand if you want to access resources individually, which is necessary if you want to edit or delete them.


### Identifying namespaced resources

Kubernetes uses namespaces as a mechanism for isolating a group of resources within a cluster. Within this framework, there are types of resources that can be separated by namespace ("namespaced" resources) and those that cannot and exist cluster wide. Instances of resources are accessed differently in the API depending on whether their resource type is namespaced or not, so we need to know this fact about a resource type before we can access it in the API.

The following demonstrates how you can identify namespace status for standard resource types - we basically just modify the grep statement filtering result data to list both the `name` and `namespaced` parameters in the response when listing resource types: 

```
$ curl -s --cacert ca.pem --cert client.pem --key client.key  https://kubernetes:6443/api/v1/ | grep '"name"\|"namespaced"'
      "name": "bindings",
      "namespaced": true,
      "name": "componentstatuses",
      "namespaced": false,
      "name": "configmaps",
      "namespaced": true,
      "name": "endpoints",
      "namespaced": true,
      "name": "events",
      "namespaced": true,
      "name": "limitranges",
      "namespaced": true,
      "name": "namespaces",
      "namespaced": false,
      "name": "namespaces/finalize",
      "namespaced": false,
      "name": "namespaces/status",
      "namespaced": false,
      "name": "nodes",
      "namespaced": false,
      "name": "nodes/proxy",
      "namespaced": false,
      "name": "nodes/status",
      "namespaced": false,
      "name": "persistentvolumeclaims",
      "namespaced": true,
      "name": "persistentvolumeclaims/status",
      "namespaced": true,
      "name": "persistentvolumes",
      "namespaced": false,
      "name": "persistentvolumes/status",
      "namespaced": false,
      "name": "pods",
      "namespaced": true,
      "name": "pods/attach",
      "namespaced": true,
      "name": "pods/binding",
      "namespaced": true,
      "name": "pods/ephemeralcontainers",
      "namespaced": true,
      "name": "pods/eviction",
      "namespaced": true,
      "name": "pods/exec",
      "namespaced": true,
      "name": "pods/log",
      "namespaced": true,
      "name": "pods/portforward",
      "namespaced": true,
      "name": "pods/proxy",
      "namespaced": true,
      "name": "pods/status",
      "namespaced": true,
      "name": "podtemplates",
      "namespaced": true,
      "name": "replicationcontrollers",
      "namespaced": true,
      "name": "replicationcontrollers/scale",
      "namespaced": true,
      "name": "replicationcontrollers/status",
      "namespaced": true,
      "name": "resourcequotas",
      "namespaced": true,
      "name": "resourcequotas/status",
      "namespaced": true,
      "name": "secrets",
      "namespaced": true,
      "name": "serviceaccounts",
      "namespaced": true,
      "name": "serviceaccounts/token",
      "namespaced": true,
      "name": "services",
      "namespaced": true,
      "name": "services/proxy",
      "namespaced": true,
      "name": "services/status",
      "namespaced": true,
```

Identifying whether "extended" resources are namespaced or not is done in a similar manner. Heres an example for identifying namespaced resources from an apiGroup in the `authorization.k8s.io` apiGroup for apiVersion `v1`:

```
$ curl -s --cacert ca.pem --cert client.pem --key client.key  https://kubernetes:6443/apis/authorization.k8s.io/v1/ | grep '"name"\|"namespaced"'
      "name": "localsubjectaccessreviews",
      "namespaced": true,
      "name": "selfsubjectaccessreviews",
      "namespaced": false,
      "name": "selfsubjectrulesreviews",
      "namespaced": false,
      "name": "subjectaccessreviews",
      "namespaced": false,
```

### Accessing non-namespaced resources 

Resources that are NOT namespaced can be individually accessed in a very straightforward manner. 

The URL patterns to use for standard resource types are:
* `/api/<API_VERSION>/<RESOURCE_TYPE>/` to list all the resources, and;
* `/api/<API_VERSION>/<RESOURCE_TYPE>/<RESOURCE_NAME>` to access an individual resource by name 

**Note:** Its generally fine to either include or exclude a trailing slash for these URL paths, which is why Ive alternated the use of this in these examples.

Lets look at examples.

Based on the output we viewed previously, the `componentstatuses` resource type is not namespaced. Let's list these resources by name:

```
$ curl -s --cacert ca.pem --cert client.pem --key client.key  https://kubernetes:6443/api/v1/componentstatuses/ | grep '"name"'
        "name": "controller-manager",
        "name": "scheduler",
        "name": "etcd-0",
```

Now lets access the `controller-manager` named resource:

```
$ curl -s --cacert ca.pem --cert client.pem --key client.key  https://kubernetes:6443/api/v1/componentstatuses/controller-manager
{
  "kind": "ComponentStatus",
  "apiVersion": "v1",
  "metadata": {
    "name": "controller-manager",
    "creationTimestamp": null
  },
  "conditions": [
    {
      "type": "Healthy",
      "status": "True",
      "message": "ok"
    }
  ]
}
```


For extended non-namespaced resources, the URL patterns to use are:
* `/apis/<API_GROUP>/<API_VERSION>/<RESOURCE_TYPE>/` to list all the resources, and;
* `/apis/<API_GROUP>/<API_VERSION>/<RESOURCE_TYPE>/<RESOURCE_NAME>` to access an individual resource by name 



Heres an example of fetching the list of non namespaced `clusterroles` (`rbac.authorization.k8s.io` apiGroup of apiVersion `v1`):

```
$ curl -s --cacert ca.pem --cert client.pem --key client.key  https://kubernetes:6443/apis/rbac.authorization.k8s.io/v1/clusterroles/ | grep '"name"'
        "name": "admin",
        "name": "argocd-application-controller",
        "name": "argocd-applicationset-controller",
        "name": "argocd-server",
        "name": "calico-kube-controllers",
        "name": "calico-node",
        "name": "cluster-admin",
        "name": "edit",
        "name": "istio-reader-clusterrole-istio-system",
        "name": "istiod-clusterrole-istio-system",
        "name": "istiod-gateway-controller-istio-system",
        "name": "kiali",
        "name": "kiali-viewer",
        "name": "kubeadm:get-nodes",
        "name": "prometheus",
        "name": "system:aggregate-to-admin",
        "name": "system:aggregate-to-edit",
        "name": "system:aggregate-to-view",
        "name": "system:auth-delegator",
        "name": "system:basic-user",
        "name": "system:certificates.k8s.io:certificatesigningrequests:nodeclient",
        "name": "system:certificates.k8s.io:certificatesigningrequests:selfnodeclient",
        "name": "system:certificates.k8s.io:kube-apiserver-client-approver",
        "name": "system:certificates.k8s.io:kube-apiserver-client-kubelet-approver",
        "name": "system:certificates.k8s.io:kubelet-serving-approver",
        "name": "system:certificates.k8s.io:legacy-unknown-approver",
        "name": "system:controller:attachdetach-controller",
        "name": "system:controller:certificate-controller",
        "name": "system:controller:clusterrole-aggregation-controller",
        "name": "system:controller:cronjob-controller",
        "name": "system:controller:daemon-set-controller",
        "name": "system:controller:deployment-controller",
        "name": "system:controller:disruption-controller",
        "name": "system:controller:endpoint-controller",
        "name": "system:controller:endpointslice-controller",
        "name": "system:controller:endpointslicemirroring-controller",
        "name": "system:controller:ephemeral-volume-controller",
        "name": "system:controller:expand-controller",
        "name": "system:controller:generic-garbage-collector",
        "name": "system:controller:horizontal-pod-autoscaler",
        "name": "system:controller:job-controller",
        "name": "system:controller:namespace-controller",
        "name": "system:controller:node-controller",
        "name": "system:controller:persistent-volume-binder",
        "name": "system:controller:pod-garbage-collector",
        "name": "system:controller:pv-protection-controller",
        "name": "system:controller:pvc-protection-controller",
        "name": "system:controller:replicaset-controller",
        "name": "system:controller:replication-controller",
        "name": "system:controller:resourcequota-controller",
        "name": "system:controller:root-ca-cert-publisher",
        "name": "system:controller:route-controller",
        "name": "system:controller:service-account-controller",
        "name": "system:controller:service-controller",
        "name": "system:controller:statefulset-controller",
        "name": "system:controller:ttl-after-finished-controller",
        "name": "system:controller:ttl-controller",
        "name": "system:coredns",
        "name": "system:discovery",
        "name": "system:heapster",
        "name": "system:kube-aggregator",
        "name": "system:kube-controller-manager",
        "name": "system:kube-dns",
        "name": "system:kube-scheduler",
        "name": "system:kubelet-api-admin",
        "name": "system:monitoring",
        "name": "system:node",
        "name": "system:node-bootstrapper",
        "name": "system:node-problem-detector",
        "name": "system:node-proxier",
        "name": "system:persistent-volume-provisioner",
        "name": "system:public-info-viewer",
        "name": "system:service-account-issuer-discovery",
        "name": "system:volume-scheduler",
        "name": "view",
```


Here is an example of accessing the `cluster-admin` clusterrole:
```
$ curl -s --cacert ca.pem --cert client.pem --key client.key  https://kubernetes:6443/apis/rbac.authorization.k8s.io/v1/clusterroles/cluster-admin
{
  "kind": "ClusterRole",
  "apiVersion": "rbac.authorization.k8s.io/v1",
  "metadata": {
    "name": "cluster-admin",
    "uid": "880b3e9c-a543-45bd-b3b6-f00b128395be",
    "resourceVersion": "72",
    "creationTimestamp": "2023-10-04T01:33:05Z",
    "labels": {
      "kubernetes.io/bootstrapping": "rbac-defaults"
    },
    "annotations": {
      "rbac.authorization.kubernetes.io/autoupdate": "true"
    },
    "managedFields": [
      {
        "manager": "kube-apiserver",
        "operation": "Update",
        "apiVersion": "rbac.authorization.k8s.io/v1",
        "time": "2023-10-04T01:33:05Z",
        "fieldsType": "FieldsV1",
        "fieldsV1": {
          "f:metadata": {
            "f:annotations": {
              ".": {},
              "f:rbac.authorization.kubernetes.io/autoupdate": {}
            },
            "f:labels": {
              ".": {},
              "f:kubernetes.io/bootstrapping": {}
            }
          },
          "f:rules": {}
        }
      }
    ]
  },
  "rules": [
    {
      "verbs": [
        "*"
      ],
      "apiGroups": [
        "*"
      ],
      "resources": [
        "*"
      ]
    },
    {
      "verbs": [
        "*"
      ],
      "nonResourceURLs": [
        "*"
      ]
    }
  ]
}
```


### Accessing namespaced resources

Resources that are namespaced need to be accessed individually from within that namespaces path in the REST API.

The URL patterns to use for standard resource types are:
* `/api/<API_VERSION>/namespaces/<NAMESPACE>/<RESOURCE_TYPE>/` to list all the resources, and;
* `/api/<API_VERSION>/namespaces/<NAMESPACE>/<RESOURCE_TYPE>/<RESOURCE_NAME>` to access an individual resource by name 

Lets look at an example for the standard resource type of `serviceaccounts` in the `default` namespace. We can list these like so:

```
$ curl -s --cacert ca.pem --cert client.pem --key client.key  https://kubernetes:6443/api/v1/namespaces/default/serviceaccounts | grep '"name"'
        "name": "default",
```

If we want to access the `default` serviceaccount in this namespace, it is done like so:

```
$ curl -s --cacert ca.pem --cert client.pem --key client.key  https://kubernetes:6443/api/v1/namespaces/default/serviceaccounts/default
{
  "kind": "ServiceAccount",
  "apiVersion": "v1",
  "metadata": {
    "name": "default",
    "namespace": "default",
    "uid": "c4dbaef1-e953-4585-990d-7c450908aa46",
    "resourceVersion": "290",
    "creationTimestamp": "2023-10-04T01:33:12Z"
  }
}
```

For extended resources, the URL patterns to use for namespaced resources are:
* `/apis/<API_GROUP>/<API_VERSION>/namespaces/<NAMESPACE>/<RESOURCE_TYPE>/` to list all the resources, and;
* `/apis/<API_GROUP>/<API_VERSION>/namespaces/<NAMESPACE>/<RESOURCE_TYPE>/<RESOURCE_NAME>` to access an individual resource by name 


Lets look at an example of fetching the list of `roles` resources (`rbac.authorization.k8s.io` apiGroup of apiVersion `v1`) in the `kube-system` namespace.

```
$ curl -s --cacert ca.pem --cert client.pem --key client.key  https://kubernetes:6443/apis/rbac.authorization.k8s.io/v1/namespaces/kube-system/roles/ | grep '"name"'
        "name": "extension-apiserver-authentication-reader",
        "name": "kube-proxy",
        "name": "kubeadm:kubelet-config",
        "name": "kubeadm:nodes-kubeadm-config",
        "name": "system::leader-locking-kube-controller-manager",
        "name": "system::leader-locking-kube-scheduler",
        "name": "system:controller:bootstrap-signer",
        "name": "system:controller:cloud-provider",
        "name": "system:controller:token-cleaner",
```


Now lets individually access the `kube-proxy` role from the list above:

```
$ curl -s --cacert ca.pem --cert client.pem --key client.key  https://kubernetes:6443/apis/rbac.authorization.k8s.io/v1/namespaces/kube-system/roles/kube-proxy
{
  "kind": "Role",
  "apiVersion": "rbac.authorization.k8s.io/v1",
  "metadata": {
    "name": "kube-proxy",
    "namespace": "kube-system",
    "uid": "b47d3e52-a934-4b4e-a0d2-5f300dbd806e",
    "resourceVersion": "224",
    "creationTimestamp": "2023-10-04T01:33:07Z",
    "managedFields": [
      {
        "manager": "kubeadm",
        "operation": "Update",
        "apiVersion": "rbac.authorization.k8s.io/v1",
        "time": "2023-10-04T01:33:07Z",
        "fieldsType": "FieldsV1",
        "fieldsV1": {
          "f:rules": {}
        }
      }
    ]
  },
  "rules": [
    {
      "verbs": [
        "get"
      ],
      "apiGroups": [
        ""
      ],
      "resources": [
        "configmaps"
      ],
      "resourceNames": [
        "kube-proxy"
      ]
    }
  ]
}
```

Now that you hopefully understand how to individually access all of the possible different Kubernetes resources, lets look at a more complex example of how to do "kubectl" style `apply` operations using `curl`.

### Applying and deleting a YAML resource 

Another thing you might want to do is to make changes to the Kubernetes cluster via "applying" a YAML resource file similar to how you would with kubectl, e.g. `kubectl apply -f basicsa.yaml`, as well as deleting (`kubectl delete -f basicsa.yaml`)

Lets look at this via the medium of an example.

Take the following very simple YAML resource file `basicsa.yaml` which creates the `basic` serviceaccount in the `default` namespace.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: basic
  namespace: default
```


This is how you would apply the resource via curl:
```
curl --cacert ca.pem --cert client.pem --key client.key -X POST https://kubernetes:6443/api/v1/namespaces/default/serviceaccounts  -H "Content-Type: application/yaml" --data-binary @/tmp/basicsa.yaml 
```


And this is how you can delete the resource. If you're interested there is some more information about the specifics of the delete options used [here](https://kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/)
```
curl --cacert ca.pem --cert client.pem --key client.key -X DELETE https://kubernetes:6443/api/v1/namespaces/default/serviceaccounts/basic  -H "Content-Type: application/json" -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}'
```


# Conclusion

Hopefully this post gave you a good understanding of how the Kubernetes API works so you can use it properly even without friendly command line clients like `kubectl`.
