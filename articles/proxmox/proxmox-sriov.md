# Network performance optimization with nvidia ConnectX on Proxmox

## Introduction

To improve network performance for virtual machines, we can use SR-IOV and tune network settings.
SR-IOV (Single Root I/O Virtualization) is a feature that lets a single PCI Express (PCIe) device directly connect to virtual machines. This improves communication speed and reduces delays by bypassing the hypervisor.

The modern network adapter supports virtual functions (VFs) that can be used by virtual machines.
The network adapter can create multiple virtual network adapters that can be assigned to virtual machines directly by SR-IOV.

This guide will show how to enable SR-IOV and VFs to adjust network performance in Proxmox with Nvidia ConnectX network adapters.
We will use Mellanox ConnectX-6 Lx 25GbE NICs and linux bonding (port agrigation) to get the best reliability and speed.

Using bonding interfaces with SR-IOV can be tricky. However, Mellanox supports handling bonding interfaces at the hardware level.
This means all virtual adapters connect to network hardware switches linked to the bonded interfaces.
To virtual machines, it seems like they are using just one network adapter.

## Requirements

I assume you have a Proxmox server already installed and running.

Check the network adapter. In this example, we use Mellanox Technologies MT2894 (ConnectX-6 Lx) 25GbE NICs.
One network adapter with two ports.

```bash
lspci -nn | grep Ethernet
81:00.0 Ethernet controller [0200]: Mellanox Technologies MT2894 Family [ConnectX-6 Lx] [15b3:101f]
81:00.1 Ethernet controller [0200]: Mellanox Technologies MT2894 Family [ConnectX-6 Lx] [15b3:101f]
```

Find the network interface name.

```bash
dmesg | grep 81:00.0 | grep renamed
mlx5_core 0000:81:00.0 enp129s0f0np0: renamed from eth2
```

We need to find the `switchid` of the network adapter `enp129s0f0np0`, result is `3264160003bd70c4`.
It is used to identify the network adapter in the Open vSwitch configuration.

```bash
ip -d link show enp129s0f0np0
5: enp129s0f0np0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether c4:70:bd:16:64:32 brd ff:ff:ff:ff:ff:ff promiscuity 0  allmulti 0 minmtu 68 maxmtu 9978 addrgenmode eui64 numtxqueues 768 numrxqueues 63 gso_max_size 65536 gso_max_segs 65535 tso_max_size 524280 tso_max_segs 65535 gro_max_size 65536 portname p0 switchid 3264160003bd70c4 parentbus pci parentdev 0000:81:00.0
```

## Enable SR-IOV

Enable SR-IOV in the BIOS.
Enter the BIOS and check the option to enable SR-IOV.
The option may be in different locations depending on the motherboard manufacturer.
Some options related only for AMD processors.

* CPU Virtualization (SVM Mode) -> Enabled
* Chipset -> NBIO Common Options -> IOMMU -> Enabled
* Chipset -> NBIO Common Options -> ACS Enable -> Enabled
* Chipset -> NBIO Common Options -> PCIe ARI Support -> Enabled

Check the SR-IOV status.

```bash
dmesg | grep -i -e DMAR -e IOMMU
```

Check the IOMUU groups.

```bash
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

Or by proxmox command.

```bash
pvesh get /nodes/`hostname -s`/hardware/pci --pci-class-blacklist ""
```

The result should have a lot of IOMMU Groups.

## Configure Open vSwitch

Open vSwitch can offload bonding interfaces to the hardware, allowing the network adapter to manage ogrigeted traffic.
It can also create a virtual switch on the network adapter, connecting virtual functions to the network.

Install Open vSwitch.

```bash
apt install openvswitch-switch ifupdown2 patch
```

Configure the network adapter to the switchdev mode. And set the number of virtual functions to 4.

vi /etc/udev/rules.d/70-persistent-net-vf.rules

```bash
# Ingress bond interface
KERNELS=="0000:81:00.0", DRIVERS=="mlx5_core", SUBSYSTEMS=="pci", ACTION=="add", ATTR{sriov_totalvfs}=="?*", RUN+="/usr/sbin/devlink dev eswitch set pci/0000:81:00.0 mode switchdev", ATTR{sriov_numvfs}="0"
KERNELS=="0000:81:00.1", DRIVERS=="mlx5_core", SUBSYSTEMS=="pci", ACTION=="add", ATTR{sriov_totalvfs}=="?*", RUN+="/usr/sbin/devlink dev eswitch set pci/0000:81:00.1 mode switchdev", ATTR{sriov_numvfs}="0"

