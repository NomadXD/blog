---
layout: post
current: post
cover: assets/images/envoy-gateway-cilium-cover.jpg
navigation: True
title: "Bulletproof Kubernetes Networking with Cilium and Envoy Gateway"
date: 2025-04-01 00:00:00
tags: tech
class: post-template
subclass: "post"
logo: assets/images/ghost.png
author: lahiru
published: false
---

The widespread adoption of Kubernetes has revolutionized application deployment and management, enabling unprecedented scalability and flexibility. This often leads to scenarios where multiple teams or applications share the same Kubernetes cluster, a concept known as multi-tenancy. While offering numerous benefits, multi-tenancy introduces significant challenges, like network isolation, storage isolation, node isolation, API server priority and fairness, and mitigating "noisy neighbor" issues. Among these, network isolation and controlled communication between different tenants or applications often emerge as the most crucial concerns for organizations, primarily because an unsegmented network can lead to unauthorized access to sensitive data and allow lateral movement for attackers, thereby compromising the confidentiality and integrity of tenant workloads.

By default, Kubernetes allows open communication between pods, regardless of their namespace. This can lead to potential security vulnerabilities in environments where multiple tenants or applications share the same cluster. The challenge lies in establishing robust network boundaries and regulating inter-namespace traffic while still facilitating necessary interactions between services. In this blog, we will explore how we can leverage Envoy Gateway along with Cilium to establish robust network boundaries and regulate north south and east west traffic based on application level and network level policies.

## Cilium Network Policies for Network Microsegmentation

Microsegmentation in Kubernetes involves creating isolated network segments within a cluster, enabling granular control over network traffic and enhancing security. Let us explore how we can implement Cilium network policies that, by default deny all inter-namespace traffic, effectively creating strong isolation boundaries around each tenant. For instance, a policy can be applied to a specific namespace to ensure that no pods within it can initiate connections to pods in other namespaces, and no external pods (from other namespaces) can connect to its pods.

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  # Apply this policy in the specific tenant's namespace
  namespace: tenant-a-ns
  name: allow-tenant-comms-and-dns
  description: "Allows all pod-to-pod communication within this namespace (tenant-a-ns), allows egress to CoreDNS for DNS resolution, and restricts other inter-namespace traffic by default."
spec:
  # endpointSelector: {}
  # An empty endpointSelector selects all pods within the namespace
  # where this policy is applied (i.e., all pods in 'tenant-a-ns').
  endpointSelector: {}

  # Ingress rules: Define what incoming traffic is allowed to the selected pods.
  ingress:
    - fromEndpoints:
        # An empty selector within fromEndpoints here means "allow traffic from
        # any other pod within this same namespace (tenant-a-ns)".
        - {}

  # Egress rules: Define what outgoing traffic is allowed from the selected pods.
  # Egress traffic is allowed if it matches ANY of the rules in this list.
  egress:
    # Rule 1: Allow all egress to other pods within the same namespace.
    - toEndpoints:
        # An empty selector within toEndpoints here means "allow traffic to
        # any other pod within this same namespace (tenant-a-ns)".
        - {}

    # Rule 2: Allow egress to CoreDNS pods in the 'kube-system' namespace for DNS.
    - toEndpoints:
        - matchLabels:
            # This selects pods that have the Kubernetes label 'k8s-app' with the value 'kube-dns'.
            # Cilium uses the 'k8s:' prefix here to refer to standard Kubernetes labels.
            "k8s:k8s-app": "kube-dns"
            # This special label 'io.kubernetes.pod.namespace' is used by Cilium
            # to specify the namespace of the pods to select.
            "k8s:io.kubernetes.pod.namespace": "kube-system"
      toPorts:
        - ports:
            - port: "53"
              protocol: "UDP"
            - port: "53"
              protocol: "TCP"
          # Optional: If you want to implement L7 DNS policy (e.g., restrict to specific domain patterns)
          # you could add a 'rules' section here. For general DNS resolution, L4 is sufficient.
          # rules:
          #   dns:
          #     - matchPattern: "*" # Allows resolution of any domain

  # --- How this restricts other inter-namespace traffic ---
  #
  # If a pod is selected by any CiliumNetworkPolicy, Cilium enforces
  # a 'default deny' posture for traffic not explicitly allowed.
  #
  # Since this policy explicitly allows:
  #   a) all traffic within 'tenant-a-ns'
  #   b) DNS traffic to 'kube-system'
  #
  # All other inter-namespace traffic (both ingress and egress) will be denied,
  # unless other specific policies are created to permit such communication.
