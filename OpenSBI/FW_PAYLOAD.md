# FW_PAYLOAD

## OpenSBI firmware with the next booting stage as a payload
1. Any S-mode BOOTLOADER/OS image as the payload to OpenSBI FW_PAYLOAD
2. Allows overriding device tree blob (i.e. DTB)
3. Very similar to BBL hence fits nicely in existing boot-flow of SiFive Unleashed board


## Down-side:
1. We have to re-create FW_PAYLOAD image whenever OpenSBI or the BOOTLOADER (U-Boot) changes
2. No mechanism to pass parameters from previous booting stage (i.e. LOADER) to FW_PAYLOAD


OpenSBI (Open Source Supervisor Binary Interface) 是一个开源的固件，用于 RISC-V 架构的处理器。它实现了 RISC-V 的 SBI（Supervisor Binary Interface）规范，为操作系统提供了与底层硬件交互的接口。OpenSBI 固件可以加载并执行下一个启动阶段的 payload（有效载荷），例如引导加载程序或操作系统镜像。

### 1. 任何 S 模式的引导加载程序/操作系统镜像作为 OpenSBI FW_PAYLOAD

在 RISC-V 架构中，S 模式是指 supervisor 模式，它是操作系统内核运行的特权模式之一。OpenSBI 可以将 S 模式的引导加载程序或操作系统镜像作为其 payload。这意味着你可以将以下类型的镜像作为 payload 传递给 OpenSBI：

- 引导加载程序：如 U-Boot、Coreboot 等。
- 操作系统内核：如 Linux 内核。

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
1. ROM 启动代码：首先从 ROM 中运行初始启动代码。
2. OpenSBI：加载并运行 OpenSBI 固件。
3. OpenSBI 初始化：OpenSBI 初始化硬件和 SBI 接口。
4. 加载 Payload：OpenSBI 加载指定的 payload（如 U-Boot 或 Linux 内核）。
5. 执行 Payload：OpenSBI 将控制权交给 payload，继续启动过程。

这种设计使得 OpenSBI 可以无缝集成到现有的启动流程中，提供额外的功能和灵活性，同时保持与现有系统的兼容性。