# Set the number of virtual functions to 4
SUBSYSTEM=="net", ACTION=="add", ATTR{phys_switch_id}=="52ac120003bd70c4", ATTR{phys_port_name}=="p0", ATTR{device/sriov_totalvfs}=="?*", ATTR{device/sriov_numvfs}=="0", ATTR{device/sriov_numvfs}="4"
# Rename the virtual network adapter to ovs-sw1pf0vf0, ovs-sw1pf0vf1, ovs-sw1pf0vf2, ovs-sw1pf0vf3
SUBSYSTEM=="net", ACTION=="add", ATTR{phys_switch_id}=="52ac120003bd70c4", ATTR{phys_port_name}!="p[0-9]*", ATTR{phys_port_name}!="", NAME="ovs-sw1$attr{phys_port_name}"
```

We need to patch the openvswitch to support the hardware offload.
Add the following lines to the `ovs-ctl` script.
You need to disable auto update of the `openvswitch-switch` package after the changes, otherwise, the changes will be reverted.

```bash
patch  -d/ -p0 --ignore-whitespace <<'EOF'
--- /usr/share/openvswitch/scripts/ovs-ctl.diff	2024-10-16 01:25:28.369482552 +0000
+++ /usr/share/openvswitch/scripts/ovs-ctl	2024-10-16 01:27:32.740490528 +0000
@@ -162,6 +162,8 @@
         # Initialize database settings.
         ovs_vsctl -- init -- set Open_vSwitch . db-version="$schemaver" \
             || return 1
+        ovs_vsctl -- set Open_vSwitch . other_config:hw-offload=true other_config:tc-policy=skip_sw ||:
+        ovs_vsctl -- set Open_vSwitch . other_config:lacp-fallback-ab=true ||:
         set_system_ids || return 1
         if test X"$DELETE_BRIDGES" = Xyes; then
             for bridge in `ovs_vsctl list-br`; do
EOF
```

After the changes, restart the server.
When the server is up, check the openvswitch configuration and offload status.

```bash
ovs-vsctl show
ovs-vsctl get Open_vSwitch . other_config
```

Make sure the `hw-offload=true` is set.

```bash
{hw-offload="true", lacp-fallback-ab="true", tc-policy=skip_sw}
```

And network adapter is in the switchdev mode.

```bash
ip -d link show enp129s0f0np0
```

Output should be like this.

```bash
4: enp129s0f0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq master ovs-system state UP mode DEFAULT group default qlen 1000
    link/ether c4:70:bd:16:64:32 brd ff:ff:ff:ff:ff:ff promiscuity 1  allmulti 0 minmtu 68 maxmtu 9978
    openvswitch_slave addrgenmode none numtxqueues 768 numrxqueues 63 gso_max_size 65536 gso_max_segs 65535 tso_max_size 524280 tso_max_segs 65535 gro_max_size 65536 portname p0 switchid 52ac120003bd70c4 parentbus pci parentdev 0000:81:00.0
    vf 0     link/ether c4:70:ff:ff:ff:e0 brd ff:ff:ff:ff:ff:ff, spoof checking off, link-state auto, trust off, query_rss off
    vf 1     link/ether c4:70:ff:ff:ff:e1 brd ff:ff:ff:ff:ff:ff, spoof checking off, link-state auto, trust off, query_rss off
    vf 2     link/ether c4:70:ff:ff:ff:e2 brd ff:ff:ff:ff:ff:ff, spoof checking off, link-state auto, trust off, query_rss off
    vf 3     link/ether c4:70:ff:ff:ff:e3 brd ff:ff:ff:ff:ff:ff, spoof checking off, link-state auto, trust off, query_rss off