```

This policy ensures two key things for pods within a designated tenant namespace (e.g., tenant-a-ns): first, it permits unrestricted communication between any pods residing within that same namespace, facilitating the necessary interactions for the tenant's applications. Second, it explicitly allows these pods to send DNS queries to CoreDNS pods (typically located in the kube-system namespace) on UDP/TCP port 53, which is essential for service discovery and external hostname resolution.

Beyond this baseline, Cilium offers much more granular control. For instance, if you require more fine-grained control over DNS traffic on a per-tenant basis, you can leverage Cilium's L7 DNS-aware policies. This would allow you to specify exactly which domain names or patterns each tenant's pods are permitted to resolve, adding an extra layer of security and preventing unauthorized DNS lookups.

Furthermore, when tenants need to access services or endpoints outside the Kubernetes cluster, such as external databases, APIs, or other cloud services, you can extend this policy. Cilium allows you to define egress rules using toCIDR to permit traffic to specific IP address ranges, or toFQDNs (Fully Qualified Domain Names) to allow traffic to specific external domain names, ensuring that even external communication is explicitly controlled and audited.

Now let us consider a more practical multi-tenant deployment setup where multiple teams deploy their workloads to the same kubernetes cluster. Let us assume there are 3 teams in this organization; sales, marketing and finance. Sales team needs to access salesforce via APIs and marketing team needs to access Google Analytics via APIs and the finance team needs to access their legacy system running in the on premise data center via 203.0.113.10. Following are the relevant Cilium Network Policies for this scenario.

<div class="tabs">
  <div class="tab-buttons">
    <button class="tab-button active" onclick="openTab(event, 'sales-tab')">Sales Team</button>
    <button class="tab-button" onclick="openTab(event, 'marketing-tab')">Marketing Team</button>
    <button class="tab-button" onclick="openTab(event, 'finance-tab')">Finance Team</button>
  </div>

<div id="sales-tab" class="tab-content active">
<pre><code class="language-yaml">
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  namespace: tenant-sales
  name: default
spec:
  endpointSelector: {}
  ingress:
    - fromEndpoints:
        - {}
  egress:
    - toFQDNs:
        - matchName: "api.salesforce.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: "TCP"
    - toEndpoints:
        - matchLabels:
            "k8s:k8s-app": "kube-dns"
            "k8s:io.kubernetes.pod.namespace": "kube-system"
      toPorts:
        - ports:
            - port: "53"
              protocol: "UDP"
            - port: "53"
              protocol: "TCP"
</code></pre>
</div>

<div id="marketing-tab" class="tab-content">
<pre><code class="language-yaml">
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  namespace: tenant-marketing
  name: default
spec:
  endpointSelector: {}
  ingress:
    - fromEndpoints:
        - {}
  egress:
    - toFQDNs:
        - matchName: "analyticsdata.googleapis.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: "TCP"
    - toEndpoints:
        - matchLabels:
            "k8s:k8s-app": "kube-dns"
            "k8s:io.kubernetes.pod.namespace": "kube-system"
      toPorts:
        - ports:
            - port: "53"
              protocol: "UDP"
            - port: "53"
              protocol: "TCP"
</code></pre>
</div>

  <div id="finance-tab" class="tab-content">
<pre><code class="language-yaml">
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  namespace: tenant-finance
  name: default
spec:
  endpointSelector: {}
  ingress:
    - fromEndpoints:
        - {}
  egress:
    - toCIDR:
        - 203.0.113.0/24
      toPorts:
        - ports:
            - port: "443"
              protocol: "TCP"
    - toEndpoints:
        - matchLabels:
            "k8s:k8s-app": "kube-dns"
            "k8s:io.kubernetes.pod.namespace": "kube-system"
      toPorts:
        - ports:
            - port: "53"
              protocol: "UDP"
            - port: "53"
              protocol: "TCP"
</code></pre>
  </div>
</div>

<style>
.tabs {
  margin: 20px 0;
  width: 100%;
  overflow-x: hidden;
}

.tab-buttons {
  display: flex;
  gap: 5px;
  margin-bottom: 10px;
  flex-wrap: wrap;
}

.tab-button {
  padding: 8px 16px;
  border: 1px solid #ddd;
  background: #f8f8f8;
  cursor: pointer;
  border-radius: 4px 4px 0 0;
  flex: 1;
  text-align: center;
  min-width: 120px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.tab-button.active {
  background: #fff;
  border-bottom: 1px solid #fff;
  font-weight: bold;
}

.tab-content {
  display: none;
  padding: 20px;
  border: 1px solid #ddd;
  border-radius: 0 4px 4px 4px;
  width: 100%;
  overflow-x: auto;
}

.tab-content.active {
  display: block;
}

@media screen and (max-width: 600px) {
  .tab-buttons {
    flex-direction: column;
    gap: 8px;
  }
  
  .tab-button {
    width: 100%;
    border-radius: 4px;
    margin: 0;
  }
  
  .tab-button.active {
    border-bottom: 1px solid #ddd;
  }
  
  .tab-content {
    border-radius: 4px;
    padding: 15px;
  }
}
</style>

<script>
function openTab(evt, tabName) {
  var i, tabcontent, tabbuttons;
  
  // Hide all tab content
  tabcontent = document.getElementsByClassName("tab-content");
  for (i = 0; i < tabcontent.length; i++) {
    tabcontent[i].classList.remove("active");
  }
  
  // Remove active class from all buttons
  tabbuttons = document.getElementsByClassName("tab-button");
  for (i = 0; i < tabbuttons.length; i++) {
    tabbuttons[i].classList.remove("active");
  }
  
  // Show the current tab and add active class to the button
  document.getElementById(tabName).classList.add("active");
  evt.currentTarget.classList.add("active");
}
</script>

As we can see from the examples above, for outbound traffic to external services (like Salesforce and Google Analytics), our applications act as API consumers. In these scenarios, network-level restrictions through Cilium policies are sufficient since the authentication and authorization are handled by the external services themselves. This is why we see the sales team's applications authenticating directly with Salesforce, and the marketing team's applications authenticating with Google Analytics.

However, for inbound traffic from external clients and internal service-to-service communication, our applications act as API providers. In these scenarios, network-level restrictions alone are not enough - we need to implement application-level security controls to properly authenticate and authorize clients, validate request contents, and enforce fine-grained access policies. This brings us to our next crucial component: a dual-gateway architecture.

## Gateway Architecture for Secure Traffic Management

<p align="center">
  <img alt="Multi-tenant Kubernetes Architecture" src="assets/images/topology-mermaid.svg">
    <em>Multi-tenant Kubernetes Architecture with Envoy Gateway and Cilium</em>
</p>

## Reference Implementation

Let's walk through a reference implementation of this architecture. Before proceeding with the configuration, ensure you have the following prerequisites installed in your Kubernetes cluster:

1. **Cilium**: Follow the [official Cilium installation guide](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/) to install Cilium as your CNI provider.
2. **Envoy Gateway**: Install Envoy Gateway following the [official installation documentation](https://gateway.envoyproxy.io/v0.6.0/user/quickstart.html).

### Gateway Configuration

First, let's configure our dual gateway setup. We'll create two Gateway resources: an external gateway for handling incoming traffic from outside the cluster, and an internal gateway for service-to-service communication.

```yaml
# external-gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: external-gateway
  namespace: gateway-system
