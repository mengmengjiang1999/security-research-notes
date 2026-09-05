# HyperEnclave

## Background

这篇文章的工作背景是TEE，可信执行环境。

TEE会提供一个硬件隔离的环境，称为enclave，在enclave中的隐私数据可以被很好保护。这个独立的区域可能是一个进程的地址空间（例如Intel SGX enclave），可能是一个独立的virtual machine（例如AMD SEV），也可以是一个和普通的OS不同的一个独立的系统（例如ARM TrustZone）

TEE的攻击者可能不仅来自OS级别，也可能是来自平台的物理攻击。

## Motivation

接下来介绍一下Motivation，就是现有的TEE仍然存在的一些问题。
1，大多数TEE技术都没有开源，而且需要一定的硬件或者固件的修改。编程难度大，而且不易验证

2，大多数现有的TEE都要求enclave运行在特定的模式下，使得envalve难以根据实际的情况进行修改自己的运行模式。
以Intel SGX enclaves为例，它运行在user mode，并不能访问例如页表这样的privileged资源。因此，运行某些特定类型的负载会导致较为明显的性能损失。

## Design Goals

介绍一下平台的设计目标

1. 首先是对硬件依赖最小。目标是支持x86和ARM，不需要特殊的硬件feature，只要求虚拟化扩展和TPM。

2. 第二是SDK和现有的Intel SDK保持一致，方便现有的SGX程序迁移，只需要进行很少的改动。

3. 第三是支持不同的elclave运行模式，指的是允许enclave运行在不同的特权级下。

## Threat Model

接下来介绍的是威胁模型，有以下几个要点

1. 信任底层硬件
In our threat model, we trust the underlying hardware.

2. 假设系统在boot期间不会受到物理攻击，但是在boot之后可能被攻击

3. 假设application, the OS kernel, and the enclave code都可能是恶意的

4. 考虑特定的物理内存攻击和基于页表的侧信道攻击，不考虑DoS攻击和其他侧信道攻击
We also consider certain physical memory attacks and page-table-based side channel attacks.
We don’t consider DoS attacks and other side channel attacks.


## HyperEnclave Overview

HyperEnclave整体设计。

总体的设计思路是，利用硬件虚拟化特性的二级地址转换来实现隔离。

HyperEnclave的软件有三个部分，运行在三种不同的模式下：

1. 最高特权级下运行的是轻量级的hypervisor叫RustMonitor，实现的功能是管理enclave的内存，强制进行内存隔离，控制enclave的状态转换等。
It works as a resource monitor, while complicated tasks are offloaded to the primary OS.

2. primary OS，例如一个Linux，作为一个被RustMonitor创建的特殊的guest VM运行在normal mode。primary OS负责运行应用不被信任的部分。

3. secure mode模式（也称为enclave）会运行应用被信任的部分。secure mode可以根据enclave的实际需求，选择不同的硬件特权级。
VMX non-root, ring-3 (GU-Enclave)
VMX non-root, ring-0 (P-Enclave)
VMX root, ring-3 (HU-Enclave)
例如，P-Enclave can run in ring-zero of the VMX root operation, 可以访问guest page table。


## Design Details

这一部分讲的是本文的四个重要的解决的问题。第一个是内存管理和保护，第二个是灵活的enclave运行模式，第三个是受信任的Boot过程，第四个是enclave SDK。

1. Memory management and protection
2. Flexible enclave operation mode
3. Trusted Boot
4. The enclave SDK


## Memory Management and Protection

### 现有的内存管理及其问题

以Intel SGX为例。Intel SGX 允许不被信任的OS来管理enclave的页表。为了防止mapping attack，（注：mapping attack说的是把两个elclave的地址map到同一块物理内存上）SGX使用了EPCM来保证内存隔离是合法的。

如果没有EPCM硬件支持的话，SGX提供的软件方案是修改page table权限为只读，这样untrusted OS每次试图修改页表时，就会发生trap，但是这样会带来不可忽视的开销。
此外，由于enclave的page fault也是被untrusted OS处理的，可能会被基于page table的侧信道攻击。


### HyperEnclave的设计


1. 为了解决上述问题，HyperEnclave提出了如下方案：为enclave创建一个页表，RustMonitor负责管理这个页表，处理pagefault，不再由primary OS来参与这一过程。

