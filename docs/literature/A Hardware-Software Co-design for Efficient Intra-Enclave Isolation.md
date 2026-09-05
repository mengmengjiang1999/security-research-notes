# A Hardware-Software Co-design for Efficient Intra-Enclave Isolation

今天为大家推荐的工作是A Hardware-Software Co-design for Efficient Intra-Enclave Isolation，在SGX编程模型中应用MPK。


### 背景

SGX 在应用程序的地址空间中提供被称为 "enclave"的可信执行环境，以保护安全敏感的代码/数据。为了最大限度地减少代码重构工作，避免分解应用程序造成的性能损失，目前有一种流行的编程趋势，即在单个 SGX 硬件飞地内运行整个应用程序和第三方库甚至LibOS，但这样做会导致 TCB 变得臃肿，并危及敏感代码/数据。例如，一旦将包含漏洞的第三方库导入enclave，攻击者就可能利用漏洞篡改飞地的完整性甚至机密性。

MPK（Memory protection keys）是Intel公司在较新的x86-64架构的64位CPU推出的新硬件特性，最初的目标是提供轻量级用户模式下的控制页表权限的方式，以实现内存隔离。其工作原理是使用 4 个以前未使用的比特位为内存页标记保护密钥。一旦页面被标记，用户就可以在用户空间随意更改其保护权限。但是，由于更新 PTE 需要权限访问，因此首先仍需要调用系统调用来标记具有特定密钥的页面。

<!-- https://charlycst.github.io/posts/mpk/ -->

作者发现，现成的英特尔 MPK 非常适用于高效的内部隔离，如果可以将 MPK 应用到 SGX 上，将有望高效率地解决在 SGX 在应用中面临的问题。因为 MPK 在内存访问验证方面的开销几乎为零，而且两者都在用户级工作。然而，MPK 和 SGX 之间的信任模型在设计上是不兼容的。具体而言，一方面，MPK 要求信任底层操作系统，以正确的domain ID配置页表，这涉及到与 SGX 冲突的信任模型。不受信任的操作系统可以轻易修改domain ID 或禁用 MPK 检查，从而违反隔离规定。另一方面，MPK 支持用户级指令 WRPKRU，用于更改domain访问权限。enclave内的恶意实体（entity）可能会利用这一点绕过飞地内部隔离。 因此，该工作的核心技术挑战是如何在 SGX 飞地内安全地使用 MPK 进行飞地内隔离，同时面对不受信任的操作系统（操纵页表）和飞地内的恶意/受损实体（操纵域访问权限）。

### 贡献

该工作提出了一种软硬件协同设计的解决方案：LIGHT-ENCLAVE，用于实现安全、高效的飞地内部隔离。它在现有 SGX 硬件上进行非侵入式扩展，以解决 SGX 和 MPK 之间的信任模型冲突，以安全地集成 MPK，并允许在一个enclave内隔离多个light-enclave，从而将安全敏感的代码/数据与不信任的代码/数据隔离开来。与此同时，提供与现有开发流程兼容的友好的编程模型。

操作系统保留了在页表中配置 MPK 域标识的功能，但由于扩展的 SGX 硬件将在初始化和运行时验证域标识，因此操作系统无法任意修改 MPK 配置。有了 LIGHTENCLAVE，enclave开发人员可以为不同的enclave签署不同的内存域，并执行权限分离。为防止light-enclave提升自身权限（例如滥用WRPKRU），LIGHTENCLAVE将二进制检查与精心设计的light-enclave-gates结合起来。

LIGHTENCLAVE 具有以下优势：

- 与传统的多enclave隔离相比： LIGHTENCLAVE 通信效率更高。控制流传输是通过轻量级light-enclave-gates，而不是退出/重新进入硬件enclave。数据共享利用安全共享域，而不是通过未受保护的内存重新加密数据。
- 与基于仪器的方法相比： LIGHTENCLAVE 利用硬件强制权限检查代替边界检查，内存隔离的开销几乎为零，而且不需要连续的域区域。
- 与纯硬件方法相比，LIGHTENCLAVE 的性能更胜一筹： 由于采用了软硬件协同设计，LIGHTENCLAVE 更加灵活。底层硬件提供分割enclave域的机制，而软件则管理light-enclave的灵活抽象。light-enclave可以是互不信任的，也可以是分层的。


