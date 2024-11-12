# OpenSBI Deep Dive


## What is SBI？

SBI stands for RISC-V Supervisor Binary Interface

– System call style calling convention between Supervisor (S-mode OS) and Supervisor
Execution Environment (SEE)

SEE can be:
– A M-mode RUNTIME firmware for OS/Hypervisor running in HS-mode
– A HS-mode Hypervisor for Guest OS running in VS-mode

SBI calls help:
– Reduce duplicate platform code across OSes (Linux, FreeBSD, etc)
– Provide common drivers for an OS which can be shared by multiple platforms
– Provide an interface for direct access to hardware resources (M-mode only resources)

Specifications being drafted by the Unix Platform Specification Working group
– Maintain and evolve the SBI specifications
– Currently, SBI v0.1 in-use and SBI v0.2 in draft stage

## What is OpenSBI ?

OpenSBI is an open-source implementation of the RISC-V Supervisor Binary Interface
(SBI) specifications
– Licensed under the terms of the BSD-2 clause license
– Helps to avoid SBI implementation fragmentation

Aimed at providing RUNTIME services in M-mode
– Typically used in boot stage following ROM/LOADER

Provides support for reference platforms
- Generic simple drivers included for M-mode to operate
  - PLIC, CLINT, UART 8250
- Other platforms can reuse the common code and add needed drivers

## Typical Boot Flow

1. ROM
   1. Runs from On-Chip ROM
   2. Uses On-Chip SRAM
   3. SOC power-up and clock setip

2. LOADER
   1. Runs from On-chip SRAM
   2. DDR initialization
   3. Loads RUNTIME and BOOTLOADER

3. RUNTIME (OpenSBI)
   1. Runs from DDR
   2. SOC security setup
   3. Runtime services as-per specification

4. BOOTLOADER
   1. Runs from DDR
   2. Typically open-source
   3. Filesystem support
   4. Network booting
   5. Boot configuration
   6. Lots of other features

5. OS

一些名词解释：
1. ROM (Read-Only Memory)
ROM是一种非易失性存储器，意味着即使在断电后数据也不会丢失。在启动过程中，从芯片内置的ROM开始执行是最基本也是最初始的步骤。这里的ROM通常包含了一小段非常基础的代码，其主要功能是初始化一些最基本的硬件设置，并加载下一步需要运行的程序。
2. SRAM (Static Random Access Memory)
SRAM是一种静态随机存取存储器，它比DRAM（动态随机存取存储器）快得多但成本也更高。在启动初期阶段，使用片上SRAM作为临时存储空间来执行更复杂的初始化任务，比如设置SOC（System on Chip）的工作状态等。
3. SOC (System on a Chip)
SOC指的是在一个单一芯片上集成了所有或大部分构成计算机或其他数字电子系统的组件。这包括但不限于处理器、内存控制器、各种接口等。SOC的设计使得设备可以更加紧凑且高效地工作。
4. DDR (Double Data Rate)
DDR是一种同步动态随机存取存储器技术，允许数据传输速率加倍而不需要增加时钟频率。在启动流程中，初始化DDR是非常重要的一步，因为后续的操作系统和应用程序都将依赖于快速访问外部RAM的能力来进行正常运作。
5. LOADER
Loader是一个负责加载操作系统内核或其他软件到内存中的程序。在这个上下文中，loader不仅完成了DDR的初始化工作，还将运行时环境(RUNTIME)及引导加载程序(BOOTLOADER)加载进DDR中准备执行。
6. RUNTIME (如OpenSBI)
RUNTIME指的是一系列支持操作系统运行的服务和库函数。提到的OpenSBI是RISC-V架构下的一个开源实现，主要用于提供固件服务接口(Firmware Services Interface, FSI)，帮助建立安全的启动环境以及提供给上层软件调用的各种底层服务。
7. BOOTLOADER
Bootloader是一种特殊的程序，它的作用是在计算机启动时首先被加载并运行，然后负责加载操作系统内核进入内存，并最终把控制权转交给操作系统。许多bootloader都是开源项目，它们提供了丰富的功能，比如文件系统支持、网络启动选项等。




## Important Feature

