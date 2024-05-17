---
layout: post
current: post
cover: assets/images/cilium.jpg
navigation: True
title: "Multi cluster networking with cilium cluster mesh"
date: 2023-09-30 00:00:00
tags: tech
class: post-template
subclass: "post"
logo: assets/images/ghost.png
author: lahiru
---

As organizations increasingly adopt distributed architectures and scale their Kubernetes deployments, the need for robust networking and security solutions that can seamlessly operate across multiple clusters becomes paramount.In this blog, we will go through how you can use Cilium Multi-Cluster Mesh to effectively manage a fleet of Kubernetes clusters spanning across availability zones or regions, thereby achieving unparalleled levels of high availability and fault tolerance across your infrastructure.

## 🐝 Cilium Multi-cluster in a nutshell 🐝

Cilium Multi-Cluster extends Cilium's capabilities to manage and secure Kubernetes clusters across multiple kubernetes clusters across availability zones or regions. This is done by deploying an additional api server called `clustermesh-apiserver` to synchronize the shared state among the kubernetes clusters. Each kubernetes cluster holds its state in the etcd and the kubernetes clusters in the cluster mesh can access other cluster's state via the `clustermesh-apiserver`. For this each cluster should expose its `clustermesh-apiserver` as a Load balancer service. Cilium agents running in the kubernetes clusters connect to the `clustermesh-apiserver` of other clusters, watch for changes and replicate the multi-cluster relevant state into their own cluster.

Cilium cluster mesh provides the following features

- Pod IP routing across multiple Kubernetes clusters at native performance via tunneling or direct-routing without requiring any gateways or proxies.
- Transparent service discovery with standard Kubernetes services and coredns/kube-dns.
- Network policy enforcement spanning multiple clusters. Policies can be specified as Kubernetes NetworkPolicy resource or the extended CiliumNetworkPolicy CRD.
- Transparent encryption for all communication between nodes in the local cluster as well as across cluster boundaries.

## Time to get our hands dirty...

In this blog, we are going to deploy the following setup in your laptop using `kind` and demonstrate the multi cluster load balancing and fault tolerance capabilities supported by cilium cluster mesh. The setup consists of 2 kubernetes clusters which are named as US and EU for ease of relating to real world scenario of multi region clusters. We are going to install cilium in both of the clusters and then enable cilium cluster mesh on both. Then we are going to deploy a hello world application in both of the clusters along with a busybox pod to access the application within the cluster. Then we are going to make the hello world service accessible across clusters enabling load balancing across clusters. Finally we are going to use service affinity rules in cilium to demonstrate the fault tolerant capabilities across clusters.

<p align="center">
  <img alt="Kubernetes Custom Controllers" src="assets/images/multi-cluster.png">
    <em>Multi cluster networking</em>
</p>

### 1. Create the two kubernetes clusters using kind.

- Create a new directory to host the kubernetes manifests. Let's name it as `cilium-cluster-mesh`.
- Open a terminal session (Terminal 1) and set the environment variable `KUBECONFIG` to `export KUBECONFIG=./kubeconfig-us.yaml`
- Create a file with the following content and save it as `kind-us-cluster.yaml`

```yaml
---
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
networking:
  disableDefaultCNI: true
  podSubnet: 10.1.0.0/16
  serviceSubnet: 172.20.1.0/24
nodes:
  - role: control-plane
    extraPortMappings:
      # localhost.run proxy
      - containerPort: 32042
        hostPort: 32042
      # Hubble relay
      - containerPort: 31234
        hostPort: 31234
      # Hubble UI
      - containerPort: 31235
        hostPort: 31235
  - role: worker
  - role: worker
```

- Execute `kind create cluster --name us --config kind-us-cluster.yaml` to create the US cluster. If the cluster creation is successful, following logs can be seen.

```sh
Creating cluster "us" ...
 ✓ Ensuring node image (kindest/node:v1.27.3) 🖼
 ✓ Preparing nodes 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-us"
You can now use your cluster with:

kubectl cluster-info --context kind-us

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/

```

- Next open a new terminal session (Terminal 2) and set the environment variable `KUBECONFIG` to `export KUBECONFIG=./kubeconfig-eu.yaml`
- Create a file with the following content and save it as `kind-eu-cluster.yaml`

```yaml
---
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
networking:
  disableDefaultCNI: true
  podSubnet: 10.2.0.0/16
  serviceSubnet: 172.20.2.0/24
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

- Execute `kind create cluster --name eu --config kind-eu-cluster.yaml` to create the EU cluster. If the cluster creation is successful, following logs can be seen.

```sh
samples/cilium-cluster-mesh on  main [?] …
➜ kind create cluster --name eu --config kind-eu-cluster.yaml
Creating cluster "eu" ...
 ✓ Ensuring node image (kindest/node:v1.27.3) 🖼
 ✓ Preparing nodes 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-eu"
You can now use your cluster with:

kubectl cluster-info --context kind-eu

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/