### 威胁模型
LIGHTENCLAVE 继承了 SGX 的威胁模型，假定对手可以完全控制除 SGX 飞地之外的所有软件（包括操作系统）。此外，它也不假定飞地内的所有代码都是可信的。
从一个light-enclave的角度来看，它需要信任SGX硬件、安全监视器和比它权限更高的light-enclave（如果存在的话）；它不需要信任任何其他软件，包括同一SGX enclave中的其他light-enclave。但是它需要保证自己是不会主动泄露秘密，因为LIGHTENCLAVE假定每个light-enclave都不会暴露自己的秘密，并且没有采取任何措施来消除这类软件缺陷。
此外，LIGHTENCLAVE 也不考虑硬件漏洞或侧信道攻击。

### 具体设计

1. LIGHTENCLAVE
LIGHTENCLAVE可以通过在enclave页面的PTE中标记（MPK）domain ID，将enclave内存划分为不同的内存域。一个light-enclave可以独占一个内存域（名为私有域）来存储其私有数据。light-enclave 只拥有其私有域的访问权限，这意味着不同的light-enclave 之间是互不信任的。不过，LIGHTENCLAVE 也允许一个light-enclave拥有比另一个light-enclave更高的权限，即一个light-enclave可以访问另一个light-enclave的私域，反之则不行。LIGHTENCLAVE 将一个light-enclave的数据、堆栈和堆栈放在自己的私有域中，而将其代码放在 domain-0 中。light-enclave永远无法获得domain-0 的访问（读/写）权限，因此无法修改代码。不过，它可以正常执行domain-0 中的代码。

2. 安全监视器
每个硬件enclave都包含一个安全监视器。正如其名称所示，安全监视器被认为是可信的，可以访问所有内存域。它的代码和数据都位于 domain-0，因此light-enclave无法访问。它是硬件enclave的管理器，职责包括在运行时创建新的light-enclave和动态飞地内存管理。

3. 使用模式
light-enclave有足够好的抽象，使得开发人员能够在enclave中应用最小特权原则。例如，安全敏感代码可以运行在一个light-enclave中，而其他库则位于另一个light-enclave中，每个light-enclave只有必要的权限。LIGHTENCLAVE扩展了英特尔 SGX SDK来将功能开放给程序员。它能自动分隔enclave内存，把light-enclave合并成一个硬件enclave，并为它们之间的交互生成light-enclave-gates。light-enclave-gates可以在两个light-enclave之间有效地转换控制流（切换执行上下文）以及切换身份（domain访问权限）。

4. SGX and MPK之间的信任模型不兼容的问题
SGX 和 MPK 之间的信任模型不兼容。要在 SGX enclave 中使用 MPK 进行内存隔离，enclave 页面的页表项应标记不同的domain ID。由于页表由操作系统控制，LIGHTENCLAVE 必须要求操作系统设置所需的域 ID。因此 MPK 的隐含假设是，操作系统是可信的，并会忠实地配置页表中的domain ID。然而，对于 SGX enclaves 而言，操作系统通常是不可信任的。因此，在 SGX enclave 中使用现有的基于 MPK 的内存异或是不可靠的。这一问题在该工作中通过硬件方案来进行解决，具体而言是实现了非侵入式 SGX 硬件扩展。

### 结果

实验表明，LIGHT-ENCLAVE在分离服务器应用程序的SSL密钥时产生的开销最多为4%，并能通过减少通信和运行时开销，显著提高Graphene-SGX和Occlum的性能。

由于 SGX 不是开源的，作者在官方 SGX 仿真器（英特尔 SGX SDK 仿真模式）上验证了 LIGHTENCLAVE 的硬件扩展，但增加了对 enclave 内存访问的检查，因为仿真器没有对其进行仿真。作者实现了 LIGHTENCLAVE 的软件设计，并将其应用于两个最先进的 SGX LibOS，即 Graphene-SGX 和 Occlum。

此外，作者在真正的 SGX 机器上进行了性能评估。对于 Graphene-SGX，LIGHTENCLAVE 在 CPU 密集型工作负载上实现了 10.5 倍的提速，在多任务密集型工作负载上实现了 46.5 倍的提速；对于 Occlum，提速分别为 1.49 倍和 1.28 倍。除了加速SGX LibOS外，LIGHTENCLAVE还能通过将服务器程序分解为多个组件来隔离不受信任的程序，从而提高服务器程序的安全性。对于保护隐私的无服务器应用程序，LIGHTENCLAVE 可将按需启动功能的延迟降低 50%-77%。


### 总结

该工作提出了如何在 SGX enclave 中安全使用 MPK 的软件硬件协同的解决方案，提出一个易于使用的抽象，名为 light-enclave，用于enclave内部隔离，并通过测试初步评估了 LIGHTENCLAVE 在不同情况下的性能优势。
