# Kubernetes PODs with global IPv6

## Introduction

IPv6 is the future of the internet because it offers a much bigger address space, better security, and improved network performance. Kubernetes, a popular tool to manage containers, also supports IPv6.

When using IPv4, pods need a NAT gateway to connect to the internet, which makes the network more complex and slower. With IPv6, you can give pods direct global addresses, so they can access the internet without extra steps.

In a single cloud provider, enabling IPv6 in Kubernetes is easy. But it gets more difficult when you try to use multiple cloud providers or bare-metal servers because each one has a different IPv6 prefix and subnet. Kubernetes, by default, only allows one IPv6 prefix per cluster. To use multiple prefixes for different pods, you will need a custom solution.

This article will show how to set up Kubernetes with global IPv6 addresses across multiple cloud providers with different subnets.

## Prerequisites

We will use Talos as our Kubernetes distribution. Talos is a modern operating system built specifically for Kubernetes. It is designed to be secure, unchangeable (immutable), and user-friendly. If you are not familiar with Talos, you can visit the [official website](https://talos.dev/) to learn more about it.

## Prepare the environment

I hope you familiar with Talos, and you know how to create a Talos cluster.
I will show you what you need to change in default Talos configuration to enable global IPv6 addresses for pods.

Copy and paste the following configuration to the files: `controller.yaml` and `all.yaml`

```yaml
# controller.yaml
machine:
  features:
    kubernetesTalosAPIAccess:
      enabled: true
      allowedRoles:
        - os:reader
      allowedKubernetesNamespaces:
        - kube-system
cluster:
  network:
    cni:
      name: none
  controllerManager:
    extraArgs:
      node-cidr-mask-size-ipv4: "24"
      node-cidr-mask-size-ipv6: "112"
      controllers: "*,tokencleaner,-node-ipam-controller"
```

```yaml
# all.yaml
machine:
  kubelet:
    extraArgs:
      cloud-provider: external
cluster:
  network:
    podSubnets: ["10.0.0.0/12","fd40:10::/96"]
    serviceSubnets: ["10.100.0.0/22","fd40:10:100::/112"]
```

Generate the Talos configuration with the following command:

```shell
talosctl gen config --config-patch @all.yaml --config-patch-control-plane @controller.yaml --with-kubespan --with-docs=false --with-examples=false cluster-v6 https://localhost:6443
```

Those changes will:

* Enable IPv6 by defining the podSubnets and serviceSubnets with IPv6 addresses.
* Disable the default node IPAM controller by setting controllers option: `"*,tokencleaner,-node-ipam-controller"`.
* Apply the required settings for Talos CCM, see talos features and kubelet arguments.

## Deploy the cluster

How to create a VM with the Talos image is beyond the scope of this article. Please refer to the [official documentation](https://www.talos.dev/) for guidance. After bootstrapping the control plane, the next step is to deploy the `Talos CCM` along with a `CNI plugin`.

### Talos CCM

First, we need to deploy the Talos CCM.

```yaml
# Helm values talos-ccm.yaml
enabledControllers:
  - cloud-node
  - node-ipam-controller

extraArgs:
  - --allocate-node-cidrs
  - --cidr-allocator-type=CloudAllocator
  - --node-cidr-mask-size-ipv4=24
  - --node-cidr-mask-size-ipv6=80

daemonSet:
  enabled: true

tolerations:
  - effect: NoSchedule
    operator: Exists
```

Install Talos CCM with the following command:

```shell
helm upgrade -i --namespace=kube-system -f talos-ccm.yaml talos-cloud-controller-manager \
        oci://ghcr.io/siderolabs/charts/talos-cloud-controller-manager
```

Talos CCM will collect the node network information provided by the cloud provider and allocate a CIDR block from the node’s IPv6 subnet.
This means that your pods and nodes will reside in the same IPv6 subnet.

Check the pod CIDR with the following command:

```shell
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDRs}'; echo
```

### CNI plugin

Second, we need to deploy the CNI plugin.
We will be using Cilium as the CNI plugin.
The Helm values for deploying Cilium are as follows:

```yaml
# Helm values cilium.yaml
k8sServiceHost: "localhost"
k8sServicePort: "7445"

cni:
  install: true

ipam:
  mode: "kubernetes"
k8s:
  requireIPv4PodCIDR: true
  requireIPv6PodCIDR: true

enableIPv6Masquerade: false
enableIPv4Masquerade: true

cgroup:
  autoMount:
    enabled: false
  hostRoot: /sys/fs/cgroup

securityContext:
  privileged: true
```

Install cilium with the following command:

```shell
helm upgrade -i --namespace=kube-system --version=1.16.2 -f cilium.yaml cilium cilium/cilium
```

Now you can add more nodes from any cloud provider or bare-metal server to the cluster.
The Talos Cloud Controller Manager (CCM) will automatically gather the network information for each new node, and the CNI plugin (Cilium) will handle network routing, ensuring that both pods and nodes remain in the same IPv6 subnet. This setup allows seamless scaling across multiple environments.

## Test the deployment

Create a pod with the following command:

```shell
kubectl run -it --rm --restart=Never --image=ghcr.io/sergelogvinov/curl:latest curl -- sh
```

Check the pod IP address with the following command:

```shell
ip a
curl -6 icanhazip.com
```

You should now see the global IPv6 address assigned to each pod.
With this setup, you can ping the pod's IPv6 address from another pod within the cluster or even directly from the internet, as the global IPv6 address allows for direct communication without the need for NAT.

```shell
ping6 <pod-ipv6-address>
```

Check the node and pod CIDRs:

```shell
kubectl get nodes -o jsonpath='{range .items[*].status.addresses[?(@.type=="InternalIP")]}{.address}{"\n"}{end}' | grep ':'; echo
kubectl get nodes -o jsonpath='{range .items[*].spec}{.podCIDRs}{"\n"}{end}'; echo
```

Output with 5 nodes should be like this:

```shell
2001:a:b:2875::7952
2a00:a:0:1c4::64
2a09:a:f:42f6::1
2a02:a:152:202::100
2a02:a:153:201::100

["10.0.0.0/24","2001:a:b:2875::/80"]
["10.0.11.0/24","2a00:a:0:1c4::/80"]
["10.0.4.0/24","2a09:a:f:42f6::/80"]
["10.0.13.0/24","2a02:a:152:202::100/121"]
["10.0.12.0/24","2a02:a:153:201::100/121"]
```

The node IPv6 addresses are from different subnets, but the pod CIDRs are allocated from the same subnet as the node. This ensures that each node has its own unique IPv6 subnet, while the pods on each node share the same subnet as the node itself.

## Conclusion

We have successfully deployed Kubernetes pods with global IPv6 addresses. You can now ping the pods directly from the internet, as they have global IPv6 addresses.
The pods also have direct access to the internet without needing a NAT gateway, simplifying the network and improving performance.

Please remember to firewall both the pods and nodes to protect them from unauthorized access over the internet.
Ensuring proper firewall rules are in place is crucial for maintaining security in your Kubernetes cluster, especially when using global IPv6 addresses that allow direct internet access.

If you have any questions, please feel free to ask on GitHub
* [Talos](https://github.com/siderolabs/talos)
* [Talos CCM](https://github.com/siderolabs/talos-cloud-controller-manager)
* [Kubernetes hybrid cluster examples](https://github.com/sergelogvinov/terraform-talos)