1. Layered structure to accommodate various use cases
   1. Genric SBI library with platform abstraction
      1. Tyically used with external firmware and bootloader
         1. EDK2 (UEFI Implementation), Secure boot working group
   2. Platform specific library
      1. similar to core library but inluding platform specific drivers
   3. Platform specific reference firmware
      1. Three different reference of RUNTIME firmware
2. Wide range of hardware features supported
   1. RV32 & RV64
   2. Misaligned load/store handling
   3. Missing CSR emulation
   4. Protects firmware using RMP support
3. Well documented using Doxygen


## OpenSBI Platform Specific Support

1. Any SBI implementation requires hardware dependent (platform-specific) methods
   1. Print a character to console
   2. Get an input character from console
   3. Inject an IPI to any given HART subset
   4. Get value of memory-mapped system timer
   5. Start timer event for a given HART
   6. … more to come …

2. OpenSBI platform-specific support is implemented as a set of platform-specific hooks in
the form of a struct sbi_platform data structure instance
 - Hooks are pointers to platform dependent functions

3. Platform independent generic OpenSBI code is linked into a libsbi.a static library

4. For every supported platform, we create a libplatsbi.a static library
   1. – libplatsbi.a = libsbi.a + struct sbi_platform instance

5. **SBI实现需要硬件依赖的方法**：任何SBI实现都需要一些与具体硬件相关的功能，这些功能包括但不限于：
   - 向控制台打印一个字符
   - 从控制台获取一个输入字符
   - 向指定的HART子集注入一个IPI（Inter-Processor Interrupt，处理器间中断）
   - 获取内存映射系统定时器的值
   - 为指定的HART启动定时器事件
   - 还有更多其他的功能待添加

6. **OpenSBI平台特定支持的实现**：这种支持是通过一系列平台特定的钩子（hooks）来实现的，这些钩子被组织在一个名为`sbi_platform`的数据结构实例中。
   - 钩子是指向平台依赖函数的指针。

7. **平台无关的通用OpenSBI代码**：这部分代码被打包进一个名为`libsbi.a`的静态库中。

8. **针对每个支持的平台创建静态库**：对于每一个支持的平台，都会创建一个名为`libplatsbi.a`的静态库。
   - `libplatsbi.a`实际上是由`libsbi.a`和一个`sbi_platform`数据结构实例组成的。

简而言之，这段描述了如何在OpenSBI框架内处理不同硬件平台之间的差异。
通过定义一套标准接口（即`sbi_platform`结构体中的钩子），并让具体的平台提供其实现，这样就可以在保持核心代码一致的同时，适应不同的硬件环境。这种方法提高了代码的可移植性和可维护性。


# ✨ Using OpenSBI As a Firmware

## Reference Firmwares

OpenSBI provides several types of reference firmware, all platform-specific
 - `FW_PAYLOAD`: firmware with the next booting stage as a payload
 - `FW_JUMP`: firmware with static jump address to the next booting stage
 - `FW_DYNAMIC`: firmware with dynamic information on the next booting stage

OpenSBI (Open Source Supervisor Binary Interface) 是一个遵循RISC-V架构的开源实现，旨在提供一个标准接口来引导和运行操作系统。
它提供了几种类型的参考固件（firmware），这些固件都是特定于平台的，并且根据它们如何处理下一个启动阶段而有所不同。下面是这三种类型固件的具体解释：

1. FW_PAYLOAD:
   - 定义: 这种类型的固件直接包含了下一阶段的引导代码作为“有效载荷”(payload)。
   - 工作方式: 当系统使用`FW_PAYLOAD`固件时，OpenSBI不仅初始化硬件环境、设置必要的配置，还会加载并执行这个预设的有效载荷。这种方式非常适合那些需要快速从初始状态过渡到更高层次软件（如轻量级内核或用户空间程序）的情况。
   - 应用场景: 适用于开发板测试、小型嵌入式系统等场合，在这里可能希望直接从OpenSBI跳转至应用层而不经过复杂的引导链路。

2. FW_JUMP:
   - 定义: `FW_JUMP`类型的固件设计为通过固定地址跳转到下一个启动阶段。
   - 工作方式: 在这种模式下，OpenSBI完成其初始化任务后，会将控制权转移到预先设定好的内存地址处。这意味着下一阶段的代码必须已经存在于该指定位置上，通常是通过其他手段（比如通过网络下载或者由另一个引导加载器放置在那里）被加载进来的。
   - 应用场景: 适合于那些已经有明确地址规划的系统，或者是当开发者想要更灵活地控制整个启动过程中的各个步骤时。

