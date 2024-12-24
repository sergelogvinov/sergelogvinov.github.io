# Proxmox Virtual Machine optimization

Proxmox Virtual Environment (VE) is a powerful open-source virtualization platform used to manage virtual machines (VMs). To make your VMs run faster, you need to set them up correctly.

## Common optimizations

* [CPU settings](#cpu-settings)
* [Memory settings](#memory-settings)
* [Network configuration](#network-configuration)
* [Disk storage type](#disk-storage-type)

## CPU Settings

### CPU Type

If you do not want to use live migration, you need to set the CPU type to `host`. This setting allows the VM to use all features of the physical CPU but disables live migration entirely, which might not be suitable for all environments.

### CPU Affinity

Set dedicated CPU by setting CPU affinity for the VM. This setting will prevent the VM from using other CPUs.

Turn on NUMA (Non-Uniform Memory Access) for better memory handling on large servers. Choose the right CPU cores in one NUMA node.

[More details](https://dev.to/sergelogvinov/proxmox-cpu-affinity-for-vms-4dhb)

## Memory Settings

### Memory Huge Pages

Consider using huge pages for better memory performance. Huge pages are larger than normal pages and can improve memory access speed. It allocates memory in 2MB or 1GB chunks, dyring the boot process. And this memory is reserved for the VM only, the hypervisor cannot use it.

[More details](https://dev.to/sergelogvinov/proxmox-hugepages-for-vms-1fh3)

## Network Configuration

### Use VirtIO Network Driver

Always use VirtIO drivers for network cards. VirtIO drivers are paravirtualized drivers for network and disk devices. They are faster than the default drivers. Enable Multiqueue queues the same as the number of CPU cores.

### Enable Jumbo Frames

Allow Jumbo Frames for larger network packets. You can enable Jumbo Frames in Proxmox by setting the MTU (Maximum Transmission Unit) value to 9000 on the network interface configuration.

### SR-IOV

Use Single Root I/O Virtualization (SR-IOV) for better network performance. SR-IOV allows a single physical network card to appear as multiple virtual network cards. After enabling SR-IOV, you can assign a virtual function to the VM. VM will have direct access to the physical network card. The Hypervisor will not be involved in network processing.

[More details](https://dev.to/sergelogvinov/network-performance-optimization-with-nvidia-connectx-on-proxmox-5f7j)

## Disk Storage Type

The disk storage type is one of the most important factors for VM performance.
Local storage is faster than network storage. SSD is faster than HDD.
LVM (not a thin) is faster than ZFS. ZFS is faster than NFS.

For the safety of your data, use the soft RAID 1 or 10 for the local storage.
* for 2 disks use RAID 1 with mdraid implementation
* for 4 disks or more try to use ZFS

LVM backend is faster than other backends for the local storage.
ZFS backend works better with large storage, but it requires more memory and CPU.

### Use VirtIO/SCSI VirtIO drivers

Always use VirtIO drivers for disk devices. VirtIO drivers are paravirtualized drivers for disk devices. They are faster than the default drivers.

### SR-IOV

For the huge storage performance, you can use SR-IOV for disk devices. You can assign NVME directly to the VM. The VM will have direct access to the physical disk, this performance will have performance as on the host machine.
