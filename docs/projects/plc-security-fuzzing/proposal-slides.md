---
marp: true
theme: default
paginate: true
_paginate: false
header: ''
footer: ''
backgroundColor: white
---

<!-- theme: gaia -->
<!-- _class: lead -->


# 开题报告
## 基于覆盖率反馈的PLC安全性分析方法研究

<br>
<br>

2024年6月1日

---
## 目录

- 研究背景
- 研究目标
- 相关工作
- 研究内容
- 预期成果
- 研究计划
---
## 研究背景和相关工作
### 工业控制系统与PLC

工业控制系统（Industrial Control Systems, ICS）在与民生密切相关的基础行业中广泛部署，如能源、交通、水利、电力、石油等。随着互联网的快速发展，越来越多的工业控制系统通过通用协议连接到互联网。然而，这些系统在设计之初并未考虑连接到互联网的需求，因此其采用的协议缺乏加密、授权、认证等机制，存在许多漏洞。

---
### PLC的具体结构

<style scoped>
section li {
  font-size: 28px;
  line-height: 1;
}
</style>

PLC（Programmable Logic Controller）是工业控制系统的核心，其硬件由CPU、电源模块、I/O模块、内存和扩展模块组成，结构与传统计算机基本相同。
循环扫描是PLC执行其控制程序的基本工作方式。在这种模式下，PLC按照以下步骤重复执行一系列操作：

+ 输入扫描：PLC读取所有输入端口的状态，这些输入可能来自传感器、按钮或其他设备。

+ 程序执行：PLC根据输入值和预设的逻辑来执行存储在内存中的程序。这包括执行所有的逻辑运算、定时器、计数器和其他控制指令。

+ 输出更新：程序执行完成后，PLC更新输出端口的状态，这些输出控制着执行器、指示灯或其他设备。

+ 循环重复：PLC返回第一步，再次读取输入，形成一个连续的循环。

---

### PLC编程语言

<style scoped>
section li {
  font-size: 28px;
  line-height: 1;
}
</style>

PLC编程语言是专门为编写控制逻辑而设计的，以便管理和监控工业自动化过程。根据国际电工委员会制定的工业控制编程语言标准（IEC1131-3），PLC有五种标准编程语言：

+ **梯形图语言（Ladder Diagram, LD）**：这是**最常用的PLC编程语言**，它使用类似于电路图的图形符号，通过视觉化的方式来表示控制逻辑。

+ 指令列表语言（Instruction List, IL）：这种语言使用一系列的指令来编程PLC，类似于汇编语言。

+ 功能块图语言（Function Block Diagram, FBD）：这种语言使用功能块来表示操作，并通过图形化的方式将它们连接起来，以创建控制逻辑。

+ 顺序功能流程图语言（Sequential Function Chart, SFC）：也称为SFC图，它用于描述顺序控制过程，通过一系列的步骤和转换来表示过程的流程。

+ PLC ST（Structured Text）：以类似于Pascal或C语言的文本格式出现，是一种结构化的编程语言，非常适合复杂的算法和数据处理任务。

---

### PLC安全威胁

<style scoped>
section li {
  font-size: 32px;
  line-height: 1;
}
</style>

PLC常见的安全威胁有以下几类：代码安全，固件安全，网络安全，协议安全，HMI安全等等。

代码安全相关：

+ 逻辑错误：可能导致计时器振荡、无条件转移使得某些分支无法到达、代码无法通过正常控制流程到达、无限循环导致循环超时错误等。
+ 语法错误：例如比较器硬编码，容易受到攻击并改变过程。
+ 作用域和链接错误：缺少跳转和链接，可能被嵌入恶意程序。
+ 隐藏跳转器：可以改变程序执行的过程。
+ 重复对象：对象重复引用，可能导致输出失败。
+ 未使用对象：未使用的对象容易被攻击者利用。

---

### PLC安全威胁

PLC常见的安全威胁有以下几类：代码安全，固件安全，网络安全，协议安全，HMI安全等等。

固件安全相关：（这一段应该不需要写特别多，写一点点就行）

根据文章内容，提到的固件安全问题主要包括：

1. **固件修改攻击**：许多PLC允许用户远程下载固件更新，但固件验证通常采用的CRC校验和算法无法检测固件是否被故意修改。攻击者可以通过重新打包修改后的固件对PLC实施攻击。

