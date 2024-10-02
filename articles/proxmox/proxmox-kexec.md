# Install Proxmox on any bare metal server

## Introduction

Installing Proxmox on a bare metal server is easy when you have direct access.
But if you rent a server from a hosting provider, you might face some challenges:

* __No IPMI/iDRAC access__: You don’t have remote tools to manage the server.
* __No DHCP/PXE network__: The network doesn’t have DHCP or PXE, or you can't control it, so you can't boot from the network.
* __No templates available__: The hosting provider doesn’t have a ready template for your server, even though you know Proxmox works on it.
* __Can't change disk partitions__: The hosting provider’s templates don’t let you change disk partition sizes or filesystems.
* __Bonding interfaces required__: The hosting provider uses bonding interfaces, which the standard net installer doesn’t support.
* __Minimal install desired__: You want to install Proxmox with just the basics, without any extra software from the hosting provider.

These can make the installation more difficult.

I'd like to share my experience with the project [ansible-role-debian-boot](https://github.com/sergelogvinov/ansible-role-debian-boot) which solves most of the challenges.

The __main idea__ is to use the __existing__ operation system on the server to install Debian with you own partitioning table. Once Debian is installed, we will install Proxmox on top of it. Since Proxmox is based on Debian, this method works well.

We will use the Debian [netinstall](https://www.debian.org/CD/netinst/) installer, which includes the kernel and initrd, along with a custom preseed file to automate the Debian installation. This approach allows us to configure the system with the correct partitioning and network settings without manual intervention.

## Requirements

You need to have `ansible` installed on your local machine.

Download [ansible-role-debian-boot](https://github.com/sergelogvinov/ansible-role-debian-boot) role

```bash
ansible-galaxy role install sergelogvinov.debian-boot
```

or clone the project (if you want to modify the files)

```bash
git clone https://github.com/sergelogvinov/ansible-role-debian-boot.git
```

## Usage

All options you can find here [main.yml](https://github.com/sergelogvinov/ansible-role-debian-boot/blob/main/defaults/main.yml)

I will show you the most important options:

```yaml
# Reinstall the server with Debian
- hosts: all
  vars:
    # add a new entry to the GRUB menu with the Debian net installer. It will also download the netboot kernel and initrd to the server. (default: false)
    debian_grub: true
    # use kexec to reboot the server directly into the new kernel and initrd, without a full hardware reboot. (default: false)
    debian_kexec: true
    # rebuild the initrd, including the current network configuration and the custom preseed file to ensure proper booting and installation. (optional)
    debian_rebuild_initrd: true

    # Name or URL of the preseed file
    debian_preseed: proxmox.cfg

    # root password that will be used during the installation.
    debian_preseed_password: "password"
    # ssh key that will be added for root user.
    debian_sshkey: "ssh-rsa AAAA"

  roles:
    - ansible-role-debian-boot
```

Predefined __preseed files__ can be found here [proxmox.cfg](https://github.com/sergelogvinov/ansible-role-debian-boot/blob/main/files/bookworm/proxmox.cfg), [proxmox-lvm.cfg](https://github.com/sergelogvinov/ansible-role-debian-boot/blob/main/files/bookworm/proxmox-lvm.cfg).


The Ansible playbook configures the Debian net installer with the preseed file and the necessary boot arguments. Specifically, it will:

- Set up the network configuration, including the interface, IP address, gateway, and DNS (DHCP is not required).
- Use the preseed file to define the partitioning scheme and software to be installed.
- Set the root password.
- Add the SSH key for the root user.

__Important__: Remember to change the root password after the installation is complete for security reasons.

## Options

* [Install through GRUB menu](#install-through-grub-menu)
* [Install through kexec](#install-through-kexec)
* [Install in rescue mode](#install-in-rescue-mode)
* [Install with bond interface](#install-with-bond-interface)

## Install through GRUB menu

`debian_grub: true` - will add the new entry to the grub menu with the Debian net installer.
It helps to boot the server with the Debian net installer, in case if you have access to the server console or through changing the boot order.

Some old servers or arm-based boards do not support the `kexec` command, so you can use the grub menu to boot the server with the Debian net installer.

Check new entry in the grub menu:

```bash
grep -A 4 'Debian Net' /boot/grub/grub.cfg
```

Output:

```bash
menuentry "Debian Net Installer" {
    linux /boot/debian-kernel  keymap=us language=en country=US locale=en_US.UTF-8 priority=critical   url=https://raw.githubusercontent.com/sergelogvinov/ansible-role-debian-boot/main/files/bookworm/proxmox-lvm.cfg
    initrd /boot/debian-initrd.gz
}
### END /etc/grub.d/15_debian_installer ###
```

## Install through kexec

`debian_kexec: true` - will use the `kexec` to boot the server with the new kernel and initrd.
If you do not have access to the server console, you can use the `kexec` to boot the server with the new kernel and initrd.
After successful loading the new kernel and initrd, the server will boot with the `Debian Net installer` with the `preseed` file.

__Cautions__: this option will reboot the operation system without any confirmation and __format the disk__.

### Rebuild initrd

`debian_rebuild_initrd: true` - will download and rebuild the initrd with the network configuration, ssd keys and the preseed file.
All necessary files will be added to the initrd.
And dyring the boot process, the installer will use the network configuration and the preseed file stored in the initrd.

Ansible will add the following files to the initrd:

```bash
find /boot/preseed/ -type f
```

Output:

```bash
# Additional network configuration (for bonding interfaces)
/boot/preseed/lib/debian-installer.d/S25bonding-interfaces
# Sshd configuration
/boot/preseed/lib/debian-installer.d/S60sshd
# Helper script to keep the network up (for bonding interfaces)
/boot/preseed/usr/bin/if-keep-up.sh
# ssh_host keys copied from the host
/boot/preseed/etc/ssh/sshd_config
/boot/preseed/etc/ssh/ssh_host_rsa_key
/boot/preseed/etc/ssh/ssh_host_ed25519_key
/boot/preseed/etc/ssh/ssh_host_ecdsa_key.pub
/boot/preseed/etc/ssh/authorized_keys
/boot/preseed/etc/ssh/ssh_host_ecdsa_key
/boot/preseed/etc/ssh/ssh_host_ed25519_key.pub
/boot/preseed/etc/ssh/ssh_host_rsa_key.pub
# Pinning the network interface
/boot/preseed/etc/udev/rules.d/70-persistent-net.rules
# For manual network trubleshooting (network config from previous OS)
/boot/preseed/network.sh
# Preseed file
/boot/preseed/debian-preseed.cfg
```

### Preseed file

Customize the preseed file, which will be used to install the Debian.
Official [preseed example](https://preseed.debian.net/debian-preseed/bullseye/amd64-main-full.txt).

```yaml
debian_preseed: "https://raw.githubusercontent.com/sergelogvinov/ansible-role-debian-boot/main/files/{{ debian_version }}/proxmox.cfg"
```

#### Preseed file with Soft RAID1

The Proxmox installer does not support `Soft RAID1`, only `zfs` for redundancy to the root partition.
But zfs requires a lot of memory and proper configuration.
So, sometimes it is better to use Soft RAID1.

Base my experience, almost all hosting providers offer servers with 2 disks.
So, Soft RAID1 is a good choice.

Preseed file [proxmox.cfg](https://github.com/sergelogvinov/ansible-role-debian-boot/blob/main/files/bookworm/proxmox.cfg) will install the Debian with `Soft RAID1` the following partitioning:

| Partition | Type            | Mount | Size |
|-----------|-----------------|-------|------|
| 1,2       | BIOS/EFI        |       |      |
| 3         | ext4            | /boot | 1G   |
| 4         | swap            |       | 4G   |
| 5         | Softraid + lvm  |       | all  |

lvm partition: `root` - 20G, `vz` - 4G, volume group `data` - all free space

```bash
/dev/md0               943M  197M  682M  23% /boot
/dev/sdb2              512M  164K  512M   1% /boot/efi
/dev/mapper/data-root   20G  3.0G   17G  16% /
/dev/mapper/data-vz    3.8G   60M  3.7G   2% /var/lib/vz
```

#### Preseed file for one disk

Preseed file [proxmox-lvm.cfg](https://github.com/sergelogvinov/ansible-role-debian-boot/blob/main/files/bookworm/proxmox-lvm.cfg) will install the Debian on one disk with `LVM` the following partitioning:

| Partition | Type            | Mount | Size |
|-----------|-----------------|-------|------|
| 1,2       | BIOS/EFI        |       |      |
| 3         | ext4            | /boot | 1G   |
| 4         | swap            |       | 4G   |
| 5         | lvm             |       | all  |

lvm partition: `root` - 20G, `vz` - 4G, volume group `data` - all free space

```bash
/dev/sdb3              943M  197M  682M  23% /boot
/dev/sdb2              512M  164K  512M   1% /boot/efi
/dev/mapper/data-root   20G  3.0G   17G  16% /
/dev/mapper/data-vz    3.8G   60M  3.7G   2% /var/lib/vz
```

You can clone the project and modify the preseed file for your needs.
Preseed variable `debian_preseed` can be a URL to the preseed file.
Or use `debian_rebuild_initrd` flag to add everything to the initrd.

## Install in rescue mode

If you cloud provider has a `rescue mode`, you can use it to boot the server and run the ansible playbook to install Debian.
As all operation system runs in memory, you can use only `kexec` installation method.

```yaml
# Reinstall the server in rescue mode
- hosts: all
  vars:
    debian_grub: false
    debian_kexec: true
    debian_rebuild_initrd: true

    debian_preseed_password: "password"
    debian_sshkey: "ssh-rsa AAAA"

    debian_preseed: proxmox.cfg
  roles:
    - ansible-role-debian-boot
```

## Install with bond interface

The Debian net installer does not support bonding interfaces (port channels) by default.
However, some hosting providers require the use of bonding interfaces for network connectivity.

If your server does not initialize its network interfaces in `bonding mode`, the installation `will not have access` to the network, potentially causing the installation process to fail or become incomplete.

So, ansibe role define the bonding interface and take care of this issue.
You need to set `debian_rebuild_initrd: true` and `debian_interface: bond0` in the playbook.
Ansible will add some scripts to the initrd to initialize the bonding interface for you.

To resolve this issue, an ansible role can be defined to configure the bonding interface and handle network initialization during the Debian installation.
In your playbook, you need to set the following variables:

```yaml
debian_rebuild_initrd: true
debian_interface: bond0
```

Ansible will then add the necessary scripts to the initrd, ensuring that the bonding interface (bond0) is properly initialized.
This will allow the network interfaces to be set up correctly during the installation process, enabling network access for the installer.

## Post-installation tasks

After the installation, pressid run [following tasks](https://raw.githubusercontent.com/sergelogvinov/ansible-role-debian-boot/main/files/bookworm/proxmox-playbook.yaml):

* allow root login via ssh and set the password `debian_preseed_password`
* add ssh key for the root user `debian_sshkey`
* run ansible playbook to fix default settings for the Proxmox
    * ansible-role-system - basic system settings
    * ansible-role-iptables - add default iptables rules

```bash
sed -i 's/PermitRootLogin .*/PermitRootLogin Yes/g' /target/etc/ssh/sshd_config; echo 'PermitRootLogin Yes' >>/target/etc/ssh/sshd_config; \
mkdir /target/root/.ssh; \
wget -O /target/root/.ssh/authorized_keys {{ url if debian_sshkey is url else 'https://github.com/sergelogvinov.keys' }}; \
chmod 0600 /target/root/.ssh/authorized_keys; \
wget -O /target/root/proxmox-playbook.yaml https://raw.githubusercontent.com/sergelogvinov/ansible-role-debian-boot/main/files/bookworm/proxmox-playbook.yaml; \
in-target ansible-galaxy role install git+https://github.com/sergelogvinov/ansible-role-system.git,main; \
in-target ansible-galaxy role install git+https://github.com/sergelogvinov/ansible-role-iptables.git,main; \
in-target mkdir /dev/shm; \
in-target ansible-playbook --connection=local /root/proxmox-playbook.yaml; \
rm -rf /target/.ansible /target/root/proxmox-playbook.yaml;
```

Feel free to modify the pressed command `preseed/late_command` for your needs or remove it at all.

Pressed templates:
* lvm on top of soft raid1 [proxmox.cfg.j2](https://github.com/sergelogvinov/ansible-role-debian-boot/blob/main/templates/preseed/proxmox.cfg.j2)
* lvm on top of one disk [proxmox-lvm.cfg.j2](https://github.com/sergelogvinov/ansible-role-debian-boot/blob/main/templates/preseed/proxmox-lvm.cfg.j2)

## Install Proxmox

Now that Debian is installed, the next step is to install Proxmox on top of it.

You can use the following [official documentation](https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_12_Bookworm),
or use the ansible role [ansible-role-proxmox](https://github.com/sergelogvinov/ansible-role-proxmox).

Download the role:

```bash
ansible-galaxy role install sergelogvinov.proxmox
```

Add the role to the playbook:

```yaml
# Install Proxmox
- hosts: all
  roles:
    - ansible-role-proxmox
```

It adds the Proxmox repository and installs the Proxmox packages.
After the installation, reboot the server.

## Conclusion

We've had an old operating system on the server, and we've installed Debian with a custom partitioning scheme and network configuration, without manual intervention.
And than we've installed Proxmox on top of it.

The project [ansible-role-debian-boot](https://github.com/sergelogvinov/ansible-role-debian-boot) under __MIT license__, so feel free to fork and modify this project. Just remember to leave a star on the project :)

I hope this article helps you successfully install Debian and Proxmox on any hosting provider.
