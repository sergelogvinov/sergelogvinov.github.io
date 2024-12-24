# Proxmox as a Cloud Platform for Kubernetes

## Introduction

Proxmox is a powerful open-source server virtualization platform that allows you to run virtual machines (VMs) and containers. It is based on Debian and uses a web-based interface to manage your virtual machines. Proxmox is a great choice for running Kubernetes clusters because it provides a flexible and scalable environment for your applications.

Proxmox has different storage types: local storage, and shared storage. In this article, we will focus on using Proxmox storage as persistent volumes in Kubernetes. We will use the Proxmox Cloud Controller Manager (CCM) and the Proxmox Container Storage Interface (CSI) to manage the storage resources in Kubernetes.

![ProxmoxClusers!](/articles/proxmox/img/proxmox-regions.png)

## Requirements

I assume you have a Proxmox server already installed and running.
You already have a Kubernetes control plane up and running. If not, use the following guide [Install Kubernetes on Proxmox](#step-0-install-kubernetes-optional).

## Step 0: Install Kubernetes (optional)

We will use [Talos](https://github.com/siderolabs/talos) as a Kubernetes distribution. Talos is a modern operating system designed for Kubernetes. It is lightweight, secure, and easy to use. If you are not familiar with Talos, you can visit the [official website](https://talos.dev/) to learn more about it.

## Step 1: Install Proxmox CCM

## Step 2: Install Proxmox CSI

## Use Proxmox volumes in Kubernetes

## Links

- [Talos](https://github.com/siderolabs/talos)
- [Proxmox](https://www.proxmox.com/)
- [Proxmox Cloud Controller Manager](https://github.com/sergelogvinov/proxmox-cloud-controller-manager)
- [Proxmox CSI](https://github.com/sergelogvinov/proxmox-csi-plugin)
- [Kubernetes hybrid cluster examples](https://github.com/sergelogvinov/terraform-talos)