2. **固件完整性检查不严格**：PLC的固件一般支持在线更新，但完整性检查通常不够严格，这使得固件容易被黑客篡改。

3. **固件安全性检测方法**：文章提到了设计可信PLC的方法，使用信任芯片和信任度量根，结合哈希算法，在系统启动阶段通过信任链传递来实现PLC的可信启动。

4. **认证机制**：PLC系统启动文件包括BOOT.BIN、devicetree.dtb、uImage和uramdisk.image。信任链依靠固件代码BL0作为信任基础，启动链接测量下一个启动的安全性，并将度量值扩展到PCR。只有当下一个启动链接安全时，控制权才会转移。

5. **完整性检查**：设备上电时，系统启动顺序调用每个文件的哈希码段执行完整性检查。如果哈希值与之前存储的标准值不一致，文件将被直接删除。所有启动文件都将依次经过验证，确保一旦有一个启动文件被修改，设备将无法启动，从而保证PLC固件的安全性。

6. **固件签名检测方法**：基于固件签名的检测方法允许在运行时检查固件。该技术基于高效代码认证（ECC）技术，当加载固件模块时，将根据标准安全策略进行验证。然而，这种方法只针对恶意固件，对于恶意硬件和拒绝服务攻击则无法检测。

文章强调了固件在PLC安全中的重要作用，并提出了一些检测和保护方法，以防止固件被篡改或攻击，从而确保工业控制系统的完整性和可靠性。

---

### PLC安全威胁

PLC常见的安全威胁有以下几类：代码安全，固件安全，网络安全，协议安全，HMI安全等等。网络安全相关：

1. **SCADA系统威胁**：SCADA系统负责现场数据采集和监控，与PLC之间的通信存在安全问题。一旦SCADA系统连接到互联网，潜在威胁更可能被攻击者利用。

2. **协议漏洞**：文章提到了Modbus、DNP3和Allen Bradley等SCADA采用的协议，它们容易受到包括响应注入、命令注入、中间人攻击、重放攻击和拒绝服务攻击等安全威胁。

3. **重放攻击**：攻击者可以通过捕获或在同一网络中的另一主机上进行重放攻击，这些攻击能够控制PLC的启动和停止。

4. **中间人攻击**：利用TCP/IP和COTP协议以及以太网的ARP地址解析协议，攻击者可以被动地监控PLC通信流量或主动地篡改数据和命令，以控制PLC并干扰正常系统操作。

5. **隐蔽命令修改攻击**：结合了重放攻击和中间人攻击，攻击者可以获取PCS7与PLC之间的通信，并在隐蔽模式下重放其他命令以干扰PLC操作。

6. **PLC蠕虫病毒**：由于PLC采用周期性扫描，每次执行程序后会刷新输入和存储区域，这为攻击者提供了修改用户程序内容的机会。

7. **网络攻击检测**：文章中提到了Chen等人提出的工业控制网络的安全监管程序，该程序采用三种数据采集方案，能够深入到控制系统的现场控制设备，并监控原始网络数据或控制设备。

8. **外部入侵检测**：监管系统可以监控网络主机是否激活远程连接服务或添加未授权进程或端口，以判断系统是否遭受外部网络入侵。

9. **网络层监控**：监管系统能够获取工业系统的数据流量，检测来自未知来源的OPC读命令，判断监控网络层是否遭受入侵。

10. **数据一致性检测**：对于敏感的“写”命令，监管系统可以采用一致性检测方法，比较从设备获取的命令数据与监控软件检测到的值，以判断是否被篡改。

文章强调了网络安全在工业控制系统中的重要性，并提出了一些检测和保护措施来增强系统的安全性和抵御网络攻击的能力。

---

### PLC安全威胁


PLC常见的安全威胁有以下几类：代码安全，固件安全，网络安全，协议安全，HMI安全等等。

协议安全相关：

1. **缺乏认证**：Modbus协议没有包含身份认证机制，这意味着攻击者可以伪造合法会话来控制现场设备。

2. **缺乏授权**：协议中没有访问控制的界限，导致任何用户都能够发起任何代码执行，这为攻击者提供了便利。

3. **缺乏加密**：Modbus协议不采用加密保护数据，使得攻击者可以容易地窃听通信并获取敏感信息。

