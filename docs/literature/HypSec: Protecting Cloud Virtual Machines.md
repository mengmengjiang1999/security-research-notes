# HypSec

Protecting Cloud Virtual Machines from Commodity Hypervisor and Host Operating System Exploits


1. 魔改KBM，大幅度减小了TCB。TCB比KVM小很多个数量级
2. 主要功能是进行访问控制，加密依靠虚拟机或应用程序端到端加密
3. 核心管理程序可完全访问硬件资源，提供基本的 CPU 和内存虚拟化，并调解所有异常和中断，确保只有虚拟机和核心管理程序才能访问虚拟机在 CPU 和内存中的数据。更复杂的操作（包括 I/O 和中断虚拟化）以及资源管理（如 CPU 调度、内存管理和设备管理）则委托给主机管理程序，主机管理程序也可以利用主机操作系统。主机管理程序可以从系统中导入或导出加密的虚拟机数据，以启动虚拟机映像或支持快照和迁移等管理程序功能，但除此之外无法访问虚拟机数据。
4. HypSec并没有验证自己的TCB

## 相关工作

1. KVM/ARM’s split-mode virtualization [22,23]：使得KVM在不对Linux进行重大修改的情况下使用ARM硬件虚拟化扩展，但是并没有减小TCB
2. Nested Kernel：部分原因是其 TCB 和不受信任的组件都在最高硬件权限级别运行。相比之下，HypSec 取消了主机管理程序的权限，并使用其 TCB 提供数据保密性和完整性，即使主机管理程序中存在管理程序漏洞也不例外。
3. Bare-metal hypervisors：例如Xen，实际上TCB并没有比KVM更小。
4. Microhypervisors：microkernel方法，例如NOVA。确实减小了TCB，但是Hypsec相比之下保留了很多重要功能（动态资源分配等）
5. HyperLock [86], DeHype [88], and Nexen [70]：为每一个虚拟机实力都分配一个hypervisor function。缺点在于如果一个虚拟机实例存在可以利用的漏洞，那么很有可能在另外的虚拟机里也被利用。Nexen：以Nested Kernel为基础魔改Xen。以上系统只关注可用性，没有关注机密性、完整性。
6. Cloudvisor：需要对Xen进行修改。CloudVisor 对虚拟机的 I/O 和内存进行加密，但不能完全保护 CPU 状态，这与它声称的 "为虚拟机状态（包括 CPU 状态）提供保密性和完整性 "背道而驰。例如，虚拟机程序计数器暴露给 Xen 以支持 I/O。与任何嵌套虚拟化方法一样，应用程序工作负载的性能开销也是一个问题。此外，CloudVisor 不支持广泛使用的准虚拟 I/O。CloudVisor 的 TCB 较小，不支持公钥加密，因此密钥管理很成问题。相比之下，HypSec 通过访问控制（而不是加密）来保护 CPU 和内存状态，因此可以支持全功能的管理程序功能，如准虚拟 I/O。HypSec 还不需要嵌套虚拟化，避免了性能开销。
7. 虚拟机专用支持硬件：无法保护整个虚拟机。
8. Overshadow[16]和Inktag[33]等：假定应用程序使用端到端加密网络 I/O，不过它们通过将文件 I/O 替换为内存映射 I/O 到加密内存来保护文件 I/O。首先，HypSec 不使用内存加密，而是主要使用访问控制，这种方法更轻便，而且不需要模拟在内存加密时会出现问题的函数。其次，HypSec 依赖硬件虚拟化机制来干预相关的硬件事件，而不是检测或模拟复杂的系统调用。最后，HypSec 不是针对客户操作系统的漏洞，而是针对管理程序和主机操作系统的漏洞，而其他方法都无法做到这一点。


## 整体设计

分为corevisor和hostvisor。corevisor拥有所有的硬件权限，hostvisor负责资源分配和管理。

hostvisor并不拥有读data的权限，如果需要执行VM migration等需要用到data的操作，那么由corevisor负责将加密后的data传递给hostvisor，通过一个叫做GET VM STATE的hypercall api。hostvisor在设计上就是一个具备管理功能的比较特殊的VM。

TCB是corevisor，代码量较小。整体设计是corevisor如何保证hostvisor在不知道数据具体内容的情况下可以对VM的page分配等进行管理。

## 威胁模型


+ 攻击者可以远程访问管理程序及其虚拟机，包括无法实际访问机器的管理员。
+ 攻击者的目标是破坏虚拟机数据的机密性和完整性，其中包括：包含客户内核二进制的虚拟机启动映像、驻留在属于客户的内存地址中的数据、复制到硬件缓冲区的客户内存、虚拟机磁盘或文件系统上的数据以及存储在虚拟机 CPU 寄存器中的数据。虚拟机数据不包括通用虚拟硬件配置信息，如中央处理器电源管理状态或中断级别。
+ 攻击者可以利用主机管理程序或虚拟机管理接口中的漏洞访问虚拟机数据。例如，攻击者可利用主机管理程序中的漏洞执行任意代码，或从虚拟机或管理程序主机访问虚拟机内存。
+ 攻击者还可能控制外设，通过直接内存访问（DMA）执行恶意内存访问。
+ 假设提供虚拟机基础架构的整个云提供商不是恶意的
+ 远程攻击者无法对硬件进行物理访问，因此以下攻击不在攻击范围内：对硬件平台的物理篡改、冷启动攻击、内存总线窥探和物理内存访问等。
+ 虚拟机本身的漏洞不在本文考虑范围内
+ 攻击者可能会尝试从受损虚拟机攻击其他托管虚拟机。


