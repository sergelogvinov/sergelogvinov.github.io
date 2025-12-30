# Do you need a free-tier to learn Kubernetes?

This is my opinion, based on my experience and on job interviews with candidates.

It is very important to understand the core components of Kubernetes — what happens when you create a new cluster and what is happening behind the scenes.
This knowledge will help you troubleshoot issues in the future. In such cases, I do not believe ChatGPT can fully help if you do not already understand the fundamentals.

Pods, Deployments, Services, and other basic components are relatively easy to learn through video courses or ChatGPT explanations.

Since most Kubernetes clusters run in the cloud, it’s important to understand how Kubernetes communicates with a cloud provider.
Bootstrap your own simple cloud. Proxmox is a good alternative and a vendor lock-in-free solution.

One Proxmox node is enough to get started, 4 CPUs and 16 GB of RAM are sufficient to run two virtual machines.
* one Control-plane node
* one Worker node

You will delete and recreate the worker nodes many times. This process will help you understand how the cluster works.

Use well-known kubernetes distributions like Talos. It is easy to install, and GitHub provides many examples on how to set it up.
Talos has a dedicated distribution for Proxmox, so you can use it without issues by following the official documentation.

Most kubernetes clusters include components:
* CCM - cloud controller manager
* CNI - container network interface
* CSI - container storage interface
* Node automation - tools like Cluster Autoscaler or Karpenter

In well-known cloud providers, some components are configured for you by default, such as the CCM and CNI.
All other components usually need to be installed manually.
In a home lab, you need to install all components yourself.
This helps you better understand how Kubernetes works in a cloud environment.

In a Proxmox installation, all required components already exist on the internet, mainly on GitHub.
Today, self-hosted solutions based on Proxmox can give you almost the same experience as public clouds, while helping you build strong knowledge of core Kubernetes components.
After that, switching to a public cloud will be much easier for you.

1. Install Proxmox CCM, Proxmox CSI, and Karpenter. The CNI is already included in the Talos distribution.
Do everything manually. Write down all the steps and save them in your GitHub repository.

2. Play with your cluster, break things and fix them again.
Deploy simple applications, scale them up and down, and monitor resource usage.

3. Try to automate the installation using Terraform.
Save the code in your GitHub repository and create step-by-step instructions explaining how to deploy it.

4. Then use GitOps best practices to manage your cluster with Argo CD or Flux CD.

Just remember, all these steps already exist on the internet. You can search for them on Google or ask ChatGPT.

This experience will give you strong advantages in future job interviews, even if the company uses a different cloud provider.
Do not forget to mention that you have your own home lab and share links to your GitHub repositories.
The certificates show only that you know the right answers, but your home lab shows that you can more than that.

Good luck!
