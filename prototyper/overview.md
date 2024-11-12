在启动过程中，组件的调用顺序如下：

1. **电源开启（Power-On）**：
   - 系统电源被打开，硬件开始初始化。

2. **RustSBI Prototyper**：
   - 系统启动后，首先执行的是RustSBI Prototyper，它被配置为系统的第一阶段引导加载程序（Primary Bootloader）。
   - RustSBI Prototyper开始运行，并初始化硬件平台，包括设置内存、中断、设备等。

3. **U-Boot**：
   - RustSBI Prototyper加载并跳转到U-Boot，这是第二阶段引导加载程序（Secondary Bootloader）。
   - U-Boot继续硬件的初始化工作，并设置网络、文件系统等，以便能够从存储介质加载操作系统。

4. **Linux Kernel**：
   - U-Boot加载Linux Kernel到内存中，并传递启动参数（如根文件系统的位置、控制台配置等）。
   - Linux Kernel开始执行，进行进一步的硬件检测和初始化，准备挂载根文件系统。

5. **Root Filesystem (initramfs)**：
   - Linux Kernel挂载初始根文件系统（通常是initramfs或initrd）。
   - 初始根文件系统包含了启动操作系统所需的最小文件和工具。

6. **BusyBox**：
   - 一旦初始根文件系统被挂载，BusyBox提供的命令和工具被加载到内存中。
   - 这些工具包括shell、文件系统工具、网络工具等，它们是操作系统启动和运行所必需的。

7. **Linux User Space**：
   - Linux Kernel启动用户空间的初始化进程（通常是`init`或`systemd`）。
   - 初始化进程开始运行，启动系统服务和应用程序，完成操作系统的启动过程。

8. **QEMU**：
   - 在这个过程中，QEMU作为模拟硬件的平台，允许整个过程在没有实际硬件的情况下进行。
   - QEMU加载U-Boot和Linux Kernel，并模拟所需的硬件设备，如硬盘、网络卡等。

总结调用顺序：
1. 电源开启
2. RustSBI Prototyper（第一阶段引导加载程序）
3. U-Boot（第二阶段引导加载程序）
4. Linux Kernel
5. Root Filesystem (initramfs/initrd)
6. BusyBox（提供用户空间工具）
7. Linux User Space（初始化进程和系统服务）
8. QEMU（模拟硬件）

这个顺序确保了从硬件初始化到操作系统完全运行的整个过程的顺利进行。每个组件都在特定的阶段发挥作用，确保系统的正确引导和初始化。
