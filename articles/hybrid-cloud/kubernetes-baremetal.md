# Kubernetes on Hybrid Cloud: Bare-metal or Hypervisor

It is a very popular question: What is the best choice to deploy a Kubernetes node - directly on bare metal or by setting up a hypervisor first and then deploying the Kubernetes node on VMs? There is no single correct answer: it depends on your requirements and the power of your hardware.

Let's compare both solutions.

## Bare-metal installation

If you have a small bare-metal server with 32 cores (or less) and 64-128 GB of RAM, the best choice is to install the Kubernetes node directly on the server. By default, the kubelet has a limitation of 110 pods on the node. If your workload requires a larger number of pods with small resource requirements, you need to think about adjusting the following values:

* Allocate the subnet which has more then 256 IPs (by default subnet is /24 and it has 256 IPs)
* Reserving more resources for kubelet, each pod requires extra resources for kubelet
* Adjusting secrets/config maps sync period
* Check the NUMA and L3 cache architecture on the server
* Run special daemonsets to manage power, monitoring hardware, and other features

## Hypervisor installation

Imagine you have a powerful server with 128 cores and 512 GB of RAM or more. You deploy a Kubernetes node on this server. During maintenance, you lose all the server's capacity. But if you use a hypervisor, you can create virtual machines (VMs) and maintain them one by one, and keep the rest of the VMs running. Also, the Linux kernel and applications are not designed to work efficiently with such a large amount of RAM and CPU cores. The kernel spends time managing TLB, NUMA, and other CPU features. Splitting the server into VMs can help solve many of these problems.

* Each VM will use a separate NUMA node and Memory, L3 cache, see [CPU Affinity](https://dev.to/sergelogvinov/proxmox-cpu-affinity-for-vms-4dhb)
* You can maintain the VMs separately and do not lose the whole server capacity during maintenance, hypervisor kernel usually does nothing and work pritty stable. Restart whole server is not required so often
* You can use the hypervisor features like snapshots, dynamic disk attachment and resizing
* Kubernetes resources isolation, you can deploy different workloads on different VMs
* Simplified node management, add or remove the kubernetes node becomes easier

The very famous open-source hypervisors are:
* [Proxmox VE](https://www.proxmox.com/en/products/proxmox-virtual-environment/overview)
* [Cloud-hypervisor](https://github.com/cloud-hypervisor/cloud-hypervisor)

Using Proxmox as your hypervisor for Kubernetes can provide a cloud-like experience similar to well-known cloud providers, like:
* Dynamic disk attachment, resizing (PV, PVC)
* Network load balancing, firewall, and other features
* Cluster bootstrapping by terraform