```

### 2. Install cilium cni in the two clusters.

- In the terminal 1, execute the following command to install the cilium cni in the US cluster.

```sh
cilium install \
  --set cluster.name=us \
  --set cluster.id=1 \
  --set ipam.mode=kubernetes
```

Following logs can be seen after executing the above command.

```sh
🔮 Auto-detected Kubernetes kind: kind
✨ Running "kind" validation checks
✅ Detected kind version "0.20.0"
ℹ️  Using Cilium version 1.15.4
ℹ️  Using cluster name "eu"
🔮 Auto-detected kube-proxy has been installed
```

- Execute the command `cilium status` to check the status of the cilium installation.

<p align="center">
  <img alt="Kubernetes Custom Controllers" src="assets/images/cilium-status.png">
    <em>Multi cluster networking</em>
</p>

- In the terminal 2, execute the following command to install the cilium cni in the EU cluster.

```sh
cilium install \
  --set cluster.name=eu \
  --set cluster.id=2 \
  --set ipam.mode=kubernetes
```

Following logs can be seen after executing the above command.

```sh
🔮 Auto-detected Kubernetes kind: kind
✨ Running "kind" validation checks
✅ Detected kind version "0.20.0"
ℹ️  Using Cilium version 1.15.4
ℹ️  Using cluster name "eu"
🔮 Auto-detected kube-proxy has been installed
```

Now if you followed the instructions upto this point correctly, you would have 2 kubernetes clusters up and running in your laptop with cilium installed as the cni.

### 3. Enable cilium clustermesh in both clusters

Next we are going to enable cilium clustermesh in both US and EU clusters. This will deploy an additional deployment for the `clustermesh-apiserver` in both of the clusters. Cilium agents running in the EU cluster will use this `clustermesh-apiserver` in the US cluster to replicate the states of US cluster to its own EU cluster and vice versa. Therefore this `clustermesh-apiserver` should be exposed as a Load balancer service for accessing outside the cluster. But for demonstrating purposes, we are going to expose the `clustermesh-apiserver` as a NodePort service as we don't have dynamic load balancers available. This is not recommended for production use cases as the service becomes unavailable when the node goes down.

To enable `clustermesh-apiserver` run the following command in both Terminal 1 (US) and Terminal 2 (EU).

```sh
cilium clustermesh enable --service-type NodePort
```

Finally to connect the clusters, execute the following command.

```sh
cilium clustermesh connect --context us --destination-context eu
```

To check the status of the cilium clustermesh execute the command `cilium clustermesh status --wait`

### 4. Deploy the sample hello world application

Deploy the sample hello world application by executing the following commands.

```sh
kubectl apply -f deployment-us.yaml
kubectl apply -f deployment-eu.yaml
```

Check whether sample hello world is working as expected by executing the following command in both US and EU clusters.

```sh
kubectl exec -it busybox-deployment-b7bc87c95-8q7l9 -- /bin/sh -c 'for i in $(seq 1 10); do wget -qO- nginx:80; echo ""; done'
```

If everything is working as expected, you will see a output similar to the following logs.

```
{Hello world}
```

Did you notice that all the responses in the US are returned from the nginx in US ? And all the responses in the EU are returned from the nginx in EU ? Eventhough we have enabled cilium clustermesh in both clusters and connected the two clusters, we haven't specified the hello world service as a global service. So the requests are load balanced among the pods within the same cluster.

### 5. Specify hello world service as a global service to enable cross cluster service discovery and load balancing.

To enable hello world service as a global service, execute the following command in both US and EU clusters.

```sh
kubectl annotate service nginx service.cilium.io/global="true"
```

Now invoke the hello world service again and notice the responses received.

```sh
kubectl exec -it busybox-deployment-b7bc87c95-8q7l9 -- /bin/sh -c 'for i in $(seq 1 10); do wget -qO- nginx:80; echo ""; done'
```

Now you can see that the requests from the US cluster are load balanced to US and EU and vice versa. But in most cases, we want to load balance cross cluster for fault tolerance. Load balancing cross cluster is not ideal always as the latency may get high

But in an ideal scenario, we want to load balance globally for fault tolerance only if the local services are not available. To achive such behaviour, we can use the service affinity rules in cilium.

### 6. Specify hello world service affinity as local for fault tolerant cross cluster load balancing.

Execute the following command to specify the hello world service affinity as `local`. This will load balance requests to the hello world service among the pods in the cluster and route requests to the other cluster only if the local service is not available.

```sh
kubectl annotate service nginx service.cilium.io/affinity="local"
```

To demonstrate the fault tolerant behaviour, scale down the hello world service in US cluster and invoke the hello world service from the US cluster.

```sh
kubectl scale deployment rebel-base --replicas 0
```

```sh
kubectl exec -it busybox-deployment-b7bc87c95-8q7l9 -- /bin/sh -c 'for i in $(seq 1 10); do wget -qO- nginx:80; echo ""; done'
```

Eventhough the hello world service is down in the US, the requests get routed to the hello world service in EU.