文章还提到了一些针对Modbus协议安全性的研究和增强措施，包括：

- **在网关上添加安全机制**：通过在通信路径上的网关增加安全措施来提高Modbus协议的安全性。
- **在终端设备上添加安全机制**：直接在PLC等终端设备上增加安全功能，以保护Modbus通信。
- **对协议数据进行加密**：使用加密算法来保护Modbus协议传输的数据，防止数据被窃听。

此外，文章中还提到了一些具体的安全增强措施，例如：

- **使用RSA和SHA-2进行加密**：Aamir等人提出使用AES、RSA和SHA-2等高级加密算法来增强Modbus会话的安全性。
- **设计安全的Modbus架构**：Igor等人设计了一个基于套接字通信的Modbus安全架构，增加了认证和完整性检查。
- **形式化建模和模型检查**：Roberto等人提出了Modbus协议的形式化建模，并使用模型检查方法来定位易受攻击的属性。

这些措施旨在解决Modbus协议的固有安全问题，提高工业控制系统中协议通信的安全性。


---

### PLC安全威胁

人机界面（HMI）和显示终端单元（DTU）的安全问题主要涉及以下几个方面：

1. **远程访问和互联性**：随着HMI和DTU的远程访问和与其他网络及设备的互联，它们变得更易受攻击。这种互联性增加了遭受黑客攻击和威胁的风险。

2. **操作系统漏洞**：HMI和DTU通常基于通用操作系统（如Windows）构建，并运行基于Windows的程序。这意味着它们继承了底层操作系统的所有漏洞，容易受到网络威胁的攻击。

3. **外部恶意软件**：HMI和DTU可能受到通过互联网、公司网络或用户本地部署的恶意软件的攻击。恶意软件可能用于监视、破坏工业系统，延迟或阻断网络，甚至包括PLC rootkit。

4. **欺骗攻击**：攻击者可能通过冒充授权设备的身份来发起攻击，这可能导致远程访问权限的非法获取，并对软件和硬件造成严重损害。

5. **SQL注入**：针对基于Web的HMI和具有数据库应用程序的服务器，攻击者可能利用SQL注入攻击来控制系统或在查询中插入意外的SQL语句，以操纵数据库。

6. **通用软件问题**：由于HMI现在是基于通用软件的产品，它们更容易受到攻击，因为攻击者可能将它们视为网络上的常规PC或另一个易受攻击的操作系统。

7. **安全配置不当**：如果HMI和DTU配置不当，比如没有适当的访问控制，未经授权的用户可能会访问敏感数据或执行恶意操作。

8. **USB威胁**：恶意攻击可能通过感染的USB设备部署，如果HMI或DTU连接了这些设备，就可能受到威胁。

9. **内部威胁**：不满的员工可能对系统造成重大损害，他们可能破坏代码、感染HMI或PC、在梯形逻辑中编写休眠的恶意代码，或向黑客开放某些端口。

10. **缺乏安全文化和意识**：如果组织没有培养出一种安全文化，员工可能不会意识到安全威胁和不当行为，从而增加了安全风险。

为了应对这些安全问题，文件提出了一系列安全建议，包括但不限于：
- 将安全性作为设计、更新或运行任何PLC-BS项目时的首要考虑因素。
- 培养组织文化，提高员工对安全威胁的认识，并提供定期的安全培训。
- 实施基于角色的访问控制，确保只有授权人员才能访问敏感信息和设备。
- 隔离PLC-BS系统网络，避免与其它网络混合。
- 定期备份关键文件，并确保可以快速恢复。
- 对系统进行定期检查和比较，确保运行的文件的完整性。
- 限制远程访问和物联网（IoT）设备的使用，确保所有通信都是经过过滤和检查的。
- 在HMI和相关PC上物理禁用USB端口，只允许经过认证和批准的USB设备使用。
- 禁用任何设备的备用端口。
- 保持系统日志，并定期进行系统审计和渗透测试。
- 持续进行漏洞和威胁评估，并制定预防性解决方案。

这些措施旨在提高HMI和DTU的安全性，减少潜在的安全风险。

---

### PLC安全威胁

PLC安全研究现状：

许多公司和供应商错误地认为梯形逻辑是安全的，不会被黑客和入侵者访问，因为PLC网络通常是隔离的。然而，由于随机威胁、内部威胁和外部威胁，这种假设是不正确的。