3. FW_DYNAMIC:
   - 定义: 此类固件允许在运行时动态获取关于下一个启动阶段的信息。
   - 工作方式: 与前两者不同的是，`FW_DYNAMIC`不会假设任何固定的路径或地址来继续执行。相反，它会在启动过程中查找或接收有关下一步操作所需信息的指示。这种方法更加灵活，可以根据实际运行环境做出调整。
   - 应用场景: 非常适用于复杂多变的环境中，例如云计算平台，在那里你可能不总是知道确切的目标是什么直到最后一刻；或者是对于支持多种不同类型的操作系统或应用程序加载需求的场景。

总结来说，OpenSBI提供的这三种固件选项分别针对不同的使用案例和需求：`FW_PAYLOAD`简化了从低级别向高级别软件过渡的过程；`FW_JUMP`给予了一定程度上的灵活性同时保持了简单性；而`FW_DYNAMIC`则提供了最大程度的适应性和可定制性。选择哪种取决于具体的应用场景以及对启动流程控制的需求。



• SOC Vendors may choose:
– Use one of OpenSBI reference firmwares as their M-mode RUNTIME firmware
– Build M-mode RUNTIME firmware from scratch with OpenSBI as library
– Extend existing M-mode firmwares (U-Boot_M_mode/EDK2) with OpenSBI as library

## 1. FW_PAYLOAD

opensbi firmware with the next booting stage as a payload
1. Any S-mode BOOTLOADERS/OS image as the payload to OpenSBI FW_PAYLOAD
2. Allows overriding device tree blob (i.e. DTB)
3. Very similar to BBL hence fits nicely in existing boot-flow of SiFive Unleashed board

OpenSBI (Open Source Supervisor Binary Interface) 是一个开源的固件，用于 RISC-V 架构的处理器。它实现了 RISC-V 的 SBI（Supervisor Binary Interface）规范，为操作系统提供了与底层硬件交互的接口。OpenSBI 固件可以加载并执行下一个启动阶段的 payload（有效载荷），例如引导加载程序或操作系统镜像。

### 1. 任何 S 模式的引导加载程序/操作系统镜像作为 OpenSBI FW_PAYLOAD

在 RISC-V 架构中，S 模式是指 supervisor 模式，它是操作系统内核运行的特权模式之一。OpenSBI 可以将 S 模式的引导加载程序或操作系统镜像作为其 payload。这意味着你可以将以下类型的镜像作为 payload 传递给 OpenSBI：

- **引导加载程序**：如 U-Boot、Coreboot 等。
- **操作系统内核**：如 Linux 内核。

这些镜像会被加载到内存中，并由 OpenSBI 调用执行。具体来说，OpenSBI 会设置好必要的寄存器和环境，然后跳转到 payload 的入口点，从而开始执行 payload。

### 2. 允许覆盖设备树 Blob (DTB)

设备树（Device Tree）是一种描述硬件配置的数据结构，通常用于嵌入式系统。设备树 Blob (DTB) 是设备树的二进制表示形式。
OpenSBI 支持通过命令行参数或其他方式来覆盖默认的 DTB。这样做的好处是可以在不修改固件的情况下，灵活地调整硬件配置。

例如，你可以通过以下方式指定一个新的 DTB 文件：
```sh
fw_payload_path=<path_to_payload> dtb=<path_to_dtb>
```

### 3. 与 BBL 非常相似，适合现有的 SiFive Unleashed 板的启动流程

BBL (Berkeley Boot Loader) 是另一个流行的 RISC-V 引导加载程序，它主要用于 Berkeley Out-of-Order Machine (BOOM) 处理器。
OpenSBI 在设计上借鉴了 BBL 的一些特性，因此它们在很多方面非常相似。

对于 SiFive Unleashed 板（一种基于 RISC-V 架构的开发板），OpenSBI 可以很好地集成到现有的启动流程中。
SiFive Unleashed 板通常使用 U-Boot 作为引导加载程序，而 OpenSBI 可以作为 U-Boot 之前的一步，负责初始化硬件和设置 SBI 接口。

