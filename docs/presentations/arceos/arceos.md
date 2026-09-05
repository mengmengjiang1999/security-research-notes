---
theme: gaia
_class: lead
paginate: true
backgroundColor: #fff
_backgroundImage: url('./figures/hero-background.svg')
marp: true
style: |
    section {
        font-size: 2.1em;
    }
    .columns {
        display: grid;
        grid-template-columns: repeat(2, minmax(0, 1fr));
        gap: 1rem;
    }
    .right-label {
        right: 230px;
        position: absolute;
        display: inline-block;
    }
---

# **ArceOS: 组件化操作系统的初步探索**

##### ArceOS: An Exploration of Component-Based Operating System

贾越凯
清华大学 计算机系

2023/3/26
OS2ATC 2022

---

<style scoped>
section img {
    position: absolute;
    top: 22%;
    left: 57%;
    width: 40%;
}
</style>

## 背景

+ 目前计算机软硬件的发展趋势：
  + 硬件：新型硬件层出不穷
  + 应用：对性能、安全的需求越来越高
  + 操作系统：发展滞后
    + 通用 OS：Linux，Windows
    + 实时 OS：RT-Thread、FreeRTOS

#### <center style="color:var(--color-highlight)">*现有的操作系统已经难以满足硬件和应用的发展需求*</center>

> 旧仓库未保存此页的架构示意图。

<!-- 首先来介绍一下相关背景。随着计算机软硬件技术的发展，新型硬件层出不穷，应用更是五花八门，而且应用对性能和安全的需求也越来越高。 对于 操作系统来说，发展就相对滞后，目前用的大多还是以 Linux 为代表通用操作系统，或像是 RT-Thread 这 样的实时操作系统。 这就形成了一个两头宽，中间窄的格局。 因此我们认为现有的操作系统已经难以满足 硬件和应用的发展需求。 -->

---

## 背景

<style scoped>
strong {
  color:var(--color-highlight);
}
.fig-group {
    position: absolute;
    font-size: 18px;
}
.fig-group img {
    width: 100%;
    margin-bottom: 10px;
}
</style>

+ 目前已有的 OS 存在的问题：
  + 面向通用场景，**臃肿而笨重**
  + 组件之间紧耦合，**难以重用**
  + **开发复杂**，门槛高

<div class="fig-group" style="left: 57%;top: 35%;width: 35%;">

> 旧仓库未保存此页的 Linux 漏洞统计图。

<center>Linux 每年引入的安全漏洞数量：https://www.cvedetails.com/product/47/Linux-Linux-Kernel.html
</center>

</div>

<div class="fig-group" style="left: 15%; top: 53%;width: 31%;">

> 旧仓库未保存此页的 Linux 组件依赖图。

<center>Linux 内核组件依赖关系图 (来源：Unikraft)</center>

</div>

<!-- 目前已有的 OS，主要存在以下几方面的问题：
1. 它们大多是面向通用场景，考虑了各种应用与硬件，这使得它们都巨大无比，显得十分臃肿与笨重。由于代码量大，再加上使用不安全的 C 语言，也因此存在许多潜在的安全漏洞和 bug。此外，丰富的功能也使得通用 OS 内核镜像大，启动速度慢，资源消耗大，即使应用只需要用到其中很少一部分的功能，并不适合目前的嵌入式、serverless 等场景。
2. 像 Linux 这样的 OS 虽然也采用了模块化的设计思路，但模块之间的数据访问和函数调用都是一种紧耦合的方式，很难将其中的几部分抠出来，用到其他 OS 或系统软件上。另一方面，这些 OS 也难以复用其他成熟软件中的模块。
3. 结合以上两点，庞大的代码量与紧耦合的模块依赖，使得这些 OS 的开发非常复杂，门槛极高，而且往往是牵一发动全身。
-->

---

## 组件化 OS 的设计目标

* **易于定制**
  + 适应硬件与应用需求
  + 高安全、高性能、专用化
* **易于复用**
  + Rust crate
  + Rust for Linux
* **易于开发**
  + 层次化组件隐藏底层细节
  + 像开发应用一样开发 OS

<!--

因此我们认为，庞大的通用 OS 不再是未来 OS 的主要发展方向，小型的、专用的 OS 更适合今后新型领域的应用场景。所以我们希望开发一种组件化的 OS，从一系列组件中，迅速组合出一个 OS 来，以适配硬件的特点，满足应用的需求。

我们希望这个组件化 OS 具有以下特点：