2. 这种设计保证了enclave的地址映射的安全性，而且并不需要硬件的支持。而且由于RustMonitor负责管理page fault，就可以有效防止基于页表的攻击。

3. 支持Enclave动态内存管理。（没看懂）通过OCALL把内存管理请求发送到SGX，然后再验证这个修改。
In addition, this design makes it easy to support enclave dynamic memory management, more details are in our paper.

On SGX2 platforms, the enclaves need to send the EDMM request to the SGX driver through OCALLs, who then makes the requested changes. Since the driver is untrusted by the enclaves, the changes need to be explicitly checked and ac- cepted by the enclaves to take effect, which involves heavy enclave mode switches.


### 由HyperEnclave管理页表带来的新问题

enclave可以访问应用的整个地址空间，那么enclave的页表也需要保留一部分应用的地址映射。但是如果OS改变了内存映射（例如产生了page交换），RustMonitor管理的enclave page table也需要同步更新mapping。

为了减少这种同步更新带来的开销，enclave的page table里面不会维护app的内存映射。替代方案是在app的内存中分配一个marshaling buffer，和enclave共享。marshaling buffer和物理内存的映射在整个enclave的生命周期都不会改变，所有enclave和app之间的数据交换都通过marshaling buffer进行。
这种设计也可以减少enclave对应用的攻击。


### 内存管理小结

总之，HyperEnclave使用了二级地址转换实现了以下几点安全要求：

1. The primary OS and apps 不可以访问属于 RustMonitor and the enclaves 的物理内存。
2. 每一个enclave都不能访问属于 RustMonitor 和其他 enclave 的物理内存。
3. DMA也不能访问属于 RustMonitor 和 enclave 的物理内存，这一点由IOMMU来保证。


## flexible enclave operation mode

下面讲的是flexible enclave operation mode，就是说enclave可以根据workload的特点来选择运行在不同的mode下。

为什么要做这样的设计：
TEE的负载有不同的特性，例如computing-intensive，IO intensive，memory-intensive。
然而，现有的TEE的设计只支持enclave跑在特定的模式下，不能对不同特性的enclave有很好的适配。
例如，Intel SGX enclaves 跑在user mode。因此如果程序中有大量的特权指令，就会频繁进出enclave，产生大量开销。

为了解决以上问题，HyperEnclave支持三种不同的运行模式。（如图所示）


### GU-Enclave

1. GU-Enclave, or guest user enclave, is the basic form that is typically running computing-intensive tasks.
The enclave runs in the guest user mode, that is, ring-3 of the VMX non-root operation mode.

2. RustMonitor为GU-Enclave创建vCPU和GPT、NPT。

3. handle the interrupts and exceptions during the enclave running：

    + RustMonitor configures the vCPU to trap all interrupts and exceptions to the monitor mode

    + RustMonitor then saves the enclave’s context, forwards the interrupt or excep- tion to the normal VM

    + primary OS处理完成后，再返回enclave中


### P-Enclave

1. P-Enclaves which run in guest privileged mode. P-Enclave can access the privileged resources, such as the IDT, and the level-1 page table, which benefits a wide variety of applications.
One such example is the garbage collector, it frequently changes page permissions to trigger page faults to track the page status
For user mode enclaves, it has to involve the primary OS to update the page table and handle the page fault which suffers huge performance loss.
可以在enclave内部处理一些异常，减少world switch，以减少开销

2. P-Enclaves can eliminate the world switch by supporting in-enclave exception handling and level-1 page table management.

3. Furthermore, P-Enclaves can also support page-table-based in-enclave isolation schemes, for example, sandboxing untrusted third-party libraries.


### HU-Enclave

1. host user enclave is inspired by the fact that the ring switch is faster than the guest-host mode switch.

It uses syscall and syscall return instruction to exit and enter the enclave, which is seven hundred CPU cycles faster than the hypercall instruction.

2. It further eliminates the extra virtualization overhead, for example, vCPU context switching and the two-dimensional page walking.
HU-Enclave is suitable for IO intensive workloads, which have frequent world switches, and memory-intensive workloads, which have high TLB pressure.