具体的启动流程可能如下：
1. **ROM 启动代码**：首先从 ROM 中运行初始启动代码。
2. **OpenSBI**：加载并运行 OpenSBI 固件。
3. **OpenSBI 初始化**：OpenSBI 初始化硬件和 SBI 接口。
4. **加载 Payload**：OpenSBI 加载指定的 payload（如 U-Boot 或 Linux 内核）。
5. **执行 Payload**：OpenSBI 将控制权交给 payload，继续启动过程。

这种设计使得 OpenSBI 可以无缝集成到现有的启动流程中，提供额外的功能和灵活性，同时保持与现有系统的兼容性。


## 2. FW_JUMP

OpenSBI firmware with a fixed jump address to the next booting stage
- Next stage booting stage (i.e. BOOTLADER) and FW_JUMP are loaded by the previous booting stage
(i.e. LOADER)
- Very useful for QEMU because we can use pre-compiled FW_JUMP

Down-side:
– Previous booting stage (i.e. LOADER) has to load next booting stage (i.e. BOOTLADER) at a fixed location
– No mechanism to pass parameters from pervious booting stage (i.e. LOADER) to FW_JUMP

这段内容描述了OpenSBI（Open Source Berkeley Interrupts）固件中的一个特定组件，即`FW_JUMP`。下面是对这段内容的详细解释：

 1. **FW_JUMP 概述**
   - `FW_JUMP` 是 OpenSBI 固件的一部分，它提供了一个固定的跳转地址，指向下一个启动阶段。
   - 这个固定的跳转地址使得系统能够从当前的启动阶段平滑过渡到下一个启动阶段。

 2. **加载过程**
   - 在启动过程中，`FW_JUMP` 和下一个启动阶段（例如，`BOOTLOADER`）都是由前一个启动阶段（例如，`LOADER`）加载的。
   - 具体来说，`LOADER` 负责将 `FW_JUMP` 和 `BOOTLOADER` 加载到内存中，并设置好它们的执行环境。

 3. **在 QEMU 中的应用**
   - `FW_JUMP` 对于 QEMU（一种流行的开源模拟器）特别有用，因为它允许使用预先编译好的 `FW_JUMP` 固件。
   - 这样可以简化 QEMU 的配置和启动过程，因为不需要每次都重新编译 `FW_JUMP`。

 4. **缺点**
   - 固定位置加载：前一个启动阶段（例如，`LOADER`）必须将下一个启动阶段（例如，`BOOTLOADER`）加载到一个固定的内存位置。这可能会限制系统的灵活性，因为内存布局需要严格遵循预定的规则。
   - 缺乏参数传递机制：`FW_JUMP` 没有提供从前一个启动阶段（例如，`LOADER`）向 `FW_JUMP` 传递参数的机制。这意味着 `FW_JUMP` 无法获取来自 `LOADER` 的任何动态信息或配置，这可能会限制其功能和适应性。

### 总结

`FW_JUMP` 是 OpenSBI 固件中的一个重要组件，它通过提供一个固定的跳转地址来简化启动过程。尽管它在 QEMU 等环境中非常有用，但它也有一些缺点，如需要将下一个启动阶段加载到固定位置以及缺乏参数传递机制。这些缺点可能会限制其在某些复杂系统中的应用。



## FW_DYNAMIC

1. OpenSBI fireware with dynamic information about the next booting stage
   1. The next stage booting stage (i.e. BOOTLOADER) and FW_DYNAMIC are loader by the previous booting stage (i.e. LOADER)
   2. The previous booting stage (i.e. LOADER) passes the location of struct `fw_dynamic_info` to `FW_DYNAMIC` via `a2` register

2. Down-side
   1. – Previous booting stage (i.e. LOADER) needs to be aware of struct `fw_dynamic_info`

~~~c
usigned long magic
usigned long version
usigned long next_addr
usigned long next_node
usigned long options
~~~

`FW_DYNAMIC` 是 OpenSBI（Open Source Berkeley Interrupts）固件中的一个重要特性，它允许前一阶段的引导加载程序（例如 LOADER）向后一阶段的引导加载程序（例如 BOOTLOADER）传递动态信息。这种机制在嵌入式系统和RISC-V架构中特别有用，因为它提供了一种灵活的方式来传递关于下一个启动阶段的重要信息。

### 详细解释

