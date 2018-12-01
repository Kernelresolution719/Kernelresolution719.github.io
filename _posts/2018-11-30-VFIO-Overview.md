---
layout:     post
title:      VFIO Overview
subtitle:    "\"A simple analysis to VFIO\""
date:       2018-11-30
author:     Mijamind
header-img: img/post-bg-airplane.jpg
catalog: true
tags:
    - Virtualization
---
# VFIO Introduction
简单来说，VFIO(Virtual Function I/O)是一套完整的用户态驱动(userspace driver)框架，它通过定义vfio层面的容器(vfio_container)、组(vfio_group)、设备(vfio_device)等概念，对IOMMU组件和设备进行封装和描述，并实现了一系列ioctl接口安全地暴露给用户态使用。因此，用户态程序可以通过VFIO直接访问硬件设备。

# VFIO Framework
下图清晰地描述了VFIO的框架设计：
```
+-------------------------------------------+
|                                           |
|             VFIO Interface                |
|                                           |
+---------------------+---------------------+
|                     |                     |
|     vfio_iommu      |      vfio_pci       |
|                     |                     |
+---------------------+---------------------+
|                     |                     |
|    iommu driver     |    pci_bus driver   |
|                     |                     |
+---------------------+---------------------+
```
Top Layer是VFIO Interface，它负责向用户态暴露设备文件，用户态通过指定的ioctl方法操作VFIO注册的设备文件，从而达到设置和调用VFIO各种能力的目的。  
Middle Layer分别是vfio_iommu和vfio_pci，vfio_iommu是VFIO对iommu层的统一封装,主要用来实现DMA Remapping的功能，即管理IOMMU页表的能力。vfio_pci是VFIO对pci设备驱动的统一封装，它和用户态进程一起配合完成对设备的直接访问，具体包括PCI配置空间模拟，PCI MMIO空间重映射，Interrupt Remapping等。  
Bottom Layer则是硬件驱动层，iommu driver用于操作北桥上的IOMMU硬件单元(DMAR Unit)，其根据不同硬件平台有不同的实现方案，例如X86架构的intel iommu driver或ARM aarch64的smmu driver，而pci_bus driver作为一个内核最基本设备驱动，用来实现PCI设备的probe和register等操作。

# VFIO Logical Space
VFIO逻辑空间有3种基本元素：vfio_group，vfio_device，vfio_container，它们在逻辑上的关系如下图所示：
![](/img/vfio/vfio-logic-space.png)
vfio_group与iommu_group关联，是IOMMU进行DMA地址空间隔离的最小硬件单元，管理的是一个IOMMU硬件(DMAR Unit)。一个group内可能有一个或多个device，这取决于实际硬件平台上的设备拓扑。在实际应用场景中，同一个group内的device必须分配给同一台虚拟机(VM or guest)使用，不能让一个Group里的多个Device分别从属于多个不同的VM，也不允许部分Device在物理机(host)上而另一部分被分配到guest中，否则一个guest中的device在DMA时可以访问另外一个guest的地址空间，无法实现物理上的DMA地址空间隔离，这会造成一些非常严重的后果(如恶意攻击,非法访问等)。  
vfio_device对应实体硬件设备，不过这里的设备需要从IOMMU拓扑的角度去理解。如果该设备是一个硬件拓扑上独立的设备，那么它自己就构成一个iommu group。 如果该设备是一个multi-function(例如支持PCIe SRIOV)设备，那么它和其他的function一起组成一个iommu group，因为多个function设备在物理硬件上就是互联的，他们可以互相访问对方的数据，所以必须放到一个group里隔离起来。  
vfio_container是一个与VM相对应的概念，一个VM对应一个vfio_container，容器中记录了所有group的物理内存空间。  
从上图可以看出，一个或多个device从属于某个group，而一个或多个group又从属于一个container。如果要将一个device分配给VM，那么先要获取该设备所在的group，然后将整个group加入到container中。

