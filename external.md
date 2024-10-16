
# 物理机上的情况

物理设备会连接到plic的中断源上，plic又会连接到核心

核心可以通过mmio读写设备寄存器，也可以对plic进行设置，来配置各个中断源的priority、enable等。

当物理设备发送中断信号后，plic就会根据配置，看中断源是否enable、其priority是否超过threshold等，挑选出一个最佳中断源，将其编号x写入claim寄存器，并向核心发送中断。

核心收到外部中断后，读取plic的claim寄存器，得知是编号为x的设备发送了中断，于是做相应处理，并将编号x重新写回claim寄存器来告知plic自己已经完成了。

plic发现claim被重新写入后，便可以传递下一个最佳中断信号给核心。





# 虚拟机：陷入 + 模拟

陷入即“虚拟机不可以直接访问（直通）设备和plic”，这意味这二阶段页表中不做设备和plic的映射。

这样做是为了通过pagefault“截获”虚拟机对设备的访问。若虚拟机能够直接访问设备，那么vmm将感知不到guest/vcpu访问设备的行为，也就不知道该将设备的响应发送给哪个guest/vcpu。

而模拟一方面可以构造不存在的物理设备，另一方面给多vm提供了更好的设备抽象。


以串口输入/输出为例。

当guest打印输出时，访问串口设备并触发pagefault，通过地址可以得知guest是想访问串口设备。此时调用模拟串口设备的接口，模拟设备根据guest的读写操作更新自己的状态，例如可以将guest的输出字符存入一个队列。然后模拟设备可以调用虚拟plic发出中断信号，虚拟plic会根据配置给特定的vcpu发送外部中断。

vcpu收到外部中断后，便希望查询plic获知信号由哪个设备发出。此时访问plic也会触发pagefault，进而调用虚拟plic的claim接口，返回串口设备的中断号。guest得知是串口设备发送中断后，在虚拟机中做相应处理，然后向plic通知complete，这也会应用到虚拟plic中。

前台终端绑定特定guest后，从其虚拟串口中的队列中不断取出字符并打印。



>前台终端：生产者-消费者
>更复杂设备？


---

键盘输入：

键盘按下后，串口发送物理中断，并通过物理plic路由给物理cpu，cpu收到外部中断后查询物理plic的claim来得知是哪个设备发出中断；发现是串口后，读取内容，并根据发送给当前前台终端绑定的模拟串口，模拟串口更新自身状态（如放入输入字符）后，调用虚拟plic发送中断信号，虚拟plic再根据其配置通知各vcpu有外部中断。

vcpu收到中断后，查询模拟plic的claim（通过pagefault）来得知是哪个设备发出中断，得知是模拟串口设备（vcpu本身不知道是模拟的），于是访问模拟串口设备（通过pagefault）并获得输入的字符，最后向虚拟plic通知complete，等待下一个中断。



>本机plic
>crate怎么没了
>https://github.com/arceos-org/arceos/pull/64
>https://github.com/arceos-org/arceos/blob/5f20009e1e3045311fd870bafe56cc6f4e38f194/crates/plic/src/lib.rs
>https://github.com/arceos-org/arceos/blob/5f20009e1e3045311fd870bafe56cc6f4e38f194/modules/axhal/src/platform/qemu_virt_riscv/irq.rs
>如何直接在外部中断vmexit后，接使用物理plic？会不会造成和arceos中断处理的重复？
>会不会同时运行两个app？假如有arceos的其他应用，如何共存？



# 接口

```rust
loop {
    match vm.run_vcpu(vcpu_id) {
        Ok(exit_reason) => match exit_reason {
            ...
            AxVCpuExitReason::ExternalInterrupt { vector } => {
                // 查询物理plic，获取发送中断的设备信息
                let irq_no = PLIC.claim();
                // handle_irq(irq_no); ?
                match irq_no {
                    serial => {
          						// 读取物理串口
          						let ch = uart_8250_read();
          						// 将字符发送给模拟串口设备
          						let vserial_dev = CONSOLE.current_bind();
          						vserial_dev.send(ch);
        					  }
        					  ...
				        }
				
                // complete
                PLIC.complete(irq_no);
            }

            Err(err) => {
                warn!("VM[{}] run VCpu[{}] get error {:?}", vm_id, vcpu_id, err);
                wait(vm_id)
            }
        }
    }
}

//fn handle_irq(irq_no: u32) {
//	match irq_no {
//		serial => {
//			let ch = serial_read();
//			
//		}
//		...
//	}
//}
```