感觉这里应该详细说明一下。

Sizzler文章中提出了很多有待解决的代码相关问题。

文章指出，即使程序员遵循公司的标准和建议，结构不良和设计不当的梯形逻辑代码也会增加漏洞和安全风险。

---
### PLC代码安全

<style scoped>
section li {
  font-size: 30px;
  line-height: 1;
}
</style>

PLC本质上是一台简单的计算机，执行主机计算机的指令，处理用户程序，产生输出以控制现场设备的操作，并将现场设备的数据传输给主机计算机。在此执行过程中，存在用户代码漏洞、固件篡改和网络攻击等安全问题。在绘制梯形图的过程中，由于人员的疏忽可能会引入一些潜在的安全漏洞。

+ **常见的代码安全问题**：包括逻辑错误、定时器条件竞争、无条件转移、代码无法到达、无限循环、语法错误、作用域和链接错误、隐藏跳转器、重复对象和未使用对象等问题。

+ **代码安全问题的后果**：例如定时器振荡、某些分支无法访问、循环超时错误、易于受到攻击和改变过程、恶意程序嵌入、程序执行过程改变、输出失败、攻击者利用等。

<!-- Programmable logic controllers based systems PLC-BS vulnerabilities and threats.pdf -->

---

### PLC代码安全

<!-- todo：结合实际例子详细介绍一下 -->

1. **使用重复指令**：在梯形逻辑代码中重复使用某些操作数（如OTE、计数器、定时器和JSR）会导致不希望的结果。

2. **窥探**：编写能够记录某些关键参数和值的梯形逻辑代码，以便在不影响逻辑流程和目的的情况下秘密泄露信息。

3. **缺少特定线圈或输出**：当梯形逻辑中缺少其他标签依赖的特定输出线圈时，会增加漏洞风险。

4. **绕过**：通过在线强制某些操作数的值或使用空分支、跳线来绕过逻辑。

参考文献：PLC Code-Level Vulnerabilities

---

### PLC代码安全

5. **拒绝服务攻击（DoS）**：用户可以编写或上传恶意的梯形逻辑代码到PLC，该代码可能在特定时间激活或触发，导致PLC严重减慢、完全停止或造成重大故障。（这里可以说明一下什么情况下会dos）

6. **使用硬编码值**：在某些情况下，使用硬编码的值或参数可能会危及过程或其相关程序。

7. **竞争条件**：当两个代码或逻辑操作数相互竞争时，可能导致不一致的结果，并可能被用来创建威胁以损坏设备。（举例子说明）

参考文献：PLC Code-Level Vulnerabilities

---

### PLC代码安全


<style scoped>
section li {
  font-size: 32px;
  line-height: 1;
}
</style>

在PLC（可编程逻辑控制器）的上下文中，"race"指的是一种特定的竞争条件，也称为"race condition"。这通常发生在多个程序指令或逻辑分支尝试同时控制同一输出或资源时，导致不确定的结果或意外的行为。

具体来说，在PLC程序（如梯形图）中，可能发生一下情况：

1. 两个或多个指令尝试同时改变同一个输出的状态（例如，一个指令尝试置位（Set），另一个尝试复位（Reset））。

2. PLC程序在每个扫描周期内顺序执行。如果在同一个周期内，对同一输出有相互矛盾的操作，它们可能会在不同的指令执行顺序下产生不同的结果。

3. 在输入状态保持不变的情况下，由于程序逻辑中的竞争条件，输出状态在连续的扫描周期内不断变化。

---

### PLC代码安全

竞争条件可能导致的问题包括但不限于：

- **系统不稳定**：输出设备可能会在开启和关闭之间不断切换，导致机械部件的不必要磨损或过程控制的不稳定。
- **安全风险**：在安全关键的应用中，竞争条件可能导致安全措施的失败。
- **数据不一致**：在数据采集和处理过程中，可能导致数据的不一致性和错误。

因此，在设计PLC程序时，需要特别注意避免竞争条件，以确保系统的可靠性和安全性。

---

### PLC代码安全

8. **缺乏详尽的诊断和报警消息**：如果没有详细和深入的报警、诊断或前提条件，设备可能会面临巨大风险。

9. **编译器警告**：忽视某些PLC编译器警告可能是严重的，因为它们可能是真正的威胁。

