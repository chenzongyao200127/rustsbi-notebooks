~~~rust
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

~~~

## 串行设备是什么？

串行设备是一种通信设备，它允许数据以序列的形式（即一次一个比特）通过单一通信线路发送。
这种通信方式与并行通信相对，后者允许多个比特同时通过多个通道传输。串行设备广泛应用于计算机硬件和外设之间，以及远程通信系统中。

### 串行设备的工作原理

1. 数据传输：在串行通信中，数据被分解成单个比特，并按顺序一个接一个地通过通信线路发送。这种方式需要较少的物理线路，但传输速度可能较慢，因为数据是逐个比特传输的。
2. 同步：串行通信可以是同步的或异步的。在同步串行通信中，数据传输与一个时钟信号同步，这个时钟信号帮助接收设备确定何时读取数据。在异步串行通信中，数据包之间可能包含起始位和停止位，这些位帮助接收设备确定数据包的开始和结束。
3. 波特率：波特率是串行通信的速率，以每秒传输的符号数（比特）来衡量。波特率决定了数据传输的速度。

### 串行设备在嵌入式系统中的应用

在嵌入式系统中，串行设备通常用于调试（通过串行控制台输出调试信息）、与外部设备通信（如传感器或显示器）以及与其他计算机或网络设备进行远程通信。串行接口因其简单性和成本效益而在这些应用中非常流行。
