# 开题报告

## 研究背景和相关工作

LC在工业控制系统中扮演着执行器和传感器之间的桥梁角色，是实现工业自动化和过程控制的关键设备。ICS的整体性能、可靠性和安全性在很大程度上依赖于PLC的性能和安全性。

PLC是可编程逻辑控制器（Programmable Logic Controller）的缩写，它是一种专门用于工业环境中的数字操作电子系统。PLC设计用来控制机械设备或生产过程，通过接收输入信号并根据预设的程序产生输出信号来管理控制逻辑。


### 研究背景

#### 工业控制系统

Industrial Control Systems (ICS) are widely deployed in basic industries closely related to civil life,
such as energy, transportation, water conservancy, electric power, oil and so on. With the rapid development
of the Internet, more and more industrial control systems that used to be physically isolated from the outside
world are now connected to the Internet via universal protocols [1]. However, as the industrial control
systems are initially designed without taking being connected to the Internet into consideration, the
protocols it adopted are nowadays found lacking encryption, authorization, authentication and other
mechanisms. And there are many vulnerabilities in PLC’s firmware and software. The previous methods
used to penetrate traditional networks can now be applied to industrial systems. An increasing number of
cyber-attacks on SCADA and DCS, which adopted these protocols and PLC devices, has become the status
quo. And such cyber-attacks are causing more and more serious issues.
The increasing number of cyber-attacks makes countries and organizations worry more about
security of industrial control systems and invest more into the related research  [2].

#### 工业控制系统和PLC的关系

PLC and its accessories are the core of industrial control system. For a better review of industrial
control systems, the architecture of industrial control system, the equipment and working mechanisms of
PLC, this section mainly introduces the industrial system architecture, its working mode, programming
language and firmware of the PLC.

2.1 The Working Principle of Industrial Control System
A typical industrial control system is composed of field equipment layer, field control layer, control
network and process monitoring layer. The field equipment layer is mainly composed of field actuators,
field sensors, etc.; the field control layer is mainly composed of PLC and related components; the process
monitoring layer is mainly composed of PLC related configuration software and server.
There is a large number of universal protocols, such as Modbus and profibus, adopted between PLC
and the host computer or between PLC and PLC for data transmission. Due to the lack of protection, these
protocols can be easily exploited by attackers. The system structure is shown in Fig. 1.


#### PLC的具体结构

硬件结构、软件结构、编程语言、固件

2.2.1 The Hardware Composition of PLC

As shown in Fig. 2, PLC is composed of CPU, power supply, I/O modules, memory and
expansion modules [6]. Its’ hardware composition and structure are basically the same as those of
traditional computers.

The CPU is responsible for executing user programs and system diagnostics, etc. The power module
is responsible for powering modules such as CPU, memory, and I/O. The memory is used to store user
programs and system programs. The I/O module is responsible for inputting external input signals and
outputting control signals to control the actuators.

2.2.2 The Working Mechanisms of PLC

The PLC adopts the cyclic scanning working mode. When the PLC is in the STOP state, it only
responds to communication and internal processing calls. In the RUN state, during the phase of input
scanning, the PLC scans the external input successively and stores it in the input image register, then it
executes the user program and outputs the result to the output image register. Finally, during the phase of
output scanning, the contents of output image register are sent to the output latch, and then the latch
produces an output in a certain way, such as a relay, a thyristor or the like, to control the execution module.
The time that a cyclic scan takes is called the scan period [7]. As is shown in Fig. 3, Since the PLC
adopts the sequential execution and cyclic scanning flow, it will refresh the input area and the storage area
every time the program is executed, leading to that attackers have time and means to hack the PLC
device, for example, they may use worms.

2.2.3 PLC Programming Language

PLC programming languages include Ladder Language (LD), Instruction List Language (IL),
Function Block Diagram Language (FBD), and Sequential Function Flow Chart Language (SFC). Please
refer to the official programming reference manual. Fig. 4 is a simple Siemens PLC ladder diagram.

The left side of the PLC is the start line, and the right side is the end line. Ladder Language is
scanned and executed from left to right and top to bottom. Each component symbol has a label indicating
its memory address. In Fig. 4, I0.0, I0.1 represents the external input point, and Q0.0, Q0.1 represents the
output point, and M0.0 represents the intermediate state of Q0.0. When we input high level to I0.0 and
I0.1, Q0.0 outputs a high level, and M0.0 also switches to high level, then Q0.1 outputs high level. The
symbols and syntax of PLC ladder diagrams from different manufacturers are different. Please refer to the
official programming reference manual. Due to the function module language and the programming
methods it adopts, PLC is susceptible to security issues such as logical loopholes.