10. **未使用的标签或操作数**：未使用的标签可能被休眠的恶意代码或外部攻击利用。

参考文献：PLC Code-Level Vulnerabilities



---
### PLC代码安全分析方法：研究现状

当前的主流方法： **静态分析方法**

<!-- Programmable logic controllers based systems (PLC‐BS): vulnerabilities and threats -->

<!-- SoK- Attacks on Industrial Control Logic and Formal Verification-Based Defenses -->

**代码安全的形式化分析方法** 该方法将PLC梯形图或指令列表语言等转换成中间语言，以便于模型构建。然后考虑环境和其他因素构建模型，并通过符号执行来解决状态空间爆炸问题，并最终检查各种代码缺陷。

包括中间语言转换、模型检测、符号执行等步骤。

<!-- 这边有一个plc静态分析的综述可以介绍一下 -->

代码安全形式化分析方法：PLC代码安全形式化分析是为了检测PLC中的安全问题。模型检测是当前最广泛使用的PLC形式化分析方法。它将PLC梯形图或指令列表语言等转换成中间语言，方便模型构建，然后构建模型并使用符号执行来检测各种代码缺陷。

符号执行的局限性：符号执行虽然可以解决状态空间爆炸问题，但会消耗大量时间和内存，不适合检测过大的程序。同时，由于它使用静态分析方法，无法检测分配给PLC的虚假传感器值。

中间语言转换：为了更好地解决中间语言转换问题，提出了基于NuSMV的中间模型方法和基于Vine的方法。

模型检测：可以构建适当的模型来自动生成检查系统，使用的工具包括SMV、UPPALL、SPIN、Verilog、TSV等。

---

### PLC代码安全分析方法：研究现状

**静态分析的局限性**：
1. 使用符号执行虽然可以解决状态空间爆炸问题，但会消耗大量时间和内存，因此对于大型程序的检测时间较长。

2. 在单纯分析梯形图时可以分析逻辑错误。但是当把IO一起考虑在内时，就比较难处理，因为代码的实际执行逻辑和IO关系非常大，例如无法检测分配给PLC的虚假传感器值等。

这种时候就需要动态分析

参考文献：SoK- Attacks on Industrial Control Logic and Formal Verification-Based Defenses

---

### PLC代码安全分析方法：研究现状

2. **动态分析方法**

可运用Fuzzing等测试手段。Fuzzing是一种自动化的软件测试技术，通过自动生成大量异常、随机或边界测试数据并输入到程序中，观察程序是否能够妥善处理这些输入，以此来发现潜在的错误或安全漏洞。

<!-- 动态分析的适用性  -->

---
### PLC代码安全分析方法：研究现状

2. **动态分析方法** 研究现状

对梯形图进行分析

对binary进行分析

对runtime进行分析

SoK- Attacks on Industrial Control Logic and Formal Verification-Based Defenses

---

### PLC代码安全分析方法：研究现状

2. **动态分析方法**

<style scoped>
section li {
  font-size: 30px;
  line-height: 1;
}
</style>

**对梯形图**的安全分析方法，例如：**Sizzler**
1. **梯形图转换**：Sizzler首先将PLC的梯形图（Ladder Diagrams, LD）转换成ANSI C代码，以便在不同的微控制器单元（MCUs）上执行和测试。

2. **仿真环境搭建**：利用QEMU和Avatar2等工具，Sizzler创建了一个仿真环境，模拟PLC的固件行为，包括GPIO和I2C接口等硬件抽象层。

3. **变异策略**：Sizzler采用了基于SeqGAN（序列生成对抗网络）的变异策略，通过一系列操作符（如位翻转、字节翻转、算术运算和值替换）来修改输入，以触发不同的代码路径，生成有效的测试用例

4. **局限性**：对特定厂商PLC固件的依赖和仿真过程中的性能开销

Sizzler: Sequential Fuzzing in Ladder Diagrams for Vulnerability Detection and Discovery in Programmable Logic Controllers
<!-- todo：运行起来 -->

---

### PLC代码安全分析方法：研究现状

2. **动态分析方法**

**对二进制文件**的安全分析方法，例如：**ICSFuzz**

<style scoped>
section li {
  font-size: 30px;
  line-height: 1;
}
</style>

利用KBUS子系统向控制应用程序提供输入，并通过物理PLC进行测试，这限制了其可扩展性。Runtime。

