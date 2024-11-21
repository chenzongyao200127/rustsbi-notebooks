~~~rust
#![feature(naked_functions)]
#![no_std]
#![no_main]
#![allow(static_mut_refs)]

#[macro_use]
extern crate log;
#[macro_use]
mod macros;

mod board;
mod dt;
mod fail;
mod platform;
mod riscv_spec;
mod sbi;

use core::sync::atomic::{AtomicBool, Ordering};
use core::{arch::asm, mem::MaybeUninit};

use sbi::extensions;

use crate::board::{SBI_IMPL, SIFIVECLINT, SIFIVETEST, UART};
use crate::riscv_spec::{current_hartid, menvcfg};
use crate::sbi::console::SbiConsole;
use crate::sbi::extensions::{hart_extension_probe, Extension};
use crate::sbi::hart_context::NextStage;
use crate::sbi::hsm::{local_remote_hsm, SbiHsm};
use crate::sbi::ipi::{self, SbiIpi};
use crate::sbi::logger;
use crate::sbi::reset::SbiReset;
use crate::sbi::rfence::SbiRFence;
use crate::sbi::trap::{self, trap_vec};
use crate::sbi::trap_stack;
use crate::sbi::SBI;

pub const START_ADDRESS: usize = 0x80000000;
pub const R_RISCV_RELATIVE: usize = 3;

