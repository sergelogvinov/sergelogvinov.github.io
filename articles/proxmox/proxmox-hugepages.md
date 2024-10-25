# Proxmox HugePages for VMs

## Introduction

When an application needs to access values of the variables, it asks the CPU to fetch it from memory using variable address. This address is virtual, meaning the CPU has to translate it into a physical address (the actual location on the server's memory chip). The CPU handles memory in blocks called `pages`, and the page size depends on the CPU's architecture. For example, in x86_64 systems, each page is typically 4KB. So, if a server has 16GB of memory, it will have 4 million pages to store.

This can become a problem on servers with large amounts of memory because the CPU has to manage so many pages. The process of translating virtual addresses into physical addresses can slow down. To improve performance, HugePages were introduced.

A HugePage  is a page that is larger than the default page size. For x86_64 systems, HugePages can be either 2MB or 1GB. With HugePages, the CPU has fewer pages to manage, which speeds up performance.

For instance, if you create a virtual machine (VM) with 1GB of memory, the Linux system would create about 256,000 pages by default. But if you use 1GB HugePages, only one page would be needed. This reduces the number of pages the CPU has to manage, improving performance.

## How to configure HugePages

Check what HugePage sizes are supported by your system by running the following command:

```bash
ls /sys/devices/system/node/node0/hugepages/
```

Output for AMD EPYC:

```bash
hugepages-1048576kB  hugepages-2048kB
```

Output for ARM Ampere Altra:

```bash
hugepages-1048576kB  hugepages-2048kB  hugepages-32768kB  hugepages-64kB
```

It means that the AMD system supports 4KB, 2MB, and 1GB HugePages. We will use `1GB` HugePages for this example.

To configure HugePages, you have to edit the `/etc/default/grub` file and add the `default_hugepagesz` and `hugepagesz` parameters to the `GRUB_CMDLINE_LINUX` variable. We will set the HugePage size to 1GB and the number of HugePages to 16.

```bash
GRUB_CMDLINE_LINUX="default_hugepagesz=1G hugepagesz=1G hugepages=16"
```

For NUMA architecture, you have to set the `hugepagesz` and `hugepages` parameters for each node. For example, if you have 2 nodes, you have to set the `hugepages` parameters for each node, splitting by a comma `$HUNODE:$AMOUNT,$HUNODE:$AMOUNT`.

```bash
GRUB_CMDLINE_LINUX="default_hugepagesz=1G hugepagesz=1G hugepages=0:8,1:8"
```

The Linux kernel reserves 16GB of memory for HugePages, but this reserved memory is exclusively for HugePages and won’t be available to other applications. You need to calculate the number of HugePages based on your server’s total memory, ensuring there is enough left for the operating system and other applications. The operating system should have at least 1GB of free memory. If you're using ZFS or Ceph, make sure to reserve additional memory for them.

After you edit the `/etc/default/grub` file, you have to update the grub configuration file by running the following command:

```bash
update-grub
```

And don’t forget to reboot the server to apply these changes.
Check the HugePages by running the following command:

```bash
cat /proc/meminfo | grep Huge
```

Output:

```bash
AnonHugePages:         0 kB
ShmemHugePages:        0 kB
FileHugePages:         0 kB
HugePages_Total:     448
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:    1048576 kB
Hugetlb:        469762048 kB
```

If you have NUMA architecture, you have to check the HugePages for each node by running the following command:

```bash
numastat -cm | egrep "Huge|Node|Mem"
```

Output:

```bash
                 Node 0 Node 1 Node 2 Node 3  Total
MemTotal         128655 129020 129020 128977 515672
MemFree           12654  12933  12643  13139  51369
MemUsed          116001 116087 116377 115838 464303
AnonHugePages         0      0      0      0      0
ShmemHugePages        0      0      0      0      0
HugePages_Total  114688 114688 114688 114688 458752
HugePages_Free        0      0      0      0      0
HugePages_Surp        0      0      0      0      0
```

In the output, we have 4 nodes with 128Gb for each node, and node has 112 HugePages (112 * 1024 = 114688 MB)

## How to configure HugePages for VMs

Most of the HugePages parameters do not accessible from the Proxmox GUI. You have to configure them through the command line or API.

Here we will use 1GB HugePages size, and amount of HugePages is depending on the VM memory parameter (8192 MB in this example - 8 HugePages).

```yaml
# VM config /etc/pve/qemu-server/$ID.conf
memory: 8192
hugepages: 1024
keephugepages: 1
```

HugePages are dedicated memory pages assigned to a specific VM, meaning other VMs cannot access the same HugePages. The `keephugepages` parameter allows HugePages to remain reserved even after the VM is stopped. This dedicated memory capabilities not only boosts performance but also enhances the VM's security.

Some applications, like databases, can use HugePages to store data in memory. This helps them access data faster and improves performance. To allow HugePages in a VM, just add the CPU flag `+pdpe1gb` to the VM's configuration file.

```yaml
# VM config /etc/pve/qemu-server/$ID.conf
cpu: host,flags=+pdpe1gb;
```

## Conclusion

HugePages enhance system performance by reducing the number of memory pages the CPU has to manage. With fewer pages to handle, the CPU operates more efficiently, leading to improved overall system performance.

## References

* https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt
* https://pve.proxmox.com/wiki/Manual:_qm.conf