```

vf 0, vf 1, vf 2, vf 3 are the virtual functions.

## Configure the bond interface

Add network bond interface.
Where `enp129s0f0np0` and `enp129s0f1np1` are the physical network adapters.

* vi /etc/network/interfaces

```bash
auto enp129s0f0np0
iface enp129s0f0np0 inet manual

auto enp129s0f1np1
iface enp129s0f1np1 inet manual

auto vmbr1
iface vmbr1 inet static
        ovs_type OVSBridge
        ovs_ports bond1
        ovs_mtu 9000
        address 192.168.1.2/24

auto bond1
iface bond1 inet manual
        ovs_type OVSBond
        ovs_bonds enp129s0f0np0 enp129s0f1np1
        ovs_bridge vmbr1
        ovs_mtu 9000
        ovs_options lacp=active bond_mode=balance-tcp
```

Reboot the server.

## Add the virtual functions to the Open vSwitch

* vi /etc/network/interfaces.d/ovs-sw1.conf

```bash
# Add VFs to the offloaded switch

auto ovs-sw1pf0vf0
iface ovs-sw1pf0vf0 inet manual
        ovs_type OVSPort
        ovs_bridge vmbr1
        ovs_mtu 9000

auto ovs-sw1pf0vf1
iface ovs-sw1pf0vf1 inet manual
        ovs_type OVSPort
        ovs_bridge vmbr1
        ovs_mtu 9000

auto ovs-sw1pf0vf2
iface ovs-sw1pf0vf2 inet manual
        ovs_type OVSPort
        ovs_bridge vmbr1
        ovs_mtu 9000

auto ovs-sw1pf0vf3
iface ovs-sw1pf0vf3 inet manual
        ovs_type OVSPort
        ovs_bridge vmbr1
        ovs_mtu 9000
```

After the changes, restart the server. To make sure the changes are applied.
When check the virtual functions.

```bash
ip -d link show | grep ovs-sw
```

Output should be like this.

```bash
14: ovs-sw1pf0vf0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq master ovs-system state UP mode DEFAULT group default qlen 1000
15: ovs-sw1pf0vf1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq master ovs-system state UP mode DEFAULT group default qlen 1000
16: ovs-sw1pf0vf2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq master ovs-system state UP mode DEFAULT group default qlen 1000
17: ovs-sw1pf0vf3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq master ovs-system state UP mode DEFAULT group default qlen 1000
```

Now we have bonding interface `bond1` and virtual functions `ovs-sw1pf0vf0`, `ovs-sw1pf0vf1`, `ovs-sw1pf0vf2`, `ovs-sw1pf0vf3` connected to the virtual switch `vmbr1` offloaded to the network hardware. We can use these interfaces in the virtual machines configuration, attached as a PCI device `0000:81:00.2` - `0000:81:00.5`.

## Configure the Proxmox

Let's create a resource mapping for the virtual functions.
Go to the Proxmox web interface, Datacenter -> Resource Mappings -> PCI Devices -> Add.
* Name: network
* Check all virtual functions starting from `0000:81:00.2` - `0000:81:00.5`

The ofical documentation you can find here https://pve.proxmox.com/pve-docs/pve-admin-guide.html#resource_mapping

## Configure the virtual machine

I hope you have a virtual machine already created.
Add the PCI devices to the virtual machine.

Go to the Proxmox web interface, Nodes -> your node -> Hardware -> PCI Devices -> Add.
* Mappped Device: network
* all options are default

After the changes, start the virtual machine.

## Troubleshooting

```bash
ovs-vsctl show
ovs-vsctl get Open_vSwitch . other_config
ovs-appctl bond/show bond1
ovs-appctl lacp/show bond1

ovs-dpctl dump-flows -m
ovs-appctl dpctl/dump-flows --names type=offloaded
```

## Resources

* https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Prerequisites
* https://pve.proxmox.com/wiki/PCI(e)_Passthrough
* https://hpcadvisorycouncil.atlassian.net/wiki/spaces/HPCWORKS/pages/1280442391/AMD+2nd+Gen+EPYC+CPU+Tuning+Guide+for+InfiniBand+HPC