1. OpenSBI固件与动态信息
   - `FW_DYNAMIC` 功能使得前一个启动阶段（例如 LOADER）能够向后一个启动阶段（例如 BOOTLOADER）传递有关下一启动阶段的动态信息。
   - 这些动态信息通过一个结构体 `fw_dynamic_info` 来表示，并且这个结构体的位置是通过 `a2` 寄存器从 LOADER 传递给 FW_DYNAMIC 的。

2. 结构体 `fw_dynamic_info`
   - 这个结构体包含了几个关键字段：
     ```c
     struct fw_dynamic_info {
         unsigned long magic;      // 魔数，用于验证结构体的有效性
         unsigned long version;    // 结构体版本号
         unsigned long next_addr;  // 下一启动阶段的地址
         unsigned long next_node;  // 下一启动阶段的节点信息
         unsigned long options;    // 其他选项或标志
     };
     ```
   - `magic`：通常是一个预定义的常量，用于验证结构体是否有效。
   - `version`：表示结构体的版本号，确保兼容性。
   - `next_addr`：指向下一个启动阶段（例如 BOOTLOADER）的地址。
   - `next_node`：可能包含关于下一个启动阶段的节点信息，具体取决于实现。
   - `options`：其他选项或标志，可以用于传递额外的信息。

3. 传递过程
   - 前一个启动阶段（LOADER）负责填充 `fw_dynamic_info` 结构体，并将其地址存储在 `a2` 寄存器中。
   - 当控制权转移到 `FW_DYNAMIC` 时，它会检查 `a2` 寄存器中的值，并使用该地址访问 `fw_dynamic_info` 结构体。
   - `FW_DYNAMIC` 会根据 `fw_dynamic_info` 中的信息来决定如何继续启动过程，例如跳转到 `next_addr` 指定的地址。

4. 缺点
   - 前一个启动阶段（LOADER）需要知道 `fw_dynamic_info` 结构体的定义和布局，这增加了耦合性。
   - 如果 `fw_dynamic_info` 结构体的版本发生变化，所有依赖它的启动阶段都需要进行相应的更新，否则可能会导致兼容性问题。

### 示例代码

假设我们有一个简单的 `fw_dynamic_info` 结构体定义如下：

```c
struct fw_dynamic_info {
    unsigned long magic;
    unsigned long version;
    unsigned long next_addr;
    unsigned long next_node;
    unsigned long options;
};
```

在 LOADER 中，我们可以这样填充并传递 `fw_dynamic_info`：

```c
// 定义并初始化 fw_dynamic_info 结构体
struct fw_dynamic_info dynamic_info = {
    .magic = 0xdeadbeef,  // 预定义的魔数
    .version = 1,         // 版本号
    .next_addr = 0x80000000,  // 下一启动阶段的地址
    .next_node = 0,       // 节点信息
    .options = 0          // 选项
};

// 将 fw_dynamic_info 的地址存储在 a2 寄存器中
register unsigned long a2 asm("a2") = (unsigned long)&dynamic_info;

// 跳转到 FW_DYNAMIC
asm volatile("j FW_DYNAMIC");
```

在 `FW_DYNAMIC` 中，我们可以这样访问 `fw_dynamic_info`：

```c
void fw_dynamic_entry(void) {
    register unsigned long a2 asm("a2");
    struct fw_dynamic_info *info = (struct fw_dynamic_info *)a2;

    if (info->magic == 0xdeadbeef && info->version == 1) {
        // 验证成功，继续启动过程
        void (*next_stage)(void) = (void (*)(void))info->next_addr;
        next_stage();
    } else {
        // 验证失败，处理错误
        // ...
    }
}
```

通过这种方式，`FW_DYNAMIC` 可以灵活地获取并使用前一个启动阶段传递的动态信息，从而更好地控制启动过程。


## Typical use as Library
1. External M-mode firmware linked to OpenSBI library
2. Example: open-source EDK2 (UEFI implementation) OpenSBI integration


~~~shell
ROM      ->  LOADFER    ->  RUNTIME     ->  BOOTLOADER  ->  OS
(M-mode)     M-MODE         M-mode          S-mode          S-mdoe
ZSBL         U-BOOT_SPEL    FW_DYNAMIC      U-Boot          Linux Kernel
             Coreboot
             Qemu
             Nemu
~~~

## reference
1. https://riscv.org/wp-content/uploads/2019/06/13.30-RISCV_OpenSBI_Deep_Dive_v5.pdf