2.2.4 PLC Firmware

Firmware is the infrastructure of the system [8], serving the fundamental functions, such as
controlling the hardware and other communication equipment. Operating system and software rely on
firmware to implement specific functions. For example, the basic I/O system (BIOS) of a computer is a
firmware. The firmware of the PLC generally supports online update, whereas its integrity check is
generally not strict, which makes the firmware easily to be tampered by hackers.
PLC system generally consists of hardware, firmware, and software. Firmware simultaneously
provides interfaces for hardware and software. As is shown in Fig. 5.

### PLC安全漏洞

The essence of PLC is a simple computer, following the instructions from the host computer,
processing the user program, producing an output to control the operation of the field device, and
transmitting the data of the field device to the host computer. In this execution process, there are security
concerns like user code loopholes, firmware tampering and cyber-attacks. This section introduces the
safety problems which PLC confronts with and also introduces some research on the protection methods.

Race condition competition: occurs when two processes concurrently request the same resource. In the context of LD programs, two logical operands are executed simultaneously and race against each other, resulting in an unexpected output although the input is the same. As shown in Figure 3(b), the output value of ynew is changed from 0 to 1 within two cycles, even though the inputs are fixed.
• Missing certain coils or outputs: A rung missing a specific output coil, such as Output Energise (OTE), latches or sets, unlatches, etc., can lead to a dependency issue where other tag(s) are impacted.
• Infinite loop: The main PLC application execution pro- cess is a continuous cycle. If the LD contains infinite loop, it can consume excessive CPU resources to cause PLC to crash.
• Hard-coded logical comparator: embedded into the application, which can consequently be accessed by attackers. Such hard-coded instructions and values can be obtained from the PLC through reverse engineering and then modified to manipulate the operation of the PLC program. As illustrated in Figure 3(c), itvar can be changed thus the whole application will be affected.
• Missing jumps and links: Jumps and links may not be executed following the control flow. An attacker could identify these memory addresses and utilize the spaces to insert malicious code.
• Hidden jumpers: can utilize the jump mechanism in PLC to skip some elements, shown as Figure 3(a). If the jumper is coded to bypass a single element within a given rung, it is possible that more than one element or even a whole branch will be abandoned.
• Object repeat reference: This can occur when one output may be controlled by different inputs. In LD,some operands, for OTE, timers, and counters, could have different results that depend on the scanning of different rungs with similar logic. From example, the y1 output coil in Figure 3(b) is duplicated within the LD, and will be de-energised depending on which rung is executed, resulting in an undesired output.
• Unused objects: It is possible that some variables remain unused, especially in large-scale PLC programs, which will not be detected by the compiler. Open and pre-instantiated entry points present a potential vulnerability in the system. This vulnerability arises from the ease with which an attacker can insert malicious code into the system.

#### 主流方法

对PLC软件进行动态分析，可运用Fuzz等测试手段。

关于PLC fuzzing的常用的思路：

1. 有一些是编译成C代码然后用AFL做fuzzing的（类似rehosting。难点也是对于外设的识别和处理）

2. 有一些是直接在原始环境做fuzzing（这里的难点就是如何输入，如何输出，如何进行崩溃监测什么的）

3. 对Supervisory software做fuzzing，没有特别直接的关系



### 研究目标

fuzzing PLC梯形图

### 相关工作

#### OpenPLC

介绍一下

#### LibAFL

介绍一下

#### 现有的fuzzing相关的工作

Sizzer

Armor PLC: A Platform for Cyber Security Threats Assessments for PLCs

Security Challenges in Industry 4.0 PLC Systems

什么的。其中只有Sizzer是用模拟方式做的fuzz

ICS3Fuzzer: A Framework for Discovering Protocol Implementation Bugs in ICS Supervisory Software by Fuzzing

这个fuzzing的对象是ICS监控程序

ICSFuzz: Manipulating IOs and Repurposing Binary Code to Enable Instrumented Fuzzing in ICS Control Applications

这个fuzzing的方式不是用模拟的方式，而是直接跑在PLC硬件上

## 研究思路

1. 将PLC梯形图编译到C代码

2. 使用QEMU进行模拟而不是在真实硬件上跑

3. 如何处理外设给出的输入。

这部分计划是参考Sizzer给出的解决办法，其实我没太看明白这到底是把真实外设的请求转发到主机，还是搞了一个假的外设模拟一下输入，还是如何。

但是我想做的事情是


## 预期成果与研究计划

### 预期成果

基于覆盖率反馈的PLC fuzzzing系统
发表 高水平会议论文一篇
PLC安全漏洞样本


### 研究计划

不知道怎么写
