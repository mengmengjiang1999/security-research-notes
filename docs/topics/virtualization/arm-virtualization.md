arm hypervisor文档

https://developer.arm.com/documentation/100942/0100/Hypervisor-software

Virtualization support is provided in the Non-secure state only. There is no virtualization support in the Secure state, because TrustZone technology is intended to allow a Secure or trusted environment. This implies a small code base, to enable validation and certification.

（secure state：具体而言指的是TrustZone。因为SMP已经把EL2占了。）

When hypervisor code in EL2 is executing in AArch64, there are dedicated registers available, including:

Exception return state registers: SPSR_EL2 and ELR_EL2.
Stack pointer: SP_EL2 (and SP_EL0).


虚拟异常
ARM支持三种虚拟异常 Virtual SError, Virtual IRQ, and Virtual FIQ


gPA到hPA的转换使用虚拟化转换表基础寄存器 VTTBR_EL2 和 VTCR_EL2，由管理程序控制。


ARMv8-A 虚拟化还引入了虚拟机 ID (VMID) 的概念。每个虚拟机都分配有一个 VMID，它是一个存储在 VTTBR_EL2 中的 8 位值。

这些值用于标记属于特定虚拟机的翻译。对于访客访问，处理器 MMU 中的翻译旁路缓冲区 (TLB) 可以在一个条目中存储完整的 VA→IPA→PA 翻译。VMID 可确保只有正确的虚拟机才能访问 TLB 条目，这样就无需在客户操作系统之间进行上下文切换时使 TLB 失效。

架构没有定义 VTTBR_EL2.VMID 字段在硬件中的重置值。这意味着在每个活动内核上运行的启动代码软件必须将 VTTBR_EL2.VMID 字段初始化为已知值，例如 0，即使没有使用第二阶段转换也是如此。

（有意思，但是暂时并不重要）
