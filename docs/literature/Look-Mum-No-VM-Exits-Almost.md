# jailhouse

Look Mum, no VM Exits! (Almost)

## 总体思路

+ 只进行隔离，不管调度。（怎么实现的？论文里完全没讲啊）
+ 只支持直接访问硬件，但是把资源进行分割，让不同的VM访问不同的硬件（但是这是怎么实现的？）
+ bare-metal hypervisor，但是需要Linux作为bootloader
+ 嵌入式虚拟化，关注low latencies, deterministic computation cycles and maintaining real-time capabilities。本文认为嵌入式虚拟化不能采用经典虚拟化的做法，已有的相关工作主要问题是采用了经典虚拟化方法，或者功能太冗余TCB过大。
+ 因为设计比较简单所以核心代码只有几千行，而且支持4种架构
+ 目前的部分设计实现不了主要是由于硬件的限制。

<!-- Jailhouse 通过内核模块从完全启动的 Linux 系统中启用，见图 1。它控制所有硬件资源，根据系统配置将其重新分配给 Linux，并将 Linux 提升到虚拟机（VM）状态。Jailhouse 的管理程序核心充当虚拟机监控器（VMM）。这种方案并不符合管理程序的传统分类[8]--它可以被视为第一类和第二类管理程序的混合体： 它像裸机管理程序一样在原始硬件上运行，没有底层系统层，但如果没有 Linux 作为系统助手提供初始化硬件，仍无法运行。Linux 用作引导加载器，但不用于运行。其他实时分区方法（如 PikeOS [10]）旨在管理硬件资源，可能会禁止客户系统直接访问，而 Jailhouse 不同，它只支持直接访问硬件。Jailhouse 不使用复杂耗时的（准）虚拟化[2]方案来模拟设备驱动程序和共享物理硬件资源，而是采用类似外核的方法[7]，即只提供隔离（通过利用虚拟化扩展），但有意不提供调度程序或虚拟 CPU。根据硬件支持情况，只有（少数）还不能以这种方式分区的资源才在软件中进行虚拟化。
出于成本效益的考虑，许多工业应用系统都不能放弃 Linux 的功能和特性，但它们又面临着越来越多的要求，即同时满足 Linux 难以达到的安全或其他认证要求。我们的架构方法满足了这些需求。不过，我们认为它也是一个理想的框架，可以方便地将最先进的研究或实验系统与基于 Linux 的工业级解决方案整合在一起，从而以新颖的方式解决特定的问题。
在本文中，我们将介绍
- Jailhouse 的体系结构，它是一个在多种体系结构上运行的全功能、非调度、实时、静态分区的开源管理程序。
- 一个作为 Jailhouse guest 运行的非关键现实世界混合关键性应用程序的实现。
- 延迟启动管理程序的优势。
- Nvidia Jetson TK1 上中断系统的典型微基准测试 -->

## 设计

1. 硬件初始化由 Linux 完成，Jailhouse 可以完全专注于管理虚拟化扩展。
2. exohypervisor，将物理资源分配给多个VM并进行隔离，每个VM自行决定如何使用。
3. 假设物理资源无法在VM之间共享。如果需要创建一个新的house，那么需要释放部分资源，然后分配给新的house。Linux负责管理CPU资源及其分配，hypervisor负责监控没有非法访问的情况。虚拟化扩展可以保证如果发生了非法访问，会发生trap。

Jailhouse希望达到的理想设计是启动和进行分区之后就不管了，只需要在发生违规访问资源的时候介入。但是由于硬件架构的设计问题，这种方案无法实现，

• Interrupt reinjection (depending on the architecture, interrupts may not directly arrive at guests)
• Interception of non-virtualisable hardware resources (e.g., parts of the Generic Interrupt Controller (GIC) on ARM)
• Access of platform specifics (e.g., accessing Control Coprocessor CP15 or Power State Control Interface (PSCI) on ARM)
• Emulation of certain instructions (e.g., cpuid on x86)

此外以下trap也无法避免
• Access violations (memory, I/O ports)
• Cell management (e.g., creating, starting, stopping or destroying cells)

除了Linux外还有很多可以作为Guests


## 性能测试

It is important to remark that such benchmarks do not measure the overhead of the hypervisor, but the overhead of the hypervisor when running on a specific hardware platform.