ICSFuzz: Manipulating IOs and Repurposing Binary Code to Enable Instrumented Fuzzing in ICS Control Applications

局限性：使用物理PLC进行测试，限制了可扩展性

<!-- todo：仔细看看 -->

---

### PLC代码安全分析方法：研究现状

关于PLC racing问题的研究：静态分析

+ 提出了一种将LD转换为普通PNs的方法，而不是像以前研究所做的那样使用扩展PNs。
+ 提出了一种系统性的算法，可以将整个LD转换为普通PN，这使得设计自动转换软件变得更加容易。
+ 提出了一种基于PN的方法来检测、定位和纠正LD中的竞争，这种方法比现有方法更有效，因为它避免了状态空间爆炸问题。

Modeling and Race Detection of Ladder Diagrams via Ordinary Petri Nets


---
## 研究目标

本研究旨在通过Fuzzing技术对PLC（Programmable Logic Controller）的安全性进行分析和检查，尝试进一步挖掘Fuzzing在PLC应用程序安全性分析方面的能力。

---
## 相关工作

<style scoped>
section li {
  font-size: 30px;
  line-height: 1;
}
</style>

**OpenPLC**  OpenPLC是一个开源的软件项目，旨在提供一个成本低廉且实用的替代方案，以取代传统的硬件可编程逻辑控制器（PLC）。

+ 开源性质：作为开源软件，OpenPLC提供了自由访问的源代码，鼓励社区参与和贡献。

+ 兼容性：它支持多种硬件平台，不局限于特定供应商的硬件，提供了更大的灵活性。

+ 符合国际标准：OpenPLC遵循国际电工委员会（IEC）61131-3标准，支持多种编程语言，如梯形图（LD）、功能块图（FBD）、顺序功能图（SFC）、指令列表（IL）和结构化文本（ST）。

+ 应用广泛：它已被应用于多种场景，包括家庭自动化、交通控制、水处理、现场网络以及暖通空调（HVAC）系统的控制等。

<!-- todo这里要介绍详细一些 -->

---
## 相关工作

<style scoped>
section li {
  font-size: 30px;
  line-height: 1;
}
</style>

**LibAFL**：一个轻量级的AFL（American Fuzzy Lop）库，用于Fuzzing。

+ LibAFL是一个新颖且完全可扩展的模糊测试框架，它允许研究人员和开发人员构建自定义的模糊测试工具。LibAFL的设计基于模块化原则，提供了高度的灵活性，允许用户轻松地扩展核心模糊测试流程并共享新组件。该框架用Rust语言编写，利用了Rust的性能和安全性特性。

+ LibAFL的核心库提供了基础的模糊测试构件，包括输入处理、语料库管理、调度器、变异器、执行器和观察器等。它还包括了与不同执行引擎和仪器化后端的集成，如LLVM、QEMU用户模式和Frida。


---
## 整体设计

step1. 如何fuzzing梯形图代码？—— LD->C + LibAFL
step2. 结合外设输入进行一些fuzzing

---
## 具体实现

step1. 从梯形图到C代码：使用OpenPLC

使用OpenPLC可以将梯形图代码翻译成C代码。

OpenPLC是一个可以在多种平台上运行的PLC模拟器（其中包括Linux）。

OpenPLC用一个用户进程来对openplc的执行逻辑进行模拟，用几个数组来模拟IO，最终可以编译成能在Linux平台上直接运行的二进制文件。

<!-- 目前的情况大概是，找到了openplc翻译之后的c代码以及编译出来的二进制文件。看起来编译二进制文件就是单纯地编译一下。-->

---

## 具体实现

step2. 结合外设输入进行fuzzing

+ PLC编程中存在racing等问题，依靠静态分析无法解决，而且和IO关系较大
+ 探索结合外设输入的PLC代码fuzzing方法，转发真实外设请求或模拟外设输入


<!-- ---
## 具体实现

step2. 使用QEMU对硬件进行模拟

<style scoped>
section li {
  font-size: 30px;
  line-height: 1;
}
</style>

非通用架构软件fuzzing时的几种常用解决方案及其对比：