1. 易于定制。我们希望能够根据应用场景，对 OS 的功能进行深度定制，比如 OS 无需包含应用没有用到的功能，比如可以专门引入针对应用或硬件进行了优化的模块，从而提升 OS 的性能与安全性。
2. 我们希望组件之间可以更好地重用。这一点也是 Rust 语言相比 C 语言的优势之一，Rust 通过 crate 来组织各个模块，远比 C 语言要简洁易用，而且具有包管理器与丰富的生态。另一方面，Linux 内核也在推动 Rust 的使用，我们希望一个用 Rust 语言开发的组件或 crate，既可以用到我们的组件化 OS 中，也可以用到 Linux 中。
3.最后，我们希望通过层次化的组件划分，隐藏 OS 繁琐的底层细节，实现像开发应用一样开发 OS，人人都能来开发 OS 的终极目标。

总的来说，我们就是希望打造一系列 OS 的轮子，便于人们根据应用需求，将这些轮子拼出一个定制的 OS 来。
-->

---

## ArceOS: 组件化 OS 的初步探索

<style scoped>
blockquote {
    /* border-top: 0.1em dashed #555; */
    font-size: 20px;
    margin-top: auto;
}
</style>

+ https://github.com/rcore-os/arceos
+ Ar-Qs, Ar-key-O-S
+ 已实现的功能：
  + 多架构：aarch64/riscv64
  + 抢占式、协作式多处理器调度
  + VirtIO net/blk/gpu 驱动
  + 基于 [smoltcp](https://github.com/smoltcp-rs/smoltcp) 的网络栈
  + Rust / C 语言应用程序

> 名称来源：Arceus，一款游戏中创造了万物的虚构角色，可以根据携带的石板切换不同的形态和属性

<!--
我们目前已经对组件化 OS 这一方向进行了一些初步探索，开源了 ArceOS 这一原型系统。

目前已经通过一系列组件，实现了一些基本的 OS 功能，如支持抢占式，或协作式的多线程多核调度，virtio 网络、存储、gpu 驱动，基于 smoltcp 的网络协议栈，并能支持用 Rust 或 C 语言编写的简单应用程序。
-->

---

## ArceOS 整体架构

<style scoped>
section img {
  position: absolute;
  margin-left: 42%;
  top: 3%;
  width: 42%;
}
</style>

+ Unikernel
  + 单应用
  + 单地址空间
  + 单特权级
+ 层级化
  + crates
  + modules
  + ulib
  + apps

> 旧仓库未保存此页的 ArceOS 架构图。

<!--
这是 ArceOS 目前的架构图。
首先，目前 ArceOS 采用的是 unikernel 的设计方式，这种 OS 将内核与应用链接在一起，只支持一个应用。而且假设运行在一个 hypervisor 之上，已经由 hypervisor 做了隔离，就不再需要 OS 自己做了，所以它是单地址空间单特权级的。这样一来所有系统调用都可以是函数调用，从而具有比传统的 OS 更高的性能。

然后，ArceOS 的组件，自底向上，可分为 4 个层次。最底层的组件被称为 crates，是一些与 OS 的设计无关的公共组件，可以轻易地被其他 OS 复用，比如像 linked_list 这样的基本数据结构，还有 buddy system，Round robin 这样的基础算法。这些组件都可以被发布到 crates.io 中，方便其他 rust 项目使用。

其上一层的组件被称为 modules，是一些与 OS 的具体设计比较相关的组件，换了一个与 arceOS 设计完全不同的 OS，这个模块不一定能直接被复用了，比如 axtask 会利用 crates 中的调度算法，实现一套 ArceOS 特有的线程管理与调度器。此外也有一些 modules 只是对 crates 中组件的一些简单封装，如 axalloc 就封装了具体的内存分配算法，并配置了 rust 全局堆分配。这些模块都有 ax 前缀，表示与 ArceOS 的设计绑定。

modules 和 crates 合起来可认为是传统意义上的内核，在 modules 之上就是用户程序和用户库。我们在 ulib 层，实现了对 Rust 和 C 两种语言编写应用程序的支持。我们提供了 libax 库，是对底层 modules 功能的封装，用于为 Rust 应用程序提供一套类似 rust std 标准库的接口。此外还有一个 c_libax 库，是一个简化版的 libc，为 c 应用提供支持。这些库都根据 unikernel 的特点进行了接口上的优化，与内核之间不再使用 posix 接口，大大缩短了调用路径。不过，为了兼容大量已有应用程序，我们还计划支持 musl libc 和 rust std，并使用传统的 linux 系统调用接口。

-->

---

## ArceOS 组件设计

<style scoped>
section li {
  font-size: 30px;
}
</style>

<div class="columns">
<div style="width:100%">

### crates

+ 与 OS 的设计无关的公共组件
  + linked_list
  + page_table
  + allocator
  + scheduler
  + drivers
  + spinlock
  + ...

</div>
<div style="width:100%">

### modules

+ 与 OS 的设计比较耦合的组件
  + axruntime
  + axtask
  + axnet
  + axsync
+ 对 crates 组件的选取与封装:
  + axalloc
  + axdriver
  + axdisplay

</div>
</div>

<!--
因此，我们要实现的组件集中在 crates 和 modules 两部分。其中 crates 中的组件比 modules 中的组件更加灵活，因为 crates 中的模块在编写时考虑到了会被与 ArceOS 不同设计的 OS 所使用，但是也因此需要更多设计上的考虑。所以，我们的目标是尽量将 OS 的功能拆分到 crates 组件中，逐步增加 crates 组件的数量，减少 modules 组件的数量。
-->

---

## ArceOS 应用程序组件依赖

<style scoped>
section li {
    font-size: 29px;
}
</style>

+ 必选组件
  + axruntime：启动、初始化、组件总体管控
  + axhal：硬件抽象层，提供跨平台的统一的 API
  + axlog：打印日志
  + axconfig：平台相关常量与内核参数定义
+ 可选组件
  + axalloc：动态内存分配
  + axtask：多任务 (线程)
  + axdriver：设备驱动 (网络、存储、图形)
  + axnet：网络栈
  + axdisplay：图形显示

<!--
那么，如何通过这些组件，组合出一个针对应用需求定制的 OS 呢？
首先，有一些组件是必须的，不能再少了，比如 axruntime 中实现了硬盘平台的启动与初始化，以及组件间的一个总体管控。
axhal 是硬件抽象层，提供跨平台的统一的 API。
axlog 用于打印日志，axconfig 定义各种平台相关常量与内核参数。
另一些组件是可选的，应用不需要时就可以不包含进内核中。比如当应用需要动态内存分配时，就引入 axalloc 模块。应用需要多线程时，就引入 axtask 模块。其他类似的还有 驱动、网络、图形显示等等。
-->

---

## ArceOS 代码统计

<style scoped>
section table {
    font-size: 28px;
    width: 100%;
    margin: 0 auto;
    margin-top: 10px
}
</style>

| crates (23)      | 行数   | modules (11) | 行数   |
| ---------------- | ---- | ------------ | ---- |
| linked_list      | 452  | axhal        | 1885 |
| slab_allocator   | 437  | axtask       | 855  |
| page_table       | 397  | axnet        | 637  |
| page_table_entry | 384  | axruntime    | 280  |
| percpu_macros    | 324  | axalloc      | 206  |
| spinlock         | 313  | axsync       | 196  |
| memory_addr      | 278  | axlog        | 177  |
| driver_virtio    | 272  | axdriver     | 172  |
| ...              |      | ...          |      |
| 合计               | 4715 | 合计           | 4896 |

<!--
目前已经有 23 个 crate，11 个 modules，共 30 多个组件。
可以发现，每个组件的代码量都比较少，大多只有几百行。这个 axhal 比较多的原因是支持多个体系结构。

-->

---

## 演示 1：ArceOS 应用程序

```Rust
// apps/helloworld/src/main.rs
#![no_std]
#![no_main]

#[no_mangle]
fn main() {
    libax::println!("Hello, world!");
}
```

```toml
# apps/helloworld/Cargo.toml
[package]
name = "arceos-helloworld"

[dependencies]
libax = { path = "../../ulib/libax" }
```

<!--
下面来演示一下如何写一个应用程序在 arceps 上运行。
要编写一个最简单的 hello world，与在本机使用 rust 标准库编写基本相同，只要将 libax 作为依赖引入，并将 std 库换成 libax。
-->

---

## 演示 1：ArceOS 应用程序

+ 根据应用需求进行功能定制：

  ```toml
  [dependencies]
  libax = {
      path = "../../ulib/libax",
      features = [
          "alloc",        # 动态内存分配
          "paging",       # 页面映射
          "multitask",    # 多线程
          "smp",          # 多核
          "net",          # 网络
          "fs",           # 文件系统
          "display",      # 图形显示
      ]
  }
  ```

<!--
刚才的 hello world 只启用了 arceOS 最基本的功能。我们可以在 features 中，根据应用的需求，指定更多功能，如动态内存分配，多线程、网络等等，就可以在 libax 中启用相应的函数或结构。

-->

---

## 演示 1：ArceOS 应用程序

+ 同类模块，不同实现：

  ```toml
  [dependencies]  # apps/task/parallel/Cargo.toml
  libax = {
      path = "../../ulib/libax",
      default-features = false,
      features = [
          "alloc", "paging",  "multitask",
          "sched_fifo", # can be "sched_rr",
      ]
  }
  ```

+ 更多支持不同实现的模块：

  + 内存分配算法：buddy system/slab
  + 网络协议栈：smoltcp/lwip
  + 文件系统：fat32/ext

<style scoped>
section li {
    font-size: 29px;
}
</style>

<!--
另一个例子是针对同类模块，能够选择不同的实现。比如我们可以在开启了多线程功能后，同时在指定调度算法，sched_fifo 是基于 FIFO 的非抢占式协作调度，sched_rr 是基于 round robin 的抢占式调度。

对于其他模块，也能支持类似的不同实现之间的切换。如可以切换内存分配算法，切换网络协议栈，切换文件系统等等。
-->

---

## 演示 1：ArceOS 应用程序

<style scoped>
section table {
    font-size: 28px;
    width: 100%;
    margin: 0 auto;
    margin-top: 30px;
}
section table tr > th:nth-child(1) {
  width: 10%;
}
section table tr > th:nth-child(2) {
  width: 28%;
}
section table tr > th:nth-child(3) {
  width: 23%;
}
section table tr > th:nth-child(4) {
  width: 8%;
}
section table tr > th:nth-child(5) {
  width: auto;
}
</style>

| 应用         | 启用的功能                                | 额外依赖的模块                          | 镜像大小 | 描述                 |
| ---------- | ------------------------------------ | -------------------------------- | ---- | ------------------ |
| helloworld |                                      |                                  | 40K  | 打印 "Hello, world!" |
| memtest    | alloc, paging                        | axalloc                          | 65K  | 内存分配测试             |
| display    | alloc, paging, display               | axalloc, axdisplay               | 73K  | 图形显示测试             |
| parallel   | alloc, paging, multitask, sched_fifo | axalloc, axtask                  | 69K  | 并行计算测试             |
| httpclient | alloc, paging, net                   | axalloc, axdriver, axnet         | 114K | 发送一个 HTTP 请求       |
| httpserver | alloc, paging, net, multitask        | axalloc, axdriver, axnet, axtask | 131K | 多线程 HTTP 服务器       |

---

## 演示 2：crate_interface

* 使用场景：
  1. crates 组件需要调用 modules 组件中的实现
  2. 组件间存在循环依赖
* 示例：
  + crates/kernel_guard 需要关闭内核抢占
  + 具体实现位于 modules/axtask
* 解决思路：
  + 通过 Rust FFI 进行调用
  + 提供类似 `#[global_allocator]` 的接口

<!--
第二个示例是介绍 arecOS 中一个重要的基础组件 crate_interface。

我们在编写组件时，可能会遇到这样一些问题：一个 crates 中的组件需要依赖 modules 中组件的服务，但是在 arceos 的设计中，crates 是最底层的一层组件，不能去调用上层 mouldes 的组件。或者是另外一种情况，组件之间存在循环依赖，但是这本来在 rust 中是不允许的。

这里举一个实际的例子，比如我们在 crates 中实现了一个组件，它需要关闭内核抢占。而实际关抢占的操作，是在 modules 的的一个组件中实现的。如果使用通常的写法，这个组件也需要位于 modules 中，这与我们将组件尽量拆解到 crates 层的设计理念不符。

但是这在 C 语言，这不是问题，可以直接随意调用，但同时也是一个大问题，C 语言编写的系统的复杂性，紧耦合性就是从混乱的调用关系开始的。

那么如何解决这一问题呢？我们受到了 Rust 全堆分配器的启发，可以在代码的任何一处实现一个 global allocator，并为它加上一个这个宏，就能在代码的任意一处使用堆分配。

-->

---

## 演示 2：crate_interface 使用

```rust
// crates/kernel_guard/src/lib.rs
#[crate_interface::def_interface] // define
pub trait KernelGuardIf {
    fn disable_preempt();
    fn enable_preempt();
}
crate_interface::call_interface!(KernelGuardIf::disable_preempt); // call
```

```rust
// modules/axtask/src/lib.rs
struct KernelGuardIfImpl;
#[crate_interface::impl_interface]
impl kernel_guard::KernelGuardIf for KernelGuardIfImpl {
    fn disable_preempt() { do_disable_preempt(); } // implementation
    fn enable_preempt() { do_enable_preempt(); }
}
```

这是在代码中的一个使用实例。我们在那个需要关闭内核抢占的组件中，先定义一个 trait，里面有关抢占，开抢占的方法，并为这个 trait 加上 def_interface 属性宏。在调用时，使用 call_interface 这个宏进行调用。

在 modules 里对抢占进行具体实现时，首先定义一个空结构体，为它实现刚才定义的 trait，并加上 impl_iterface 宏，就可以了。

---

## 演示 2：crate_interface 宏展开

```rust
// crates/kernel_guard/src/lib.rs
extern "Rust" {
    fn __KernelGuardIf_disable_preempt();
    fn __KernelGuardIf_enable_preempt();
}
unsafe { __KernelGuardIf_disable_preempt() }; // call
```

```rust
// modules/axtask/src/lib.rs
impl kernel_guard::KernelGuardIf for KernelGuardIfImpl {
    fn disable_preempt() {
        #[export_name = "__KernelGuardIf_disable_preempt"]
        fn __KernelGuardIf_disable_preempt() { KernelGuardIfImpl::disable_preempt() }
        do_disable_preempt(); // implementation
    } // fn enable_preempt() {}
}
```

<!--
它的基本原理是用利用 rust FFI 进行跨 crate 的调用，并用 rust 过程宏封装为一个简洁的接口。

比如上一页定义的宏，展开后就会变成这样。原来 call_interface 会变成直接读调用 FFI 函数。

原来对 crate 的实现，会导出为 FFI。
-->

---

## 演示 3：用户态单元测试

+ 可以只运行要测试的组件，无需运行完整的 OS
+ 在本机用户态执行，方便调试
+ 示例：axtask 单元测试

> 旧仓库未保存此页的单元测试示意图。

<style scoped>
section img {
    display: block;
    margin: 0 auto;
    width: 46%;
    margin-top: -20px;
}
</style>

<!-- 组件化的另一好处是可以方便的进行单元测试。原来我们写 OS 时，各个组件耦合在一起，要运行相关代码只能将 OS 从头启动一遍。

现在，我们可以为每个组件单独编写单元测试，并使用 cargo test，在用户态进行测试。

也就是说，一个组件既可以运行在 QEMU 内核态，也可以运行在本机用户态。

下面是对 axtask 进行单元测试的演示。
 -->

---

## ArceOS 未来工作

<div class="columns">
<div style="width:100%">

+ 功能完善：
  + 支持真实世界应用：SQLite、Redis、NGINX
  + 兼容 linux 应用，支持 musl gcc、rust std
  + Rust async
+ 多种内核形态配置：
  + Unikernel/用户内核隔离
  + 新型硬件隔离机制：MPK、Memory Tag

</div>
<div style="width:100%">

+ 更丰富的组件选择：
  + 调度：FIFO/CFS
  + 网络：smoltcp/lwip
  + 文件系统：fat32/ext
+ 更灵活的组件使用：
  + 在应用程序层编写组件
  + 与 Rust for Linux 组件的相互复用

</div>
</div>

<!--
最后来展望一下 arceOS 的未来工作与发展方向。

首先是功能完善，我们正在进行对真实世界应用 SQLite、Redis、NIGINX 等的支持，

第二个方向是扩展多种内核形态。作为一个组件化 OS，它应该能够通过组件间的组合与配置，快速构造出宏内核、微内核、unikernel、libos等内核形态。这就需要为目前的 unikernel 架构增加地址空间隔离、特权级隔离等。

我们希望能支持目前的新硬件提供的更灵活的隔离机制，比如 intel MPK，arm memory tag。

我们希望能够直接在 Unikernel 的应用程序层编写组件，而不是全部把它们包含进 arceos 内核中，重蹈 linux 等通用 OS 的覆辙。

我们也希望 能够与 rust-for-linux 的组件相互复用。
-->

---

<!-- _class: lead -->

<!-- _paginate: false -->

<!-- _backgroundImage: url('./figures/hero-background.svg') -->

# Thanks!

<div style="margin: 0 auto;">

+ Repo: https://github.com/rcore-os/arceos
+ Email: equation618@gmail.com
+ Github ID: [equation314](https://github.com/equation314)

</div>