#[no_mangle]
extern "C" fn rust_main(_hart_id: usize, opaque: usize, nonstandard_a2: usize) {
    // Track whether SBI is initialized and ready.
    static SBI_READY: AtomicBool = AtomicBool::new(false);

    let boot_hart_info = platform::get_boot_hart(opaque, nonstandard_a2);
    // boot hart task entry.
    if boot_hart_info.is_boot_hart {
        let fdt_addr = boot_hart_info.fdt_address;

        // 1. Init FDT
        // parse the device tree.
        // TODO: shoule remove `fail:device_tree_format`.
        let dtb = dt::parse_device_tree(fdt_addr).unwrap_or_else(fail::device_tree_format);
        let dtb = dtb.share();

        // TODO: should remove `fail:device_tree_deserialize`.
        let tree =
            serde_device_tree::from_raw_mut(&dtb).unwrap_or_else(fail::device_tree_deserialize);

        // 2. Init device
        // TODO: The device base address should be find in a better way.
        let console_base = tree.soc.serial.unwrap().iter().next().unwrap();
        let clint_device = tree.soc.clint.unwrap().iter().next().unwrap();
        let cpu_num = tree.cpus.cpu.len();
        let console_base_address = console_base.at();
        let ipi_base_address = clint_device.at();

        // Initialize reset device if present.
        if let Some(test) = tree.soc.test {
            let reset_device = test.iter().next().unwrap();
            let reset_base_address = reset_device.at();
            board::reset_dev_init(usize::from_str_radix(reset_base_address, 16).unwrap());
        }

        // Initialize console and IPI devices.
        board::console_dev_init(usize::from_str_radix(console_base_address, 16).unwrap());
        board::ipi_dev_init(usize::from_str_radix(ipi_base_address, 16).unwrap());

        // 3. Init the SBI implementation
        unsafe {
            SBI_IMPL = MaybeUninit::new(SBI {
                console: Some(SbiConsole::new(&UART)),
                ipi: Some(SbiIpi::new(&SIFIVECLINT, cpu_num)),
                hsm: Some(SbiHsm),
                reset: Some(SbiReset::new(&SIFIVETEST)),
                rfence: Some(SbiRFence),
            });
        }

        // Setup trap handling.
        trap_stack::prepare_for_trap();
        extensions::init(&tree.cpus.cpu);
        SBI_READY.swap(true, Ordering::AcqRel);

        // 4. Init Logger
        logger::Logger::init().unwrap();

        info!("RustSBI version {}", rustsbi::VERSION);
        rustsbi::LOGO.lines().for_each(|line| info!("{}", line));
        info!("Initializing RustSBI machine-mode environment.");

        info!("Number of CPU: {}", cpu_num);
        if let Some(model) = tree.model {
            info!("Model: {}", model.iter().next().unwrap_or("<unspecified>"));
        }
        info!("Clint device: {}", ipi_base_address);
        info!("Console deivce: {}", console_base_address);
        info!(
            "Chosen stdout item: {}",
            tree.chosen
                .stdout_path
                .iter()
                .next()
                .unwrap_or("<unspecified>")
        );

        // TODO: PMP configuration needs to be obtained through the memory range in the device tree
        use riscv::register::*;
        unsafe {
            pmpcfg0::set_pmp(0, Range::OFF, Permission::NONE, false);
            pmpaddr0::write(0);
            pmpcfg0::set_pmp(1, Range::TOR, Permission::RWX, false);
            pmpaddr1::write(usize::MAX >> 2);
        }

        // Get boot information and prepare for kernel entry.
        let boot_info = platform::get_boot_info(nonstandard_a2);
        let (mpp, next_addr) = (boot_info.mpp, boot_info.next_address);

        // Start kernel.
        local_remote_hsm().start(NextStage {
            start_addr: next_addr,
            next_mode: mpp,
            opaque: fdt_addr,
        });

        info!(
            "Redirecting hart {} to 0x{:0>16x} in {:?} mode.",
            current_hartid(),
            next_addr,
            mpp
        );
    } else {
        // Non-boot hart initialization path.

        // TODO: PMP configuration needs to be obtained through the memory range in the device tree.
        use riscv::register::*;
        unsafe {
            pmpcfg0::set_pmp(0, Range::OFF, Permission::NONE, false);
            pmpaddr0::write(0);
            pmpcfg0::set_pmp(1, Range::TOR, Permission::RWX, false);
            pmpaddr1::write(usize::MAX >> 2);
        }

        // Setup trap handling.
        trap_stack::prepare_for_trap();

        // Wait for boot hart to complete SBI initialization.
        while !SBI_READY.load(Ordering::Relaxed) {
            core::hint::spin_loop()
        }
    }

    // Clear all pending IPIs.
    ipi::clear_all();

    // Configure CSRs and trap handling.
    unsafe {
        // Delegate all interrupts and exceptions to supervisor mode.
        asm!("csrw mideleg,    {}", in(reg) !0);
        asm!("csrw medeleg,    {}", in(reg) !0);
        asm!("csrw mcounteren, {}", in(reg) !0);
        use riscv::register::{medeleg, mtvec};
        // Keep supervisor environment calls and illegal instructions in M-mode.
        medeleg::clear_supervisor_env_call();
        medeleg::clear_illegal_instruction();
        // Configure environment features based on available extensions.
        if hart_extension_probe(current_hartid(), Extension::Sstc) {
            menvcfg::set_bits(
                menvcfg::STCE | menvcfg::CBIE_INVALIDATE | menvcfg::CBCFE | menvcfg::CBZE,
            );
        } else {
            menvcfg::set_bits(menvcfg::CBIE_INVALIDATE | menvcfg::CBCFE | menvcfg::CBZE);
        }
        // Set up vectored trap handling.
        mtvec::write(trap_vec as _, mtvec::TrapMode::Vectored);
    }
}

#[naked]
#[link_section = ".text.entry"]
#[export_name = "_start"]
unsafe extern "C" fn start() -> ! {
    core::arch::asm!(
        // 1. Turn off interrupt.
        "   csrw    mie, zero",
        // 2. Initialize programming langauge runtime.
        // only clear bss if hartid matches preferred boot hart id.
        "   csrr    t0, mhartid",
        "   bne     t0, zero, 4f",
        "   call    {relocation_update}",
        "1:",
        // 3. Hart 0 clear bss segment.
        "   lla     t0, sbss
            lla     t1, ebss
         2: bgeu    t0, t1, 3f
            sd      zero, 0(t0)
            addi    t0, t0, 8
            j       2b",
        "3: ", // Hart 0 set bss ready signal.
        "   lla     t0, 6f
            li      t1, 1
            amoadd.w t0, t1, 0(t0)
            j       5f",
        "4:", // Other harts are waiting for bss ready signal.
        "   li      t1, 1
            lla     t0, 6f
            lw      t0, 0(t0)
            bne     t0, t1, 4b", 
        "5:",
         // 4. Prepare stack for each hart.
        "   call    {locate_stack}",
        "   call    {main}",
        "   csrw    mscratch, sp",
        "   j       {hart_boot}",
        "  .balign  4",
        "6:",  // bss ready signal.
        "  .word    0",
        relocation_update = sym relocation_update,
        locate_stack = sym trap_stack::locate,
        main         = sym rust_main,
        hart_boot    = sym trap::msoft,
        options(noreturn)
    )
}