1) On-device Analysis：这种方法需要真实设备来进行测试。它得到了最真实可信的结果，但可扩展性较差，且缺乏可见性。因为在裸机上收集执行信息较为困难。
2) Full Emulation：为了克服真实硬件上的性能和可扩展性问题，研究人员提出了使用如QEMU这样的完整仿真器来模拟固件执行。主要面临的挑战是对外设的方针难以模拟。
3) Peripheral Forwarding：将外设访问转发到真实设备，并将固件在仿真器内运行。由于依赖真实硬件，性能和可扩展性问题仍未解决。
4) Semi-rehosting：固件的主要逻辑仍然在仿真器内执行，外设相关的处理被识别出来，在主机上运行。 -->


<!-- ---
## 具体实现

step3. 探索处理外设输入的方法，包括转发真实外设请求或模拟外设输入

在现有的研究中：采用了外设映射的方法。当遇到fuzzing框架没有处理过的外设时，就会crash，影响对程序结构的进一步探究

考虑采用模拟外设输入的方式完成 -->


---
## 预期成果

- 开发一个基于覆盖率反馈的PLC Fuzzing系统
- 发表一篇高水平会议论文
- 提供PLC安全漏洞样本

---
## 研究计划

1. **2024.8** **系统设计**：设计Fuzzing系统架构，包括代码编译、模拟环境设置和输入处理
3. **2024.11** **系统实现**：开发Fuzzing系统，集成QEMU模拟和外设输入处理
4. **2024.12** **测试与评估**：对系统进行测试，评估其发现安全漏洞的能力
5. **2025.3** **论文撰写**：撰写毕业论文

---
## 参考文献

<style scoped>
section li {
  font-size: 28px;
  line-height: 0.9;
}
</style>

1. A Systemic Review of Kernel Fuzzing

2. Armor PLC: A Platform for Cyber Security Threats Assessments for PLCs

3. From Library Portability to Para-rehosting: Natively Executing Microcontroller Software on Commodity Hardware

4. Fuzzing of Embedded Systems: A Survey

5. Fuzzing the Internet of Things: A Review on the Techniques and Challenges for Efficient Vulnerability Discovery in Embedded Systems

6. Fuzzing: a survey

7. Fuzzware: Using Precise MMIO Modeling for Effective Firmware Fuzzing

8. Hydra: Finding Bugs in File Systems with an Extensible Fuzzing Framework

9. ICS3Fuzzer: A Framework for Discovering Protocol Implementation Bugs in ICS Supervisory Software by Fuzzing

10. ICSFuzz: Manipulating IOs and Repurposing Binary Code to Enable Instrumented Fuzzing in ICS Control Applications

---
## 参考文献

<style scoped>
section li {
  font-size: 28px;
  line-height: 0.9;
}
</style>

11. IEC 61850 Compatible OpenPLC for Cyber Attack Case Studies on Smart Substation Systems

12. Investigating the Security of OpenPLC: Vulnerabilities, Attacks, and Mitigation Solutions

13. kAFL

14. KernelGPT: Enhanced Kernel Fuzzing via Large Language Models

15. LibAFL QEMU: A Library for Fuzzing-oriented Emulation

16. LibAFL: A Framework to Build Modular and Reusable Fuzzers

17. OpenPLC: An IEC 61,131–3 compliant open source industrial controller for cyber security research

18. OpenPLC: An Open Source Alternative to Automation

19. Review of PLC Security Issues in Industrial Control System

20. Security Challenges in Industry 4.0 PLC Systems

---
## 参考文献


<style scoped>
section li {
  font-size: 28px;
  line-height: 0.9;
}
</style>

21. SHiFT: Semi-hosted Fuzz Testing for Embedded Applications

22. Sizzler: Sequential Fuzzing in Ladder Diagrams for Vulnerability Detection and Discovery in Programmable Logic Controllers

23. SoK: Enabling Security Analyses of Embedded Systems via Rehosting

24. Tardis: Coverage-Guided Embedded Operating System Fuzzing

25. What You Corrupt Is Not What You Crash: Challenges in Fuzzing Embedded Devices

26. 𝜇AFL: Non-intrusive Feedback-driven Fuzzing for Microcontroller Firmware

27. PLC Code-Level Vulnerabilities

28. Programmable logic controllers based systems (PLC‐BS): vulnerabilities and threats

29. SoK- Attacks on Industrial Control Logic and Formal Verification-Based Defenses

30. SoK: Security of Programmable Logic Controllers
