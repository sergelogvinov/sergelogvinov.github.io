# Install Talos on any cloud servers

## Introduction

[Talos](https://talos.dev) is a modern OS for Kubernetes.
It is designed to be secure, immutable, and minimal. Talos is built on top of the Linux kernel and includes everything required to run Kubernetes.
It is designed to be installed on bare-metal servers, virtual machines, and cloud instances.
Unfortunately, many cloud providers do not have Talos as an option in their marketplace.
In this guide, I will show you how to install Talos on any cloud server.

Let's assume your cloud does not provide:

* __No IPMI/iDRAC access__: You don’t have remote tools to manage the server, and you can't mount an ISO image.
* __No DHCP/PXE network__: The network doesn’t have DHCP or PXE, or you can't control it, so you can't boot Talos from the network.
* __No Talos image available__: The hosting provider doesn’t have a Talos image available in their marketplace.
* __No CDROM support__: You can't mount/attache an ISO image to the virtual server.

But you already have a cloud server with a Linux OS installed, and you have SSH access to it.

I'd like to share my experience with the project [ansible-role-talos-boot](https://github.com/sergelogvinov/ansible-role-talos-boot) which helps you to install Talos on any cloud server.

The __main idea__ is to use the __existing__ operation system on the server to install Talos.
Most of the OSs allow you to boot a kernel and initrd image though the `kexec` tool.
Or use grub to boot a Talos from boot menu.

## Requirements

You need to have `ansible` installed on your local machine.

Download [ansible-role-talos-boot](https://github.com/sergelogvinov/ansible-role-talos-boot) role

```bash
ansible-galaxy role install sergelogvinov.talos-boot
```

## Usage

All options you can find here [main.yml](https://github.com/sergelogvinov/ansible-role-talos-boot/blob/main/defaults/main.yml)

I will show you the most important options:

```yaml
- hosts: all
  vars:
    # Add boot entry to the grub menu
    talos_grub: true
    # Use kexec to boot Talos kernel
    talos_kexec: true
    # Provide predefined machineconfig file
    talos_machineconfig: "worker.yaml"
  roles:
    - sergelogvinov.talos-boot
```

The Ansible playbook will gather all the necessary network information from the existing operating system and use it to create a Talos configuration patch file.
After that, it will download the Talos kernel and initrd images and add a boot entry to the GRUB menu for booting.
Or use the `kexec` tool to boot the Talos.

## Options

* [Install through GRUB menu](#install-through-grub-menu)
* [Install through kexec](#install-through-kexec)
* [Predefined machineconfig](#predefined-machineconfig)

### Install through GRUB menu

If your server uses a GRUB bootloader, you can add a boot entry to the GRUB menu.
The role will download the Talos kernel and initrd images and automatically add a boot entry to the GRUB menu for you.

```yaml
- hosts: all
  vars:
    talos_grub: true
  roles:
    - ansible-role-talos-boot
```

Check new entry in the grub menu:

```bash
grep -A 4 'Talos' /boot/grub/grub.cfg
```

Output:

```bash
menuentry "Talos" {
    linux /boot/talos-kernel init_on_alloc=1 slab_nomerge pti=on console=tty1 console=ttyS0 consoleblank=0 nvme_core.io_timeout=4294967295 printk.devkmsg=on ima_template=ima-ng ima_appraise=fix ima_hash=sha512 talos.platform=metal net.ifnames=0 ip=x.x.x.x::x.x.x.x:255.255.255.0::eth0:off talos.dashboard.disabled=1
    initrd /boot/talos-initrd.xz
}
### END /etc/grub.d/10_talos ###
```

You can now review the kernel parameters before rebooting the server.
Most of these parameters are customizable, and you have the option to modify them directly in the playbook.

### Install through kexec

If your server is running a Linux OS that supports the `kexec` tool, you can use it to boot the Talos kernel and initrd image.
The role will download the Talos kernel and initrd images and use the `kexec` tool to boot into Talos.
This approach works on most virtual server environments.

```yaml
- hosts: all
  vars:
    talos_kexec: true
  roles:
    - sergelogvinov.talos-boot
```

### Predefined machineconfig

If you already have a predefined machineconfig file, you can provide it to the role.
The role will create a Talos configuration patch file and apply it once Talos boots up.

```yaml
- hosts: all
  vars:
    # Create talos configuration patch file if you provided already predefined machineconfig file
    talos_machineconfig: "worker.yaml"
    talos_machineconfig_node_labels: "topology.kubernetes.io/zone=MyZone,topology.kubernetes.io/region=MaRegion"
  roles:
    - sergelogvinov.talos-boot
```

Inventory file:

```ini
[all]
worker-node    ansible_host=x.x.x.x ansible_ssh_user=root
```

The result of patch:

```yaml
machine:
  kubelet:
    extraArgs:
      node-labels: "topology.kubernetes.io/zone=MyZone,topology.kubernetes.io/region=MaRegion"
    nodeIP:
      validSubnets:
        - "x.x.x.x/32"
        - "x:x:x::1/128"
  network:
    hostname: "worker-node"
    interfaces:
      - interface: eth0
        dhcp: false
        addresses:
          - "x.x.x.x/32"
          - "x:x:x::1/64"
        routes:
          - network: "0.0.0.0/0"
            gateway: "x.x.x.x"
          - network: "::/0"
            gateway: "x:x:x::x"
    kubespan:
      enabled: true
  install:
    image: ghcr.io/siderolabs/installer:v1.7.7
    bootloader: true
    wipe: true
    disk: /dev/vda
```

The role will define:
* hostname (from the inventory file)
* network configuration (single stack or dual stack)
* kubelet labels
* Talos installer image
* disk to install Talos

## Conclusion

We have successfully installed Talos on a cloud server without needing IPMI/iDRAC access, DHCP/PXE network, or CD-ROM support.
There's no need to contact your cloud provider to add Talos to their marketplace.
Instead, you can use the existing operating system to install Talos directly.

The project [ansible-role-talos-boot](https://github.com/sergelogvinov/ansible-role-talos-boot) under __MIT license__, so feel free to fork and modify this project. Just remember to leave a star on the project :)
