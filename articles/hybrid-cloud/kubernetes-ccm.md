# Kubernetes on Hybrid Cloud: Cloud Controller Manager (CCM)

Almost all Kubernetes services have cloud integration. You might not even know you're using it. For example, when you create a LoadBalancer service, Kubernetes will automatically create a cloud load balancer on the cloud provider's side for you. This is a great feature because you don’t need to worry about the cloud provider's API or how to set up an external load balancer.

In a self-hosted Kubernetes cluster or a hybrid cloud environment, you need to deploy a Cloud Controller Manager (CCM) to enable cloud integration. The CCM is a Kubernetes component that communicates with the cloud provider's API to create and manage cloud resources. It serves as a bridge between Kubernetes and the cloud provider, allowing you to use features such as load balancers, block storage, and network routing.

Here's what each key component of CCM does:
* `cloud-node`: Initializes new nodes during the scale-up process. It ensures nodes are correctly registered in the cluster with the necessary cloud-specific metadata.
* `cloud-node-lifecycle`: Removes nodes from the cluster during the scale-down process. It detects and handles unhealthy or deleted nodes based on information from the cloud provider.
* `node-route-controller`: Creates and manages routes inside the cloud provider's network. It ensures nodes can communicate with each other across the cloud infrastructure.
* `service-lb-controller`: Creates and manages external load balancers for LoadBalancer type services in Kubernetes. It ensures external traffic can reach the appropriate pods through cloud-managed load balancers.

## Controllers cloud-node and cloud-node-lifecycle

The `cloud-node` controller is responsible for initializing new nodes in the cluster. When a new node is added to the cluster, either by an auto-scaler or manually, the cloud-node controller communicates with the cloud provider's API to register and attach the node to the cluster. It also adds labels and taints to the node, such as:

* topology.kubernetes.io/region
* topology.kubernetes.io/zone
* node.kubernetes.io/instance-type

These labels help the Kubernetes scheduler place pods on the correct nodes. For example, you can configure a pod to run only in a specific region, zone, or cloud provider environment.

The `cloud-node-lifecycle` controller is responsible for removing nodes from the cluster. When a node is deleted—either manually, due to failure, or by an auto-scaler—the cloud-node-lifecycle controller ensures that the node is properly removed from the cluster's state. It also handles:

* Detaching cloud resources associated with the node.
* Ensuring all pods running on the removed node are rescheduled to other healthy nodes.

These controllers work together to ensure node lifecycle management aligns with both the Kubernetes cluster state and the cloud provider's infrastructure.

## Controller node-route-controller

The `node-route-controller` is responsible for creating and managing network routes inside the cloud provider's network. When a new node is added to the cluster, the node-route-controller communicates with the cloud provider's API to create a route that directs traffic to the new node.

How It Works:
* The controller ensures traffic can flow between nodes, even if they are spread across different availability zones.
* It is particularly useful when you want to route traffic from an external load balancer directly to the pods on the nodes.

Not all cloud providers support this feature. Support depends on the cloud provider's networking capabilities. It typically works best in one cloud-only environments where routing is fully managed by the cloud provider.

In hybrid cloud environments, the node-route-controller introduces additional complexity because you need to manage pod-subnet routes between: on-premises infrastructure, different cloud environments.

This means configuring consistent routing tables across both environments to ensure traffic flows correctly. Misconfigurations or lack of routing support can cause network connectivity issues between pods across on-premises and cloud nodes.

You can check the pod-subnets and node IPs by commands:

```shell
# Pod CIDR
kubectl get nodes -o jsonpath='{range .items[*].spec}{.podCIDRs}{"\n"}{end}'; echo
# Node IPs
kubectl get nodes -o jsonpath='{range .items[*].status.addresses[?(@.type=="InternalIP")]}{.address}{"\n"}{end}'
```

Output should be like this:

```shell
# Pod CIDR
["10.32.7.0/24","fd00:10:32::7:0/112"]
["10.32.10.0/24","fd00:10:32::a:0/112"]
# Node IPs
172.16.2.100
172.16.2.101
```

In a Kubernetes cluster, node IPs usually come from the same subnet, meaning all nodes are part of one network range. However, the pods on each node get their IP addresses from different subnets (Pod CIDRs). The `node-route-controller` solves this by creating routes in the cloud provider's network. These routes tell the cloud network how to reach each pod subnet through its corresponding node.