# VFIO Hello World
以下代码为VFIO版的Hello World，取自linux内核VFIO文档说明[vfio.txt](https://www.kernel.org/doc/Documentation/vfio.txt)，该代码展示了如何编写一段用户态程序来访问硬件设备：
```c
	int container, group, device, i;

	/* Create a new container */
	container = open("/dev/vfio/vfio", O_RDWR);

	/* Open the group */
	group = open("/dev/vfio/26", O_RDWR);

	/* Add the group to the container */
	ioctl(group, VFIO_GROUP_SET_CONTAINER, &container);

	/* Allocate some space and setup a DMA mapping */
	dma_map.vaddr = mmap(0, 1024 * 1024, PROT_READ | PROT_WRITE,
			     MAP_PRIVATE | MAP_ANONYMOUS, 0, 0);
	dma_map.size = 1024 * 1024;
	dma_map.iova = 0; /* 1MB starting at 0x0 from device view */
	dma_map.flags = VFIO_DMA_MAP_FLAG_READ | VFIO_DMA_MAP_FLAG_WRITE;

	ioctl(container, VFIO_IOMMU_MAP_DMA, &dma_map);

	/* Get a file descriptor for the device */
	device = ioctl(group, VFIO_GROUP_GET_DEVICE_FD, "0000:06:0d.0");

	/* Test and setup the device */
	ioctl(device, VFIO_DEVICE_GET_INFO, &device_info);

	for (i = 0; i < device_info.num_regions; i++) {
		struct vfio_region_info reg = { .argsz = sizeof(reg) };

		reg.index = i;

		ioctl(device, VFIO_DEVICE_GET_REGION_INFO, &reg);

		/* Setup mappings... read/write offsets, mmaps
		 * For PCI devices, config space is a region */
	}

	for (i = 0; i < device_info.num_irqs; i++) {
		struct vfio_irq_info irq = { .argsz = sizeof(irq) };

		irq.index = i;

		ioctl(device, VFIO_DEVICE_GET_IRQ_INFO, &irq);

		/* Setup IRQs... eventfds, VFIO_DEVICE_SET_IRQS */
	}
```
该代码主要完成了以下几步工作：
```
1. 打开vfio设备文件，新建一个container
2. 打开一个group
3. 将group加入container
4. 为container分配相应的内存空间并设置DMA Mapping
5. 获取group下指定设备(BDF号为"0000:06:0d.0")的句柄
6. 获取设备的各个region信息，用于后续设备I/O空间访问
7. 获取设备的中断信息，用于后续的中断注册。
```
下面这张思维导图可以粗略地展示这段代码的底层工作逻辑：
![](/img/vfio/vfio-helloworld-relation.png)

与一般设备访问不同的是：
>
* 设备的句柄没有直接暴露在用户空间，需要从group中根据B:D:F号获取
* 设备的 I\O Region信息需要通过ioctl的方式获得
* 设备的中断信息需要通过ioctl的方式获得

# VFIO Usage
VFIO一般用于虚拟化场景的设备直通(pci passthrough)，说的通俗点就是通过VFIO将设备直接分配给虚拟机使用，以追求虚拟机使用的设备性能最优。这就会存在一系列的问题需要探讨:  
1.VFIO对完备的设备访问支持：其中包括MMIO， I/O Port，PCI 配置空间，PCI BAR空间：   
2.VFIO中高效的设备中断机制，其中包括MSI/MSI-X，Interrupt Remapping，以及Posted Interrupt等；  
3.VFIO对直通设备的热插拔支持。
这些问题在现有qemu(vfio_realiza)+VFIO的虚拟化实现方案中都有很详尽的参考，在此不再赘述，有兴趣可以阅读
[Insight Into VFIO](https://kernelgo.org/vfio-insight.html)。

# Reference
[VFIO Introduction](https://kernelgo.org/vfio-introduction.html)  
[Insight Into VFIO](https://kernelgo.org/vfio-insight.html)

# Personal Thinking
纵览现有虚拟化直通方案的实现，还是能观察到一些不足点，辟如PCI设备 MMIO空间的直通访问，这对设备硬件资源来说是一个较大的负担，尤其是multi-function场景下，需要硬件实现大量的资源（寄存器、SRAM）用于主机访问。如果这部分资源使用Hypervisor仿真实现应该会更好。