// Handle relocations for position-independent code
#[naked]
unsafe extern "C" fn relocation_update() {
    asm!(
        // Get load offset.
        "   li t0, {START_ADDRESS}",
        "   lla t1, .text.entry",
        "   sub t2, t1, t0",

        // Foreach rela.dyn and update relocation.
        "   lla t0, __rel_dyn_start",
        "   lla t1, __rel_dyn_end",
        "   li  t3, {R_RISCV_RELATIVE}",
        "1:",
        "   ld  t4, 8(t0)",
        "   bne t4, t3, 2f",
        "   ld t4, 0(t0)", // Get offset
        "   ld t5, 16(t0)", // Get append
        "   add t4, t4, t2", // Add load offset to offset add append
        "   add t5, t5, t2",
        "   sd t5, 0(t4)", // Update address
        "   addi t0, t0, 24", // Get next rela item
        "2:",
        "   blt t0, t1, 1b",

        // Return
        "   ret",
        R_RISCV_RELATIVE = const R_RISCV_RELATIVE,
        START_ADDRESS = const START_ADDRESS,
        options(noreturn)
    )
}

#[panic_handler]
fn panic(info: &core::panic::PanicInfo) -> ! {
    use riscv::register::*;
    println!(
        "[rustsbi-panic] hart {} {info}",
        riscv::register::mhartid::read()
    );
    println!(
        "-----------------------------
> mcause:  {:?}
> mepc:    {:#018x}
> mtval:   {:#018x}
-----------------------------",
        mcause::read().cause(),
        mepc::read(),
        mtval::read()
    );
    println!("[rustsbi-panic] system shutdown scheduled due to RustSBI panic");
    loop {}
}
~~~



# 启动 Overview

整个 `rust_main` 函数的流程可以分为以下几个主要阶段，每个阶段都有其特定的目的和操作：

#### 1. 硬件信息获取与设备树解析（Hardware Information Acquisition and Device Tree Parsing）
- 目的：在系统启动时，获取硬件配置信息和启动参数。
- 操作：
  - 通过 `platform::get_boot_hart` 获取启动硬件信息，包括设备树（FDT）地址。
  - 解析设备树，获取硬件资源信息，如控制台和IPI（处理器间中断）设备的基地址。

#### 2. 硬件设备初始化（Hardware Device Initialization）
- 目的：初始化控制台、IPI和其他必要的硬件设备，以便它们可以在操作系统中使用。
- 操作：
  - 根据设备树中的信息，初始化控制台和IPI设备。
  - 如果存在，初始化重置设备。

#### 3. SBI（Supervisor Binary Interface）实现初始化（SBI Implementation Initialization）
- 目的：设置SBI实现，这是RISC-V架构中用于操作系统和硬件之间交互的接口。
- 操作：
  - 初始化SBI实现，包括控制台、IPI、HSM（硬件状态管理）、重置和远程栅栏（RFence）等组件。
  - 设置陷阱处理，为操作系统的异常和中断处理做准备。

#### 4. 日志系统初始化（Logger Initialization）
- 目的：设置日志系统，以便在系统启动和运行时记录重要的信息和事件。
- 操作：
  - 初始化日志系统，打印RustSBI版本信息和启动日志。

#### 5. 处理器模式配置（Processor Mode Configuration）
- 目的：配置处理器模式，确保所有中断和异常都被正确地委托给 supervisor 模式。
- 操作：
  - 设置PMP（物理内存保护）配置，保护内存区域。
  - 配置CSR（控制和状态寄存器）和陷阱处理，以适应操作系统的需求。

#### 6. 启动信息获取与内核启动（Boot Information Acquisition and Kernel Start）
- 目的：获取启动信息，并启动操作系统内核。
- 操作：
  - 获取启动信息，包括下一个执行地址和特权模式。
  - 启动操作系统内核，将控制权从SBI转移到操作系统。

#### 7. 非启动核心的初始化（Non-Boot Core Initialization）
- 目的：对于非启动核心，等待启动核心完成SBI初始化，并设置必要的处理器配置。
- 操作：
  - 等待启动核心完成SBI初始化。
  - 配置PMP和陷阱处理，以确保非启动核心可以安全地运行。

#### 为什么要这么做：
这些阶段确保了系统从硬件到软件的平滑过渡，每一步都是构建一个稳定、安全和可预测的操作系统环境所必需的。
通过初始化硬件设备、设置SBI接口、配置处理器模式和启动操作系统内核，`rust_main` 函数为操作系统的启动和运行提供了必要的基础。
此外，对于多核系统，它还确保了所有核心都能够正确地初始化和同步，这对于系统的稳定性和性能至关重要。



## 硬件信息获取与设备树解析（Hardware Information Acquisition and Device Tree Parsing）

### 硬件信息获取

在计算机系统中，启动过程通常涉及到从一个或多个非易失性存储设备（如硬盘、SSD、EPROM等）加载操作系统。在系统启动的早期阶段，需要获取硬件配置信息，以便操作系统能够识别和配置硬件资源。这些信息包括CPU核心数、内存布局、外设（如串口、网络接口、图形适配器等）的地址和配置。

在RISC-V架构中，启动过程可能涉及到一个或多个硬件线程（hart），每个hart都需要知道自己是否是启动hart（boot hart）。
启动hart负责初始化系统，包括解析设备树、初始化硬件设备和启动操作系统内核。非启动hart则等待启动hart完成这些初始化步骤。

### 设备树（Device Tree）

设备树是一种用于描述硬件配置的数据结构，它在系统启动时提供给操作系统。设备树以树状结构组织，每个节点代表一个硬件设备或硬件资源。
设备树的源文件通常以`.dts`（Device Tree Source）格式编写，然后使用设备树编译器（如DTC）编译成二进制格式`.dtb`（Device Tree Blob）。

设备树包含的信息非常丰富，包括：

- **CPU核心**：包括核心数、每个核心的启动地址等。
- **内存**：包括内存的大小、类型（如DDR3、DDR4等）、地址范围等。
- **外设**：包括外设的类型、基地址、中断号等。
- **总线**：包括总线的类型（如I2C、SPI、UART等）和连接的设备。

### 设备树解析

在操作系统启动过程中，设备树被解析以获取硬件配置信息。这通常涉及到以下步骤：

1. **设备树地址获取**：启动hart通过启动参数获取设备树的地址。
2. **设备树解析**：使用设备树解析库（如libfdt）解析设备树，将其转换为操作系统可以操作的数据结构。
3. **硬件资源获取**：从解析后的设备树中提取硬件资源信息，如控制台的基地址、IPI设备的基地址等。

设备树解析是操作系统启动过程中的关键步骤，因为它为操作系统提供了必要的硬件配置信息，使得操作系统能够正确地识别和配置硬件资源。

### 为什么要这么做

使用设备树的好处包括：

- **灵活性**：设备树允许操作系统在不修改代码的情况下支持不同的硬件配置。
- **可维护性**：设备树将硬件配置信息集中管理，便于维护和更新。
- **可扩展性**：设备树支持新的硬件类型和配置，便于扩展和适应新的硬件。

通过设备树，操作系统能够以一种标准化和结构化的方式获取硬件配置信息，这对于操作系统的启动和运行至关重要。





## 硬件设备初始化（Hardware Device Initialization）

### 硬件设备初始化的背景知识

在操作系统启动过程中，硬件设备初始化是一个关键步骤。
操作系统需要与硬件设备进行交互，以提供用户所需的功能，例如显示输出、键盘输入、网络通信等。
因此，在操作系统内核启动之前，必须确保所有必要的硬件设备已经正确初始化。

### 硬件设备初始化的目的

1. 确保设备可用性：初始化过程确保硬件设备处于可用状态，准备好接收操作系统的指令。
2. 配置设备参数：根据设备树中提供的信息，设置设备的参数，如内存地址、中断号等。
3. 建立通信通道：初始化过程中，操作系统会建立与硬件设备的通信通道，例如配置串口通信参数。

### 硬件设备初始化的过程

1. 控制台设备初始化：
   - 控制台设备通常是系统启动过程中的第一个输出设备，用于显示启动信息和调试消息。
   - 在`rust_main`函数中，通过从设备树中获取控制台设备的基地址，然后调用`board::console_dev_init`函数来初始化控制台设备。

2. IPI（处理器间中断）设备初始化：
   - 在多核系统中，IPI是核心之间通信的重要机制，用于同步核心或发送中断信号。
   - 通过从设备树中获取IPI设备的基地址，然后调用`board::ipi_dev_init`函数来初始化IPI设备。

3. 重置设备初始化（如果存在）：
   - 重置设备用于在系统启动或运行中进行硬件重置。
   - 如果设备树中指定了重置设备，通过获取其基地址并调用`board::reset_dev_init`函数来初始化。

### 为什么要初始化硬件设备

- 系统稳定性：正确初始化硬件设备可以避免系统运行中出现硬件故障或崩溃。
- 性能优化：初始化过程中可以优化硬件设备的配置，提高系统性能。
- 功能实现：未初始化的硬件设备无法使用，初始化是实现设备功能的基础。

### 硬件设备初始化与操作系统的关系

- 内核与硬件的桥梁：硬件设备初始化为操作系统内核提供了与硬件交互的必要接口。
- 驱动程序的基础：硬件设备初始化通常涉及到加载和初始化设备驱动程序，这是操作系统管理硬件资源的基础。

通过硬件设备初始化，操作系统能够确保在启动过程中以及之后的运行中，所有硬件设备都能够按照预期工作，为用户和应用程序提供必要的硬件支持。







## SBI（Supervisor Binary Interface）实现初始化（SBI Implementation Initialization）

### SBI（Supervisor Binary Interface）的背景知识

SBI（Supervisor Binary Interface）是RISC-V架构中定义的一套二进制接口标准，用于在操作系统的监督模式（Supervisor Mode）和硬件之间进行交互。
SBI提供了一组标准化的函数和调用约定，使得操作系统能够以统一的方式执行诸如系统调用、中断处理、电源管理等操作。

### SBI实现初始化的目的

1. 标准化硬件交互：SBI定义了一组标准的接口，使得操作系统能够以一致的方式与硬件进行交互，无论硬件的具体实现如何。
2. 简化操作系统开发：通过提供统一的接口，SBI简化了操作系统的开发工作，因为开发者不需要针对每种硬件都编写特定的代码。
3. 提高系统稳定性：SBI通过标准化的接口来管理硬件，有助于减少硬件相关错误和系统崩溃。

### SBI实现初始化的过程

1. 初始化SBI结构体：
   - 在`rust_main`函数中，使用`unsafe`代码块来初始化`SBI_IMPL`全局变量，这是一个`MaybeUninit<SBI>`类型的变量，用于存储SBI实现的具体结构。
   - 结构体`SBI`包含了指向控制台、IPI、HSM（硬件状态管理）、重置和远程栅栏（RFence）等组件的指针。

2. 设置陷阱处理：
   - `trap_stack::prepare_for_trap`函数用于准备陷阱（异常和中断）处理所需的堆栈和上下文。
   - 这确保了当系统发生异常或中断时，操作系统能够捕获并处理这些事件。

3. 标记SBI为就绪：
   - 使用`SBI_READY.swap(true, Ordering::AcqRel)`来标记SBI实现已经初始化并准备好使用。
   - 这个原子操作确保了在多核系统中，所有核心都能看到SBI实现的最新状态。