## boot过程

corevisor boot：

1. 通过UEFI来验证binary是不是可信的，corevisor和hostvisor一起签名by cloud provider，签名之后的才是可信的
2. 使用hostvisor的boot code来install corevisor。具体而言，boot时hostvisor先拥有系统控制权，然后hostvisor来负责安装corevisor。由于此时网络和串口都还没有初始化，所以可以认为攻击者无法攻击corevisor的初始化过程。
3. corevisor初始化完成后会将hostvisor降低特权级（似乎是常见的hypervisor boot流程？

VMboot：

hostvisor call：VM CREATE，VM BOOT，VM ENTER。

The hostvisor is deprivileged and cannot execute VMs. It must call VM ENTER to request the corevisor to execute a VM.

VM exit 发生时如何处理：先trap到corevisor，然后corevisor交给hostvisor处理。

hostvisor喊corevisor来验证vmimage。
（这是什么意思？）If the VM kernel binary is in the VM disk image’s boot partition, HypSec-aware virtual firmware bootstraps the VM.
后面说有个固件来帮忙启动，这个固件也是经过了签名和验证的。这个固件把binary和bootloader从磁盘中加载进来，然后告诉corevisor去验证binary或者bootloader。

## CPU

+ handling traps from the VM;
+ privileged CPU instructions executed by the guest OS to ensure the hypervisor retains control of CPU hardware;
+ saving and restoring VM CPU state, including GPRs and system registers such as page table base registers, as needed when switching among VMs and between a VM and the hypervisor;
+ scheduling VCPUs on physical CPUs;


corevisor和hostvisor的分工：corevisor可以访问虚拟机 CPU 状态，处理trap，指令仿真，以及world switches between VMs and the hostvisor。hostvisor负责VCPU 调度，因为它可以在不访问虚拟机 CPU 状态的情况下完成。


corevisor负责保存与恢复上下文：设置寄存器，将trap和部分中断委托给corevisor。depriviledge hostvisor，确保hostvisor没有权限访问这部分数据。corevisor负责在VM进入和退出时保存其CPU上下文。同理也负责保存hostvisor的上下文。


hostvisor如何进行CPU调度（看不懂）

VM和hostvisor需要共享通用寄存器中存储的值时如何安全地共享：共享指令会trap到corevisor，然后corevisor 将识别需要传递给 hostvisor 的值，然后将这些值从 GPR 复制到主机可访问的内存中每个 VCPU 的中间虚拟机状态结构。vise versa。
corevisor根据具体的指令不同来决定传递哪些值。

## 内存管理


+ memory protection to ensure VMs cannot access unauthorized physical memory
+ memory allocation to provide physical memory to VMs
+ memory reclamation to reclaim physical memory from VMs


依靠嵌套虚拟化进行内存管理。
总体思路：guest OS管理guestOS的地址映射。hypervisor管理NPT：将gPA映射到hPA的页表，因此它可以读写存储在内存中的任何数据。
corevisor负责配置NPT硬件，具体的页表管理委托给hostvisor。HypSec要求corevisor和VM的内存不受hostvisor的影响。


### 内存保护

corevisor配置NPT。

hostvisor可以管理自己的页表，将hVA转化为vhPA。vhPA通过corevisor维护的嵌套页表转化为hPA。corevisor的页表会将每个 vhPA 映射到一个相同的 hPA。
主机管理程序对核心管理程序或虚拟机内存的任何访问都会trap到corevisor，使核心管理程序能够拦截未经授权的访问。

corevisor保留对IOMMU的控制权。

### 内存分配

hostVisor为每个VM管理一个虚假的NPT（叫做vNPT），corevisor为每个VM管理一个真正起作用的NPT（叫做sNPT）。sNPT是vNPT的影子页表。
实际上的修改在vNPT里面由hostvisor来做，每次进行修改sNPT都会进行修改。然后sNPT是嵌套页表，硬件根据这个嵌套页表来对VM进行地址映射。

只在sNPT到vNPT的地方使用了影子页表技术。（不太确定）

### 内存回收

当虚拟机释放内存时，corevisor首先擦除内存页内容，然后将内存页映射到hNPT。然后从sNPT中取消映射。

内存页换到磁盘：hostvisor可以通过GET VM STATE访问加密后的虚拟内存数据，然后将其交换出去。

### 共享内存

HypSec 提供 GRANT_MEM 和 REVOKE_MEMhypercalls，客户操作系统可以明确使用它们与主机管理程序共享内存。

虚拟机通过两个hypercall将希望共享的客户物理帧号（GFN）的起始点、内存区域的大小和指定的访问权限传递给corevisor。corevisor通过控制内存区域在 hNPT 中的映射来执行访问控制策略。只有虚拟机才能使用这两个超级调用，因此hostvisor无法使用它来请求访问任意虚拟机页面。


## 实现

在ARM上实现。corevisor运行在EL2。

HypSec 的核心管理程序在机器启动时初始化，并在 EL2 中运行，以完全控制硬件。
HypSec 的代码嵌入到 Linux 内核二进制文件中，并通过 UEFI 验证和加载。
内核在 EL2 启动，并安装一个trap处理程序，以便随后返回 EL2。
然后，内核进入 EL1，以便hostvisor启动机器。hostvisor为corevisor分配资源和配置硬件。然后，hostvisor 向 corevisor 发出hypercall，以启用 HypSec。
