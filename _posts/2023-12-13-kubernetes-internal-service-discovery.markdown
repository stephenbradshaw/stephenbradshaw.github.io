---
layout: post
title:  "Kubernetes Internal Service Discovery"
date:   2023-12-13 19:20:00 +1100
author: Stephen Bradshaw
tags:
- Kubernetes
- pentesting
- red team
- service enumeration
- network enumeration
- enumeration
- discovery
- container
- service
- nmap
- istio
- pods
- services
- hacker-container
- JWT
- iPython
---

This blog post talks about methods you can use from within a compromised container to discover additional accessible network services within a Kubernetes cluster. The post assumes you have obtained code execution in the compromised container, and want to use that access to attack other internal services within the cluster. 

Some approaches discussed will require access to particular tools inside the container. You may be able to just download and run these tools if you have Internet access and sufficient permissions in the container, however I will try and suggest alternate approaches if I know of them.

I will also describe a number of internal Kubernetes components that can be used to help you discover other services to attack, as well as how the discovery process can be complicated by service meshes like [Istio](https://istio.io/).

**Note**: In a Kubernetes post exploitation scenario there are other potential attack vectors you can use such as container escapes and accessing other internal networks outside the cluster that will not be discussed in this post.

# Pods and services

The two main types of Kubernetes components you will be primarily interested in when looking to attack other network accessible applicatiions within the cluster are [pods](https://kubernetes.io/docs/concepts/workloads/pods/) and [services](https://kubernetes.io/docs/concepts/services-networking/service/).

Pods are groups of one or more running containers, and this is where the internal networked applications you want to attack will be running. Pods have internal cluster IP addresses associated with them, and have one or more exposed network ports you can use to communicate with the networked applications. 

Services are friendly ways of exposing applications running on one or more pods. These again have cluster IP addresses and one or more exposed ports, as well as various associated DNS records configured in the clusters DNS resolver. Accessing the application using a service vs accessing it directly at the pod are usually similar, however the services have additional discoverability features that can be useful to us. 

The IP addresses used for pods will usually be in a seperate private network range from those of services.

Now that we have established what we are looking for, lets cover some of the methods we can use to identify these components. 

# Situational awareness

To start, its useful to collect some specific information from the pod you have compromised that will help in the coming steps. The in-container examples I will be showing in this post are from the shell of a [hacker-container](https://github.com/madhuakula/hacker-container) pod running in my test cluster, as a demonstration of having code execution in a container in the cluster.

In order to facilitate later discovery steps, we are looking for information such as cluster IP addresses and ports, the namespace of the pod, the API server address and any secrets or Kubernetes API authentication tokens.

First, check the environment variables. These often contain IP addresses and ports of other services in the cluster that can act as a starting point for discovery. Of particular interest are variables containing the string `KUBERNETES`, which point to the Kubernetes API service. See the below example from a pod within my test cluster.

```
bash-5.1# env | grep KUBERNETES
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PORT=443
```

Other good sources of cluster IP addresses are files `/etc/hosts` (giving your pod's local IP address, which you can also obtain from the `ip` or `ifconfig` commands) and `/etc/resolv.conf` (giving the clusters DNS server address and the DNS search domains which infers the pod's namespace).

Here are examples of those files, revealing my pod address `172.16.7.159`, the pod name `hacker-container`, the nameserver `10.96.0.10` and the namespace of my pod `default` (from search entry `default.svc.cluster.local`):

```
bash-5.1# cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
172.16.7.159	hacker-container

bash-5.1# cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local thezoo.local
nameserver 10.96.0.10
options ndots:5
```


You can also look at the claims in the pod's service account token to gather information. This file provides the default credentials the pod will use to query the Kubernetes API server, and is located by default in local file `/run/secrets/kubernetes.io/serviceaccount/token`. You can decode the token's second section to extract it's claims. From this we can see fields which point to the API server, and tell you the pod name and the service account name and more.
```
bash-5.1# cat /run/secrets/kubernetes.io/serviceaccount/token | cut -d '.' -f 2 | base64 -d
{"aud":["https://kubernetes.default.svc.cluster.local"],"exp":1733371260,"iat":1701835260,"iss":"https://kubernetes.default.svc.cluster.local","kubernetes.io":{"namespace":"default","pod":{"name":"hacker-container","uid":"9beba93e-df52-4154-927c-3c30ba0f3bcf"},"serviceaccount":{"name":"default","uid":"c4dbaef1-e953-4585-990d-7c450908aa46"},"warnafter":1701838867},"nbf":1701835260,"sub":"system:serviceaccount:default:default"}
```

Also useful is `/run/secrets/kubernetes.io/serviceaccount/namespace`, which tells you the namespace in which your pod is running.

```
bash-5.1# cat /run/secrets/kubernetes.io/serviceaccount/namespace
default
```

Its also worthwhile to check the mounted filesystems in the container using the `mount` command, to confirm that your container doesn't have any other accessible secrets that can be used to access the API server. The default service account secret mount is at `/run/secrets/kubernetes.io/serviceaccount`, any other mounted filesystems with `secret` or `serviceaccount` in the name are worth looking at. Additionally, all the normal local enumeration techniques also apply - e.g. look in application configuration files in the pod.



# The Kubernetes API server

Your first port of call to get details about pods and services in a Kubernetes cluster is the Kubernetes API. This API is the primary means by which Kubernetes clusters are configured, managed and audited, and it is definitely the easiest way to get comprehensive information on pods and services in the cluster. You do have to be authorised to use it however. 

The previous section mentions a few ways in which you can locate the API server, which is also usually accessible from within the cluster by a default service at `https://kubernetes.default.svc.cluster.local/`. 

So its quite easy to find where the API server is, but getting useful information out of it will usually be another problem. 

By default, in more recent Kubernetes varions at least, the in built Kubernetes service account represented by the token we showed in the previous section wont have any useful access to the Kubernetes API. You should still test the access this token gives you just in case additional rights have been added however - the next section will go into more detail on how this can be done. 

As a simple example, in the case below, the service account token in this pod **has** been allocated extra rights, and it **can** list resources in its own (`default`) namespace...

```
bash-5.1# kubectl --token "$(cat /run/secrets/kubernetes.io/serviceaccount/token)" get pods -o wide
NAME                              READY   STATUS    RESTARTS      AGE   IP             NODE                         NOMINATED NODE   READINESS GATES
hacker-container                  2/2     Running   3 (48d ago)   48d   172.16.7.159   kubecontainer.thezoo.local   <none>           <none>
haproxy-77766c8866-2hf44          2/2     Running   0             46d   172.16.7.182   kubecontainer.thezoo.local   <none>           <none>
ichnaea-5566dd5bd7-hxccb          2/2     Running   2 (48d ago)   48d   172.16.7.145   kubecontainer.thezoo.local   <none>           <none>
mitmproxy-7658d6f68d-d67bh        2/2     Running   0             46d   172.16.7.184   kubecontainer.thezoo.local   <none>           <none>
theia-d88b7b7b4-kd99k             2/2     Running   0             28d   172.16.7.130   kubecontainer.thezoo.local   <none>           <none>
```

The token **cannot** access resources cluster wide however:

```
bash-5.1# kubectl --token "$(cat /run/secrets/kubernetes.io/serviceaccount/token)" get pods -o wide --all-namespaces
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:default:default" cannot list resource "pods" in API group "" at the cluster scope
```

In a coming section, we will look at another way you might be able to get Kubernetes API access if you are inside an AWS EKS cluster, but first let's look in some more detail at various ways in which you can use the API server to get useful information on pods and services. 


# Extracting pod and service information from the Kubernetes API 

The easiest way to query the Kubernetes API is use the kubectl command line tool. If the tool is not already on the container which you have compromised (and it probably won't be), you can easily download it if your pod has outbound Internet access. 

You can get the version of the Kubernetes API server unauthenticated at `https://kubernetes.default.svc.cluster.local/version`, and you can get the latest available version of Kubernetes from [here](https://kubernetes.io/releases/) (at the time of writing it is `1.28.4`). 

Once you know what version of the tool you want you can download the tool for the appropriate CPU architecture (`uname -a` in your container) from one of the following URLs. Just replace `[VERSION]` in the URL with the appropriate Kubernetes version:
* **ARM64** `https://storage.googleapis.com/kubernetes-release/release/v[VERSION]/bin/linux/arm64/kubectl`
* **AMD64** `https://storage.googleapis.com/kubernetes-release/release/v[VERSION]/bin/linux/amd64/kubectl`

Then, to get a list of all the running pods in the cluster, with IP addresses, you can run something like the following:

```
kubectl get pods --all-namespaces -o wide
```

The above will just list out the IP addresses of the pods, if you want to get the exposed ports too, you will need to use a more verbose output format. There are various query options using jsonpath in the kubectl tool you can use to get individual fields details, but when I need multiple specific fields I usually just dump all output to JSON and parse it offline. A later section of this post will show an easy way to extract the required information from the JSON output. You can get all the pod details as JSON like so. 

```
kubectl get pods --all-namespaces -o json > /tmp/pod_details.json
```

Services are a little easier - the below will get both IP address and port details for all services in the cluster.

```
kubectl get svc --all-namespaces
```

You can also use the `-o json` and redirect to file to just a JSON dump here too if you like. 

## kubectl options

The above examples of kubectl demonstrate the simplest way to use the tool, using the default options or those configured in `~/.kube/config`. If you want to try multiple authentication sources to see if there are differences in access, it's helpful to know a few options that help you modify the way that kubectl operates.

The `--all-namespaces` (or `-A` for short) option might not work for you if you are not authorised to query details cluster wide (as in the service account example shown in the previous section). If that's the case, you can try to query only specific namespaces using the `-n` option (the command `kubectl get ns` will list configured namespaces assuming you have permissions to do so).

See the example below, querying the `default` namespace. (The `default` namespace is the default namespace used by kubectl if you dont explicitly configure one as we have done below.)

```
bash-5.1# kubectl get pods -n default -o wide
NAME                              READY   STATUS    RESTARTS      AGE   IP             NODE                         NOMINATED NODE   READINESS GATES
hacker-container                  2/2     Running   3 (48d ago)   49d   172.16.7.159   kubecontainer.thezoo.local   <none>           <none>
haproxy-77766c8866-2hf44          2/2     Running   0             46d   172.16.7.182   kubecontainer.thezoo.local   <none>           <none>
ichnaea-5566dd5bd7-hxccb          2/2     Running   2 (48d ago)   49d   172.16.7.145   kubecontainer.thezoo.local   <none>           <none>
mitmproxy-7658d6f68d-d67bh        2/2     Running   0             46d   172.16.7.184   kubecontainer.thezoo.local   <none>           <none>
theia-d88b7b7b4-kd99k             2/2     Running   0             29d   172.16.7.130   kubecontainer.thezoo.local   <none>           <none>
```

Kubernetes supports a number of ways in which you can authenticate to the API server. You can [check here](https://thegreycorner.com/2023/11/15/kubernetes-auth-deep-dive.html) for a deep dive into this subject.

One of the more commonly supported authentication methods is via JWT tokens. In the command above, kubectl is using it's default setting of authenticating using the pods service account token `/run/secrets/kubernetes.io/serviceaccount/token`. 

If we want to specify a particular token to use, we can do this using the `--token` option, as shown below. 

```
bash-5.1# kubectl --token "$(cat /run/secrets/kubernetes.io/serviceaccount/token)" get pods -o wide
NAME                              READY   STATUS    RESTARTS      AGE   IP             NODE                         NOMINATED NODE   READINESS GATES
hacker-container                  2/2     Running   3 (48d ago)   48d   172.16.7.159   kubecontainer.thezoo.local   <none>           <none>
haproxy-77766c8866-2hf44          2/2     Running   0             46d   172.16.7.182   kubecontainer.thezoo.local   <none>           <none>
ichnaea-5566dd5bd7-hxccb          2/2     Running   2 (48d ago)   48d   172.16.7.145   kubecontainer.thezoo.local   <none>           <none>
mitmproxy-7658d6f68d-d67bh        2/2     Running   0             46d   172.16.7.184   kubecontainer.thezoo.local   <none>           <none>
theia-d88b7b7b4-kd99k             2/2     Running   0             28d   172.16.7.130   kubecontainer.thezoo.local   <none>           <none>
```


## Accessing the API server using curl

If you dont dont want to use kubectl to query the API server, you can also do it using a generic HTTP client like curl. The following example shows how to perform such a query. 

We add the authorisation token (using the `Authorization` header), specify the Certificate Authority CA file to verify the SSL connection (`--cacert`) and perform a namespaced query for pods in the `default` namespace (using path `/api/v1/namespaces/default/pods`).

```
bash-5.1# curl -H "Authorization: Bearer $(cat /run/secrets/kubernetes.io/serviceaccount/token)" --cacert /run/secrets/kubernetes.io/serviceaccount/ca.crt https://kubernetes.default.svc.cluster.local/api/v1/namespaces/default/pods
[SNIP]
```

The output will be a JSON dump of all the data, similar to what would be returned by the `-o json` in kubectl. Ive removed this in the above output in the interests of space. It's very detailed, so you will probably want to redirect it to disk for parsing. If you wanted namespaced service information instead you would replace `pods` in the above with `services`.

The following URI paths can be used to perform cluster-wide queries for pods and services respectively, assuming you have the required permissions:

* /api/v1/pods
* /api/v1/services

I would suggest testing both non-namespaced and namespaced queries for both pods and services using each API authentication method you have available to you before giving up on the API, because it's the most authorative and easiest way to get the information you want. For namedspaced queries, start with the namespace your pod is in and if that works try each of the others. If you can't get the namespace list from the API (at path `/api/v1/namespaces`) read the section later on Kubernetes DNS for some ideas on how to enumerate namespaces.


## Parsing the Kubernetes API JSON output for useful info

Once I have API output for pods and services in JSON format I like to parse out the useful information using iPython. See [here](https://thegreycorner.com/2023/08/16/iPython-for-cyber-security.html) for a good overview on how I use iPython for security related processing. Below are some quick snippets on how to quickly extract the useful information from JSON output for pods or services from the API (either from kubectl or via a HTTP client like curl).

Import the JSON module
```
import json
```


Show the IP and ports for the first container in each pod.
```
pods = json.load(open('/tmp/pods.json'))
{a['metadata']['name'] : [[b['containerPort'] for b in a['spec']['containers'][0]['ports']], a['status']['podIP']] for a in pods['items'] if 'ports' in a['spec']['containers'][0]}
```

Show the IP and ports for each service.
```
services = json.load(open('/tmp/services.json'))
{a['metadata']['name']: [a['spec']['clusterIPs'], [b['port'] for b in a['spec']['ports']]] for a in services['items']}
```


# Authentication to the Kubernetes API using instance credentials in an AWS EKS Kubernetes cluster

If the cluster you are attacking is an AWS EKS Kubernetes cluster, there is another option potentially available to you to authenticate to the Kubernetes API. It involves using the AWS credentials from the internal AWS metadata service to authenticate to the AWS API and obtain the related Kubernetes token. This method does require that you know the EKS clusters name, which you might have to guess.

The first thing you can try is to get the cluster name by listing known EKS clusters like so.

```
aws eks list-clusters
```

If this gives you the cluster name, great! If not, you might be able to guess it, inferring from other values you find elsewhere in the cluster. The caller identity (`aws sts get-caller-identity`) of the instance's credential might contain the cluster name within its role name. You might also be able to come back to this step armed with new information after exploring other services in the cluster found using some of the other methods in this post. Metrics and monitoring services and key value stores like redis can be good sources of information for this.


You can test to see if you have correctly guessed the EKS cluster name by using it in a command like the following.

```
aws eks describe-cluster --name [CLUSTER_NAME]
```

Once you have the correct cluster name, you can get a token you can use for API server authentication like so.

```
aws eks get-token --cluster-name [CLUSTER_NAME]
```

Alternatively, if you want to use the kubectl tool, you can write a config file (`~/.kube/config`) to automatically configure this to use a token as generated by the previous command like so. Once this is done, you can just run kubectl as normal.

```
aws eks update-kubeconfig --name [CLUSTER_NAME]
```

Once you have a EKS token, try it against the API as discussed in the previous section - it may have more access than the pod's service account token.


# Traditional host and port discovery

If you can't use the Kubernetes API, which will give you a complete list of pods, service addresses and ports, the next approach is to use more traditional discovery techniques, which we can target specifically for the peculiarities of Kubernetes.

Start with the IP addresses obtained as discussed in the Situational Awareness section above. We will use these known addresses to infer what the network ranges for pods and services might be.

The "services" range will be the one in which your Kubernetes API server (`10.96.0.1` in my example) and Kubernetes DNS server (`10.96.0.10`) reside. 

Then there is the "pod" range, in which my pod ip (`172.16.7.159`) sits.  

You might have found more IP addresses in your target cluster (in environment variables, local files, etc), take these into account too.


## Possible range of IPs

Given the IP addresses collected above in my example, the possible range of IP addresses that **could** potentially be in use is as follows, based on the IANA privat network range definitions:

```
10.0.0.0 to 10.255.255.255 (16777216 hosts)
172.16.0.0 to 172.31.255.255 (1048576 hosts)
```

This is obviously a large number of addresses to scan, even for a local scan. The approach I usually like to take when discovering services across such a large network range is a two step one of first identifing live hosts and then trying to identify listening services on those hosts.

Let's look at some other potential complicating factors as well as exploring a Kubernetes feature that can help us identify hosts and services.


## Checking environment behaviour

Before we take any further steps to try and discover hosts on nearby addresses, we need to see how these systems respond when we try and probe them.

If we ping the two external known IPs in the service range, we will see they dont respond to ICMP echo requests.

```
bash-5.1# ping 10.96.0.1
PING 10.96.0.1 (10.96.0.1) 56(84) bytes of data.
^C
--- 10.96.0.1 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1025ms

bash-5.1# ping 10.96.0.10
PING 10.96.0.10 (10.96.0.10) 56(84) bytes of data.
^C
--- 10.96.0.10 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2030ms

```

This makes host discovery more challenging - we can't just ping to reliably determine if an IP is assigned in the cluster. Machines not responding to pings is obviously a common issue with host discovery and network scanning, but it's good to confirm that we need to deal with it here too.


## Service mesh complications

Certain service meshes like [Istio](https://istio.io/) work by intercepting traffic to certain pods and services in order to provide more featureful traffic routing. In this case components of the mesh will complete the TCP three way handshake for all valid ports and all valid IP addresses within its configured ranges, before then forwarding the connection in the backend only if there is a configured service mesh route to a pod or service. This can have the effect of TCP ports appearing to be open even when there is nothing actually listening at the associated IP address and/or port. Port scanners that use the TCP handshake to determine if a host is live or a port is open will give you wildly inaccurate results when this occurs. In these cases you are left to rely on application level responses coming back before you can tell if supposedly listening TCP servers actually have something behind them.

The range of IP addresses within a cluster that this behaviour applies to can vary based on configuration, but you can tell its happening for a given IP address if every single applicable TCP port you try on the host shows open. Consequently, if you know the cluster you are looking at uses Istio or another similar service mesh, you need to account for this when doing internal service discovery. If this is not the case you obviously have the option to use more traditional TCP port scanning approaches to identify live hosts and ports.


## Kubernetes DNS to the (partial) rescue

We actually have access to a Kubernetes specific way to identify live services and sometimes pods in the form of the Kubernetes DNS service. 

This service automatically creates various DNS records for both pods and services in the cluster as discussed in the official documentation [here](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/).

There are two types of records for pods
* Forward (e.g. A) records for the pod IP in the format `<pod-ip>.<namespace>.pod.cluster.local` (where dots `.` in the IP address are replaced with dashes `-`), and
* Forward (A) and reverse (PTR) records for any pod exposed by a service at `<pod-ip>.<service-name>.<namespace>.svc.cluster.local`.

And for services there are:
* Forward (A) and reverse (PTR) records for each service in the form `<service-name>.<namespace>.svc.cluster.local`, and
* Service (SRV) records for each listening port of the service in the form  `_<port-name>._<port-protocol><service-name>.<namespace>.svc.cluster.local`.


Unfortunately, the first pod record type is useless for host discovery purposes as there are no reverse entries and forward lookups for any valid IPv4 address in any valid namespace will work, regardless if it is assigned to a pod in the cluster.

For example, using this pod address pattern to check the IP address `127.0.0.1` in the `default` namespace works fine. 

```
bash-5.1# dig +short 127-0-0-1.default.pod.cluster.local
127.0.0.1
```

The only useful enumeration purpose we can achieve with these records is brute forcing valid namespaces - the `default` namespace in the above can be replaced with any other namespace in the system and a result will be returned, as long as the namespace exists in the cluster.

The reverse (PTR) records however are much more useful. 

If we do reverse DNS lookups on our known IPs, we can see that the service IPs both have inverse lookup entries, but the pod IP does not.

```
bash-5.1# dig +short -x 10.96.0.10
kube-dns.kube-system.svc.cluster.local.
bash-5.1# dig +short -x 10.96.0.1
kubernetes.default.svc.cluster.local.
bash-5.1# dig +short -x 172.16.7.159
bash-5.1#
```

This lines up with the documentation - the services both have reverse DNS entries, but the pod (which in this case is not associated with a service) does not. This gives us a way in which we can identify cluster IP addresses associated with services, as well as pod IP addresses that are backed by services, by doing reverse DNS lookups across applicable network ranges. This will identify all IPs associated with services and most of the ones associated with pods, and will help in narrowing the potential IP ranges in use for each. We will look an example of this in the next section.

The other record type mentioned above was SRV records. Taking the most obvious example of the default kubernetes API service, we can get the associated SRV record for the https port like so:

```
bash-5.1# dig +short SRV _https._tcp.kubernetes.default.svc.cluster.local
0 100 443 kubernetes.default.svc.cluster.local.
bash-5.1# dig +short _https._tcp.kubernetes.default.svc.cluster.local
10.96.0.1
```

This tells us the TCP port that the service listens on - 443, as well as  a priority (0) and weight (100) for this particular endpoint. We can also see above that if we request the A record for that name we get the expected service IP address that should exist for all DNS SRV records.

This does give us a way to identify all listening ports associated with a service once we know its DNS name, however the port names come from potentially arbitrary values provided in the service definition and not all of them are as obvious as `https`.

To demonstrate this, I will use the Kubernetes API from my Kubernetes host system to perform a describe operation on the `jaeger-collector` service that comes with Istio (and runs in the `istio-system` namespace), so we can see what some of the service port names are:

```
stephen@kubecontainer:~$ kubectl describe service jaeger-collector -n istio-system
Name:              jaeger-collector
Namespace:         istio-system
Labels:            app=jaeger
Annotations:       <none>
Selector:          app=jaeger
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.97.213.168
IPs:               10.97.213.168
Port:              jaeger-collector-http  14268/TCP
TargetPort:        14268/TCP
Endpoints:         172.16.7.142:14268
Port:              jaeger-collector-grpc  14250/TCP
TargetPort:        14250/TCP
Endpoints:         172.16.7.142:14250
Port:              http-zipkin  9411/TCP
TargetPort:        9411/TCP
Endpoints:         172.16.7.142:9411
Port:              grpc-otel  4317/TCP
TargetPort:        4317/TCP
Endpoints:         172.16.7.142:4317
Port:              http-otel  4318/TCP
TargetPort:        4318/TCP
Endpoints:         172.16.7.142:4318
Session Affinity:  None
Events:            <none>
```


If we wanted to get the SRV record for the `grpc-otel` service port listening on port 4317 mentioned in the output above, we would need to do a request like the following:

```
bash-5.1# dig +short SRV _grpc-otel._tcp.jaeger-collector.istio-system.svc.cluster.local
0 100 4317 jaeger-collector.istio-system.svc.cluster.local.
```

So, while we can easily brute force service PTR/A records by reverse resolving across a range of IPs, to get service records we would need to start with the service A record names and then try various values for the first part of the SRV record. Based on the example above we know that the possible values for this are not included within common service name information sources like `/etc/services` or even the nmap services file, and you may have to rely on additional information to find correct values. For some standardised Kubernetes applications, the service name and namespace may help you narrow down possible service port names via the use of related documentation or source code to build a list for brute forcing.


## Reverse DNS scanning to identify live systems

Lets look at a practical example of using the Kubernetes reverse DNS entries to identify live IPS. In our example cluster we have known service IPs of `10.96.0.10` and `10.96.0.1` which sit inside the IANA assigned private range `10.0.0.0/8`. This is a huge number of addresses to scan, so as an initial compromise we can reduce this to a more managable initial range of `10.96.0.0/16` to demonstrate the concept. A nice way to perform a reverse DNS scan of the hosts in this range is to run nmap as follows and write the output to disk in greppable form.

```
bash-5.1# nmap -oG dns_scan_svc_1 -sn -Pn -R 10.96.0.0/16
[SNIP]
```

Once the scan is done, look at the entries in the file to identify the pattern you need to filter for. This is a local cluster, so in this case the only DNS entries that exist are those created by Kubernetes so we are looking to ignore entries without a reverse name. In a cloud environment like AWS however unallocated IPs in the cluster will likely have AWS created reverse entries probably ending with something like `compute.internal`, so you will want to ignore those instead.

```
bash-5.1# head dns_scan_svc_1
# Nmap 7.91 scan initiated Wed Dec  6 07:58:28 2023 as: nmap -oG dns_scan_svc_1 -sn -Pn -R 10.96.0.0/16
Host: 10.96.0.0 ()	Status: Up
Host: 10.96.0.1 (kubernetes.default.svc.cluster.local)	Status: Up
Host: 10.96.0.2 ()	Status: Up
Host: 10.96.0.3 ()	Status: Up
Host: 10.96.0.4 ()	Status: Up
Host: 10.96.0.5 ()	Status: Up
Host: 10.96.0.6 ()	Status: Up
Host: 10.96.0.7 ()	Status: Up
Host: 10.96.0.8 ()	Status: Up
```

Based on this, we want to look for entries containing the word `Host`, but excluding those that contain no reverse name (so excluding lines containing `()`). We can do this like follows:

```
bash-5.1# grep 'Host' dns_scan_svc_1 | grep -v '()'
Host: 10.96.0.1 (kubernetes.default.svc.cluster.local)	Status: Up
Host: 10.96.0.10 (kube-dns.kube-system.svc.cluster.local)	Status: Up
Host: 10.96.5.10 (my-nginx-svc1.argotest.svc.cluster.local)	Status: Up
Host: 10.96.114.235 (argocd-notifications-controller-metrics.argocd.svc.cluster.local)	Status: Up
Host: 10.96.148.230 (argocd-redis.argocd.svc.cluster.local)	Status: Up
```

We can repeat the above for some "nearby" ranges for our services (e.g. replacing the X in `10.X.0.0/16` with numbers from 90 to 120) to get a more complete list of live service hosts. 

The pods network is a bit more reasonably sized, we can cover the entire thing with CIDR `172.16.0.0/12`. Lets repeat the nmap process from above with that range.

```
bash-5.1# nmap -oG dns_scan_svc_2 -sn -Pn -R 172.16.0.0/12
[SNIP]

bash-5.1# grep 'Host' dns_scan_svc_2 | grep -v '()'
Host: 172.16.7.136 (172-16-7-136.theia.default.svc.cluster.local)	Status: Up
Host: 172.16.7.139 (172-16-7-139.istio-egressgateway.istio-system.svc.cluster.local)	Status: Up
Host: 172.16.7.140 (172-16-7-140.kiali.istio-system.svc.cluster.local)	Status: Up
Host: 172.16.7.142 (172-16-7-142.jaeger-collector.istio-system.svc.cluster.local)	Status: Up
Host: 172.16.7.143 (172-16-7-143.istiod.istio-system.svc.cluster.local)	Status: Up
Host: 172.16.7.145 (172-16-7-145.ichnaea.default.svc.cluster.local)	Status: Up
Host: 172.16.7.146 (172-16-7-146.kube-dns.kube-system.svc.cluster.local)	Status: Up
Host: 172.16.7.151 (172-16-7-151.argocd-metrics.argocd.svc.cluster.local)	Status: Up
Host: 172.16.7.152 (172-16-7-152.argocd-server.argocd.svc.cluster.local)	Status: Up
Host: 172.16.7.153 (172-16-7-153.argocd-notifications-controller-metrics.argocd.svc.cluster.local)	Status: Up
Host: 172.16.7.154 (172-16-7-154.argocd-applicationset-controller.argocd.svc.cluster.local)	Status: Up
Host: 172.16.7.155 (172-16-7-155.istio-ingressgateway.istio-system.svc.cluster.local)	Status: Up
Host: 172.16.7.156 (172-16-7-156.argocd-dex-server.argocd.svc.cluster.local)	Status: Up
Host: 172.16.7.157 (172-16-7-157.grafana.istio-system.svc.cluster.local)	Status: Up
Host: 172.16.7.159 (hacker-container)	Status: Up
Host: 172.16.7.161 (172-16-7-161.kube-dns.kube-system.svc.cluster.local)	Status: Up
Host: 172.16.7.163 (172-16-7-163.argocd-repo-server.argocd.svc.cluster.local)	Status: Up
Host: 172.16.7.165 (172-16-7-165.argocd-redis.argocd.svc.cluster.local)	Status: Up
Host: 172.16.7.166 (172-16-7-166.prometheus.istio-system.svc.cluster.local)	Status: Up
Host: 172.16.7.177 (172-16-7-177.my-nginx-svc1.argotest.svc.cluster.local)	Status: Up
Host: 172.16.7.182 (172-16-7-182.haproxy.default.svc.cluster.local)	Status: Up
Host: 172.16.7.184 (172-16-7-184.mitmproxy.default.svc.cluster.local)	Status: Up
```

Based on this, we can see that live pods associated with services seem to be all congregated within the subnet `172.16.7.0/24` - if we wanted to specifically scan to identify other pods not backed by services, this range would be a good place to concentrate with other scanning techniques. If you have a service mesh preventing TCP connect scanning as a way of identifying other hosts, see the next section for a last ditch option.


## Port scanning of live hosts

Once you have a list of live IPs the next step is identifying open ports. In the absense of a service mesh complicating things by showing all ports open as discussed above, the quickest way to achieve this is simple port scanning with nmap or similar. Otherwise, I can think of two other options.

For IPs in the service range, each valid port will have an associated SRV DNS record as discussed in the section on Kubernetes DNS above. This means that you can start with the service A record name, for example `service.namespace.svc.cluster.local` and try and brute force TCP SRV records using a pattern like `_[value]._tcp.service.namespace.svc.cluster.local`. 

As a last ditch option, I modified an existing Python port scanner I found to identify open ports by trying to trigger an application level response after a successful TCP connection. This is conceptually similar to the way UDP scanning works in nmap, and does largely work, but the implementation is still rudimentary and is not 100% reliable. If you want, you can grab this [here](https://github.com/stephenbradshaw/pentesting_stuff/blob/master/utilities/appportscan.py).

