# Talos and network management

The network management is an important part of a Kubernetes cluster, especially in hybrid and multi-cloud environments. The stability and predictability of the network are very important for the applications running on the cluster. The network is usually more stable in one physical location than in a cross-cloud environment.

The basic components can impact the stability of the application:
* DNS resolving
* Network stability
* Network latency
* Network bandwidth

## DNS resolving

The application needs to resolve DNS names to IP addresses. By default, a Kubernetes cluster uses CoreDNS as its DNS server. CoreDNS is deployed as a Kubernetes deployment and can be scaled up or down. However, if the CoreDNS pods are very far from the application pod, latency may increase, and DNS names might fail to resolve.

To solve this issue, use a DaemonSet to deploy CoreDNS on each node. Additionally, set the TrafficPolicy for the CoreDNS service to Local [Service traffic topology and routing](https://dev.to/sergelogvinov/kubernetes-on-hybrid-cloud-service-traffic-topology-and-routing-3gle). The DNS traffic will stay within the node, keeping the latency very low.

## Network stability

For kubelet and kube-proxy, network stability is crucial. These components communicate with the Kubernetes API server to configure the network and run the pods. The kubelet also updates the status of the pods and node. If the status is not updated regularly, the Kubernetes API can mark the node as unhealthy, and the pods may be rescheduled to another node.

Imagine a situation where the pods and network are working fine, but the kubelet loses connection to the API server (for example, if the Kubernetes API load balancer goes down). Kubernetes will create new copies of the pods on another node, and the old pods will be terminated once the kubelet reconnects to the API server. For stateless applications, this behavior is usually not a problem. However, for stateful applications, like databases, it can cause significant issues.

Talos solves this problem by using an embedded load balancer on each node. The kubelet and kube-proxy (or CNI plugins) connect to the local load balancer, which forwards traffic to the API server. This ensures consistent connectivity and helps avoid unnecessary disruptions caused by API server load balancer failures.

You can switch it on by setting in machine configuration:

```yaml
machine:
  features:
    kubePrism:
      enabled: true
      port: 7445
```

After this config, the Kubernetes API server becomes accessible on port 7445 on each node using the local host address.

## Network latency and bandwidth

The best way to reduce network latency is to use native network routing. However, in hybrid and multi-cloud environments, this is not possible. The CNI (Container Network Interface) provides network overlays to address this issue, using technologies like VXLAN, GRE, or WireGuard. In all these cases, the network overlay adds an additional header to the packets, increasing network latency and reducing network bandwidth.

Talos includes an embedded network mesh based on WireGuard, a fast and secure VPN protocol that encrypts traffic between nodes. Regardless of where the nodes are located or whether they are behind NAT, the nodes can communicate with each other seamlessly.

However, since this mesh is an additional component in the network stack, it can introduce latency and some instability. The recovery process in case of issues can be slow and may take a long time.

The network mesh can be enabled in the machine configuration:

```yaml
machine:
  network:
    kubespan:
      enabled: true
cluster:
  discovery:
    enabled: true
```

To reduce the recovery time, you can set filters to limit the IP addresses that can be used to create the tunnels. By specifying these filters, you can ensure that the network mesh uses only specific IP in ranges:

```yaml
machine:
  network:
    kubespan:
      filters:
        endpoints:
          - 0.0.0.0/0
          - '::/0'
          - '!192.168.0.0/16'
          - '!172.16.0.0/12'
          - '!10.0.0.0/8'
          - '!fd00::/8'
```

If you want to establish a mesh network only between datacenters while using the native network for communication between nodes within each datacenter, consider using [kilo](https://kilo.squat.ai/)

Kilo can deploy as CNI plugin that creates a WireGuard-based mesh network across Kubernetes zones, region and datacenters. It allows efficient and secure connectivity between nodes in different datacenters while maintaining native networking within each datacenter. This hybrid approach can optimize performance by reducing latency and overhead for intra-datacenter traffic while ensuring secure and reliable communication between datacenters.
