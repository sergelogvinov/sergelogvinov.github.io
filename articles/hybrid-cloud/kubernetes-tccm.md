# Kubernetes on Hybrid Cloud: Talos Cloud Controller Manager (CCM)

[Talos](https://talos.dev) is a modern operating system designed specifically for Kubernetes. It supports various cloud providers, including AWS, Azure, Google Cloud, OpenStack, and on-premises environments. Talos focuses on security, simplicity, and ease of use. Because Talos nodes are aware of the cloud environment they are running in, the concept of Talos Cloud Controller Manager (CCM) was created

The Talos Cloud Controller Manager (CCM) is built to work effectively in hybrid cloud environments. It collects information from Talos nodes and provides an easy and secure way to manage the lifecycle of Kubernetes nodes in the cluster.

The components of Talos CCM:
* `cloud-node`: is responsible for initializing and managing new nodes when they join a cluster.
* `cloud-node-lifecycle`: Talos CCM does not have any credentials to make a call to the API of the cloud provider. What is why this component is not supported. You need to run native CCM from the cloud provider only with `cloud-node-lifecycle` component, all other components should be disabled.
* `node-route-controller`: Not supported.
* `service-lb-controller`: Not supported.
* `node-ipam-controller`: Manages IP addresses for the Pods in the cluster.
* `node-csr-approval`: Approves the certificate requests from the nodes.

## Controller cloud-node

The cloud-node component is responsible for initializing and managing new nodes when they join a cluster. It performs tasks like: registering and attaching nodes to the Kubernetes cluster, adding important labels, and preparing the node for workloads.

The important labels for hybrid environments are:

* `topology.kubernetes.io/region` specifies the region of the node
* `topology.kubernetes.io/zone` specifies the availability zone
* `node.kubernetes.io/instance-type` specifies the instance type of the node
* `node.cloudprovider.kubernetes.io/platform` specifies the cloud platform where the node is running

These labels are essential for Kubernetes to place workloads (Pods) on the correct nodes based on regional and infrastructure requirements.

In bare-metal environments or environments not recognized by Talos, you can predefine rules for the Cloud Controller Manager. These rules act as a configuration blueprint and are applied to nodes based on their metadata. [Official documentation](https://github.com/siderolabs/talos-cloud-controller-manager/blob/main/docs/config.md).

```yaml
transformations:
  # All rules are applied in order, all matched rules are applied to the node

  - name: nocloud-nodes
    # Match nodes by nodeSelector
    nodeSelector:
      - matchExpressions:
          - key: platform           <- talos platform metadata variable case insensitive
            operator: In            <- In, NotIn, Exists, DoesNotExist, Gt, Lt, Regexp
            values:                 <- array of string values
              - nocloud
    # Set labels for matched nodes
    labels:
      pvc-storage-class/name: "my-storage-class"

  - name: web-nodes                 <- transformation name, optional
    nodeSelector:
      # Or condition for nodeSelector
      - matchExpressions:
          # And condition for matchExpressions
          - key: platform           <- talos platform metadata variable case insensitive
            operator: In            <- In, NotIn, Exists, DoesNotExist, Gt, Lt, Regexp
            values:                 <- array of string values
              - metal
          - key: hostname
            operator: Regexp
            values:
              - ^web-[\w]+$         <- go regexp pattern
    labels:
      # Add label to the node, in this case, we add well-known node role label
      node-role.kubernetes.io/web: ""
```

The first rule applies to nodes on `nocloud` platform environment, such as Proxmox or Oxide. It sets the `pvc-storage-class/name` label to node.
The second rule for `metal` platform with hostnames starting with `web-`. It sets the `node-role.kubernetes.io/web` label, which means the node role.

## Controller cloud-node-lifecycle

The `cloud-node-lifecycle` component plays a crucial role in managing the lifecycle of Kubernetes nodes by interacting with the cloud provider's API. Its main responsibilities include:

* Verifying if a virtual machine (instance) is still running in the cloud provider's infrastructure.
* Ensuring the node is still registered and valid within the cloud environment.
* Monitoring the overall health and status of the node from the cloud provider's perspective.

However, Talos CCM does not have credentials to access cloud provider APIs. To manage the `cloud-node-lifecycle` functionality, you need to run the native CCM provided by your cloud provider. You can run as many CCMs as you have cloud environments in cloud-node-lifecycle mode with simple changes in the code.

## Controller node-route-controller

Not supported.

## Controller service-lb-controller

Not supported.

## Controller node-ipam-controller

This controller makes sense only if you want to have global IPv6 addresses for your Pods. Kubernetes already supports IPv6 for Pods, but only within a single segment. Talos CCM can handle as many segments as you have nodes. The controller allocates an IPv6 segment from the node's range and assigns it to each Pod. See mode detailed example [Kubernetes PODs with global IPv6](https://dev.to/sergelogvinov/kubernetes-pods-with-global-ipv6-1aaj)

## Controller node-csr-approval

The Kubernetes API server must communicate with the Kubelet API to retrieve logs, metrics, and other node-level information. This communication happens over a secure channel, and the Kubelet API is protected by a TLS certificate. By default, when a Kubelet starts, it generates a self-signed certificate to enable TLS communication. However, the Kubernetes API server cannot inherently trust this self-signed certificate because there’s no verification from a Certificate Authority (CA).

To address this issue, Kubernetes uses the `node-csr-approval` controller, which is responsible for:
* When a node starts, the Kubelet generates a Certificate Signing Request (CSR) and submits it to the Kubernetes API server.
* The `node-csr-approval` controller validates the CSR to ensure it meets the security and policy requirements.
* If the CSR is valid, the controller approves it, and the Kubernetes Certificate Authority (CA) signs the certificate.
* The signed certificate is sent back to the Kubelet for use in secure communication.

## Conclusion

Whether you're running Talos in a hybrid cloud environment or in an on-premises setup, Talos CCM is an essential tool for managing your Kubernetes cluster efficiently.

Talos CCM simplifies and streamlines the lifecycle management of nodes, automating key processes such as node initialization, labeling, and certificate management. It ensures that every node in your cluster is properly registered, labeled, and ready to serve workloads in a secure and consistent way.

In `hybrid cloud deployments`, Talos CCM makes it easier to integrate nodes across different environments, ensuring smooth communication and consistent behavior across the cluster, regardless of where your nodes are hosted. By handling these tasks automatically, Talos CCM reduces operational complexity, minimizes human error, and saves time for administrators. In short:

If you're using Talos, using Talos CCM isn't just an option—it's a best practice.

## References

* [Talos](https://talos.dev)
* [Talos Cloud Controller Manager](https://github.com/siderolabs/talos-cloud-controller-manager)
* [Deploy CCMs in one cluster example](https://github.com/sergelogvinov/gitops-examples)
