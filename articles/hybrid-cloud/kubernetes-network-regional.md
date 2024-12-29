# Kubernetes on Hybrid Cloud: Network dising

In hybrid cloud environments, network design is one of the most important and fundamental parts. It is a basic requirement for any type of Kubernetes cluster, whether it is a single-zone cluster or a multi-regional cluster. In this article, we will explain how to design a network for a hybrid cloud Kubernetes cluster.

Key Considerations for Network Design:
* Ensure reliable and secure connections between on-premises and cloud data centers.
* Optimize network latency
* Plan IP address ranges to avoid conflicts between data centers
* Use firewalls to control traffic between networks

## Communication inside the cluster

Node-to-node:

All nodes in the cluster must have connectivity to each other. Control plane nodes must have access to the kubelet on each node. Nodes must have access to the control plane nodes.

Pod-to-pod:

Pods must have access to the control plane nodes to reach the Kubernetes API. Monitoring systems must have access to all nodes to collect metrics and logs. Pods must be able to communicate with other pods within the same cluster. In many cases, pods also need access to the internet and cloud provider services. The latency between nodes, zones, and regions is a crucial factor. Application deployments should be aware of the network topology to optimize performance.  Furthermore, when regions are located in different cloud providers, communication can become unstable and slower. Proper planning and optimization are required to address these challenges.

In summary, all pods and nodes must have connectivity with each other. A hybrid Kubernetes cluster has multiple zones and regions, adding more complexity to network design.

## Network design

### Hardware or Saas VPN

The well-known cloud providers offer VPN services. VPN links can be established between zones and regions, enabling secure and stable connections across different geographical locations. VPN connections can be hardware-based or software-based, depending on the infrastructure requirements. VPN helps in maintaining data security and encryption during transmission.

![Kubernetes-ipsec!](/articles/hybrid-cloud/img/kubernetes-ipsec.png)

The red line represents the VPN connection between cloud providers. The VPN connection is encrypted and secure. However, this link uses public internet connections, which can sometimes be less reliable and have unstable bandwidth.

### Direct Connect

Direct Connect is a dedicated network connection service provided by major cloud providers (e.g., AWS Direct Connect, Azure ExpressRoute, Google Cloud Interconnect). It allows organizations to establish a private, high-bandwidth connection between their the cloud provider's infrastructure.

![Kubernetes-link!](/articles/hybrid-cloud/img/kubernetes-link.png)

The latency between cloud providers is very low, and bandwidth is predictable. This link uses dedicated fiber optic cables, which are more reliable than the public internet.

### Mesh network

A mesh network is a network topology where each node establishes a connection with every other node. Unlike traditional hierarchical networks, mesh networks are decentralized, allowing every node to cooperate in data distribution and routing. All connections are encrypted and secure.

![Kubernetes-mesh!](/articles/hybrid-cloud/img/kubernetes-mesh.png)

### Public IPv6 for Pods and Services

A Kubernetes cluster supports IPv6 for both Pods and Services, allowing seamless communication across regions without requiring any special configuration for cross-region traffic. This native IPv6 support simplifies networking and ensures scalability for modern cloud-native workloads.

However, enabling IPv6 alone is not enough to guarantee a secure communication environment. Ensure that your applications uses mutual TLS (mTLS) or other security mechanisms to protect data in transit.

Note that not all cloud providers fully support IPv6 for Kubernetes Pods and Services. Check with your cloud provider for the latest information on IPv6 support.
