# Talos on GCP with Spot Instances

Using Spot Instances on Google Cloud Platform (GCP) is an excellent way to reduce infrastructure costs. This guide explains how to run Talos on GCP with Spot Instances effectively.

One unique behavior of GCP Spot Instances is that they lose their IP address when they are preempted (stopped by Google). Most Container Network Interfaces (CNI) can handle this and update the node's IP address. However, if you're using IPv6 for your Pods, the CNI cannot update the CIDR (IP range) resources assigned to the node. This means that after preemption, you cannot run Pods on that node with proper IPv6 addresses.

This guide explains how to work around this issue and run Talos on GCP with Spot Instances effectively.

## Configurations

Talos Cloud Controller Manager (CCM) is designed to work across different environments, including Google Cloud Platform (GCP). It can handle IP address changes when a node is preempted. To enable this functionality, you need to activate the `cloud-node-lifecycle` controller in your Talos configuration.

```yaml
# Helm values for Talos CCM
enabledControllers:
  - cloud-node
  - cloud-node-lifecycle
```

How It Works:
* Talos CCM watches for node eviction events.
* When a node is preempted, Talos CCM removes the node's resources from the cluster.
* When the node starts again, Talos CCM adds the updated node resources back to the cluster.
* For the CNI plugin, it will appear as if the node was replaced with a new one.

Additionally, Talos CCM marks Spot Instances with the label:

```yaml
node.cloudprovider.kubernetes.io/lifecycle: spot
```

You can use this label to schedule your Pods based on your specific requirements.

## Conclusion

Running Talos on any cloud provider with Talos Cloud Controller Manager (CCM) is an excellent way to manage your Kubernetes cluster efficiently. Talos CCM can handle IP address changes when a node is preempted and helps you effectively manage Spot Instances for better resource optimization and cost savings.

## References

* [Talos](https://talos.dev)
* [Talos CCM](https://github.com/siderolabs/talos-cloud-controller-manager)
* [Public IPv6 for Pods](https://dev.to/sergelogvinov/kubernetes-pods-with-global-ipv6-1aaj)