From the cloud provider's side it will look like this:

```shell
ip route add 10.32.7.0/24 gw 172.16.2.100
ip route add 10.32.10.0/24 gw 172.16.2.101
```

After the node-route-controller sets up the routes in the cloud provider's network, cloud services like LoadBalancer will know how to route traffic to the pods.

## Controller service-lb-controller

The `service-lb-controller` is responsible for creating and managing load balancers in the cloud provider's network. When you create a LoadBalancer service in Kubernetes, the service-lb-controller will create a cloud load balancer in the cloud provider's API and route traffic to the service's pods through the node ports or directly to the pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  loadBalancerClass: my-cloud-provider
  allocateLoadBalancerNodePorts: true
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  clusterIP: 10.0.171.239
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
      - ip: 192.0.2.127
```

The `allocateLoadBalancerNodePorts` option tells the service-lb-controller to create a node port for the service, in case if the cloud provider does not support direct routing to the pods. The cloud provider will create a load balancer and route traffic to the node port, which will then forward it to the service's pods. Default value is `true`.

The `loadBalancerClass` option specifies the cloud provider's load balancer class to use. This is useful in hybrid cloud environments, otherwise, the CCM with service-lb-controller could create a cloud load balancer in all cloud providers where the cluster is running. The default value is undefined.

The `status.loadBalancer.ingress` field shows the IP address of the cloud load balancer. You can use this IP address to access the service from outside the cluster. This IP from different subnets of you kubernetes cluster. It can be a public IP address or a private IP address, depending on the cloud provider's configuration.

## Hybrid Cloud Considerations

When deploying a Cloud Controller Manager (CCM) in a hybrid cloud environment, there are several key points to keep in mind:
* Most CCMs are designed to work only in their respective cloud environments. They assume full control over nodes, routes, and load balancers within their cloud infrastructure.
* LoadBalancer services are tightly integrated with the cloud provider's network. In a hybrid cloud environment, LoadBalancer services may not work properly, or their setup may require manual configuration and custom routing rules.
* The cloud provider's network may not be able to route traffic directly to pod IPs located in a different cloud or on-premises environment. You may need to manually configure custom routes or use dedicated networking solutions (e.g., VPNs, interconnects, or SD-WAN) to ensure connectivity.
* Running `two CCMs` in a single Kubernetes cluster is tricky and can cause conflicts. Both CCMs may attempt to manage the same nodes, routes, or load balancers. Without clear separation of responsibilities, you might encounter unexpected behavior or resource management issues.

Recommendations:
1. The `node-route-controller` and `service-lb-controller` may not work as expected in a hybrid cloud environment. Just disable them.
2. The `cloud-node-lifecycle` can remove the nodes from another cloud provider's environment, because it doesn't know about this nodes on cloud provider's side.
3. The nodes from one cloud provider may not initialize properly in another provider's environment. However, the CCM specific to each environment will usually handle node initialization correctly. So `cloud-node` keeps working well.
4. `cloud-node-lifecycle` controller may remove nodes from another cloud provider because it doesn’t recognize them. Mismanagement can cause healthy nodes to be removed unintentionally. To prevent this we need to add changes to the code to avoid removing nodes from another cloud provider.

## Conclusion

The Cloud Controller Manager (CCM) is a critical component for managing cloud resources, enabling dynamic scaling, and ensuring nodes are properly prepared in cloud environments. It acts as the bridge between Kubernetes and cloud infrastructure, automating resource management and simplifying cluster operations.

However, in hybrid cloud environments, traditional CCMs face limitations due to provider-specific designs, network routing complexities, and conflicts between multiple CCMs. With a few key adjustments to the CCM code and configuration settings, you can use multiple cloud providers in a single Kubernetes cluster without compromising performance or reliability.

You can find the CCM changes in my repositories. The core improvement involves adding a processing delay during node initialization in the CCM. One CCM gets sufficient time to fully initialize the node before another CCM takes any action. Once a node is properly initialized by its corresponding cloud provider's CCM, other CCMs will skip interfering with the node’s state.
* [cloud-provider](https://github.com/sergelogvinov/cloud-provider/tree/nodelifecycle-1.31) the base package for the cloud provider implementation
* [images with changes](https://github.com/sergelogvinov/containers)
* [fluxcd example to deploy them](https://github.com/sergelogvinov/gitops-examples)