3. 运行方式：
    + RustMonitor来准备一个process context，调用sysret来进入HU-Enclave
    + HU-Enclave调用syscall来返回RustMonitor
    + 中断异常都给RustMonitor处理

HU-Enclave切换比较快，因此IO比较多的时候比P-Enclave有用。P-Enclave在异常处理时比较有用。GU-Enclave比较普通，好像没什么特别的。

## Trusted Boot

The boot process starts with the CRTM code, it builds a measurement chain for subsequent firmware and software, including the BIOS, grub, the primary OS kernel, and initramfs. The measurement results are stored to TPM PCRs.
RustMonitor is launched by the kernel module in early userspace, which means it starts before any userspace program runs that relies on the disk file system or networks.
Then RustMonitor demotes the primary OS to the normal mode, and launches other user space services.

## The enclave SDK

API-compatible with the official Intel SGX SDK

只要By replacing the SGX user leaf functions (e.g., EENTER, EEXIT, and ERESUME) with hypercalls，Existing apps developed by the SDK can run on HyperEnclave with little or no code changes. 为了验证这一点，我们已经迁移了一部分SGX工具链到hyperenclave，例如Rust SGX SDK，以及the Occulm library OS。

Primary OS通过hypercall来实现进入和退出RustMonitor，并将这些接口通过ioctl()等暴露给应用程序，以此让RustMonitor来管理enclave的生命周期。

此外，大多数的data structure都与SGX中保持一致。


## Implementation

（看PPT）

## Evaluation

接下来介绍性能测试的结果。

### Methodology

在此之前首先需要对性能测试采用的方法进行说明。
Platform A: AMD EPYC 7601, 512 GB RAM, with HyperEnclave and SME
Platform B: Intel Xeon E3-1270 v6, 64 GB RAM, with SGX
如何在不同平台下比较性能？
本文采用的测试方法是测量性能的相对下降程度。既然是不同的平台，那么性能数据无法被直接用来比较。因此，本文比较的是相对差值。例如这个图中，the relative slowdown on Platform A is twenty percent, and on platform B is twenty-five percent. 因此，我们认为A方案优于B。

### 测试

1. World switches performance

这里测量的是world切换的开销，包括指令和函数级别的latency。结果表明频繁产生特权级切换的应用场景，HU-Enclave性能最好，因为它将hypercall变成了syscall。所有的结果都比Intel SGX要好。

2. P-Enclave use cases: exception handling

这里使用exception handling频繁的workload对Intel SGX和HyperEnclave进行测试。
首先让enclave出发一个undefined instruction exception，然后exception handler就会将指令地址转发，并返回。在Intel SGX and GU-Enclave中，处理此异常都需要经过4次world switch，但是对P-Enclave来说，可以不经过world switche，而直接在enclave中进行一场处理。因此性能测试结果显示，P-Enclave比GU-Enclave快了70倍。

接下来的这个程序模拟的是垃圾回收。（有点没懂）
The enclave first removes the write permission to the buffer by changing the enclave’s page table. After that, the enclave access the buffer to trigger the page faults, and the write permission is restored in the exception handler.
For GU-Enclave, it needs to trap into RustMonitor to update the page tables, while P-Enclave can update page tables and handle page faults by itself. P-Enclave is more than twice faster than GU-Enclave.

3. Real-world workloads

最后测试真实场景中的workload，把SQLite再内存中进行测试，作为memory-intensive workloads。在
在Intel平台上，当内存用量较小时，相对的性能下降时25%。但是当内存用量超过硬件的EPCsize时，相对性能下降会显著增加（由于EPC page swapping）。
相比之下在HyperEnclave上,仅有5%的性能下降，且不会随着内存用量增加而产生改变，因为we can use a larger EPC size on HyperEnclave.

此外还测试了轻量级的web server Lighty，以在Occulm library OS上测试了Redis。
这两种workload的特点都是frequent network requests以及frequent enclave mode switches.
HU-Enclave delivers the best performance，符合预期。


## Conclusion

最后进行一下总结：

1. 提出HyperEnclave的解决方案，an open TEE that can run on various platforms with minimum hardware requirements.

2. HyperEnclave支持the process-based TEE model, and SGX programs can run on HyperEnclave with little or no source code changes.

3.  HyperEnclave supports the flexible enclave operation modes to fulfill various enclave workloads.