spec:
  gatewayClassName: envoy
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: external-gateway-cert
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchExpressions:
              - key: kubernetes.io/metadata.name
                operator: In
                values: [tenant-sales, tenant-marketing, tenant-finance]
---
apiVersion: v1
kind: Service
metadata:
  name: external-gateway
  namespace: gateway-system
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb # Use appropriate cloud provider annotation
spec:
  type: LoadBalancer
  ports:
    - name: https
      port: 443
      protocol: TCP
  selector:
    gateway.envoyproxy.io/gateway-name: external-gateway
```

```yaml
# internal-gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: internal-gateway
  namespace: gateway-system
spec:
  gatewayClassName: envoy
  listeners:
    - name: http
      port: 8080
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchExpressions:
              - key: kubernetes.io/metadata.name
                operator: In
                values: [tenant-sales, tenant-marketing, tenant-finance]
---
apiVersion: v1
kind: Service
metadata:
  name: internal-gateway
  namespace: gateway-system
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-internal: "true" # For internal load balancer
spec:
  type: LoadBalancer # Or ClusterIP if you don't need a load balancer
  ports:
    - name: http
      port: 8080
      protocol: TCP
  selector:
    gateway.envoyproxy.io/gateway-name: internal-gateway
```

### HTTP Route Configuration

Now, let's configure the HTTPRoute resources for each tenant namespace. These routes define how traffic should be handled for different services within each namespace.

<div class="tabs">
  <div class="tab-buttons">
    <button class="tab-button active" onclick="openTab(event, 'sales-routes')">Sales Routes</button>
    <button class="tab-button" onclick="openTab(event, 'marketing-routes')">Marketing Routes</button>
    <button class="tab-button" onclick="openTab(event, 'finance-routes')">Finance Routes</button>
  </div>

<div id="sales-routes" class="tab-content active">
<pre><code class="language-yaml">
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: sales-external-route
  namespace: tenant-sales
spec:
  parentRefs:
  - name: external-gateway
    namespace: gateway-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api/sales
    backendRefs:
    - name: sales-app-1
      port: 8080
      weight: 1
    - name: sales-app-2
      port: 8080
      weight: 1
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: sales-internal-route
  namespace: tenant-sales
spec:
  parentRefs:
  - name: internal-gateway
    namespace: gateway-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /internal/sales
    backendRefs:
    - name: sales-app-1
      port: 8080
      weight: 1
</code></pre>
</div>

<div id="marketing-routes" class="tab-content">
<pre><code class="language-yaml">
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: marketing-external-route
  namespace: tenant-marketing
spec:
  parentRefs:
  - name: external-gateway
    namespace: gateway-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api/marketing
    backendRefs:
    - name: marketing-app-1
      port: 8080
      weight: 1
    - name: marketing-app-2
      port: 8080
      weight: 1
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: marketing-internal-route
  namespace: tenant-marketing
spec:
  parentRefs:
  - name: internal-gateway
    namespace: gateway-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /internal/marketing
    backendRefs:
    - name: marketing-app-1
      port: 8080
      weight: 1
</code></pre>
</div>

<div id="finance-routes" class="tab-content">
<pre><code class="language-yaml">
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: finance-external-route
  namespace: tenant-finance
spec:
  parentRefs:
  - name: external-gateway
    namespace: gateway-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api/finance
    backendRefs:
    - name: finance-app-1
      port: 8080
      weight: 1
    - name: finance-app-2
      port: 8080
      weight: 1
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: finance-internal-route
  namespace: tenant-finance
spec:
  parentRefs:
  - name: internal-gateway
    namespace: gateway-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /internal/finance
    backendRefs:
    - name: finance-app-1
      port: 8080
      weight: 1
</code></pre>
</div>
</div>

This multi-layered approach combines network-level isolation with application-level security controls, providing comprehensive protection for multi-tenant environments.
