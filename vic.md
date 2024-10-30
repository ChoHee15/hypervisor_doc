# 虚拟中断控制器 接口与定义
# structure

由于结构上的差异，各个架构的中断控制器可能不会共用数据结构，而是编写不同的struct，实现同一个trait。

如果需要的话，中断控制器连接中断源和目标，以下的结构应该是相似的：

```rust
// 中断源相关
struct Source {
    // 中断源的优先级
    priority: u32,
    // 中断源的等待处理标识
    pending: bool,
}

// 目标（核心）相关
struct Target {
    // 连接的vcpu
    vcpu_id: u32,
    or
    vcpu: Arc<Vcpu>,

    // 目标门限，当中断源优先级超过此门限时才有效
    threshold: u32,
    // 此目标对于每个中断源的使能
    enable: [bool; SOURCE_NUM],
    // 存放最优中断源
    claim: u32,
}
```

>问题：
>
>一些状态可能是bit表示的，而对其的读写是基于寄存器大小的。
>例如在riscv PLIC中，中断源ID为0-31的pending位被放置在一个4B的寄存器内。guest会直接读取这个寄存器，此时就需要遍历ID为0-31的Source的pending，将其合成为一个u32并返回。
>这样的效率可能就不如“对寄存器模拟”，即直接提供一个u32变量，而不是将其分散到每个Source结构的pending bool变量中。
>
>同时，不同架构/控制器间可能会存在不一致：假如控制器A实现enable是按bit进行的，其倾向于直接模拟整个寄存器``enable: [u32; MAX_SOURCE / 32]``。而控制器B并非如此，它使用1B实现enable，它可能倾向于模拟``enable: [u8; MAX_SOURCE]``。
>
>这些可能的差异，会给不同架构/控制器使用统一结构带来问题。








# interface

## basic

基础接口。其中关于数据类型，发送中断信号的接口，是否定义claim/complete，以及不同数据长度的读写，有多种方案，需要做具体讨论。

```rust
trait InterruptController {
    // 为设备提供发送中断信号的接口，包括连接哪个中断源（待定），和触发方式（电平/边缘）
    fn send_irq(source_id: u32, level: bool);

    // 写入，第一种实现
    fn write_u32(addr: usize, val: u32);
    fn write_u16(addr: usize, val: u16);
    fn write_u8(addr: usize, val: u8);
    ...

    // 读取，第一种实现
    fn read_u32(addr: usize) -> u32;
    fn read_u16(addr: usize) -> u16;
    fn read_u8(addr: usize) -> u8;
    ...

    // Claim过程，查看当前的最优中断源，可能会在read中使用而不需要对外暴露
    fn claim() -> u32;

    // Complete过程，告知中断控制器处理完成，可能会在write中使用而不需要对外暴露
    fn complete(val: u32);

    // get/set各种属性，可能并不需要对外暴露
    // fn set_priority(source_id: u32, priority: u32);
    // fn get_priority(source_id: u32) -> u32;
    // fn set_enable(target_id: u32, enable: bool);
    // fn get_enable(target_id: u32) -> bool;
    ...
	
}
```

>关于数据类型：
>上文假设中断源ID为u32类型，claim寄存器为u32类型。这类结构的大小，在不同控制器中可能是不一致的。

>关于发送中断信号的接口：
>当前的设计，即``fn send_irq(source_id: u32, level: bool);``，其实假设的是，设备知道自己对应于哪个中断源ID；模拟设备使用此方法时，会直接附上自己的ID。因此，需要确定这一假设是否成立。
>
>其次，现实中，物理设备连接哪个中断源，就在哪个中断源上，它无法影响其他的中断源。而在当前的设计中，模拟设备理论上可以用任意中断源ID调用``fn send_irq(source_id: u32, level: bool);``。是否需要一种机制，保证设备只能以自己的source id操作控制器，还是将这一责任交给模拟设备的实现？

>关于是否定义claim/complete：
>pub trait的所有方法都是对外可见的，然而claim/complete作为可能共有的行为，在一些实现中会蕴含于读/写行为（如riscv PLIC）。因此可能不需要被定义。

>关于不同数据长度的读写，见下文的其他实现方式


## 读写实现：数据长度作为参数

数据长度作为参数，使用最大变量（如u64）作为数据容器，在不同分支中做截断/扩展。

```rust
// 读取，第二种实现
fn read(addr: usize, len: usize) -> u64 {
    match len {
        8 => {
            let res: u8 = ...;
            return res as u64
        }
        16 => {
            let res: u16 = ...;
            return res as u64
        }
        32 => {...}
        ...
    }
}

// 写入，第二种实现
fn write(addr: usize, val: u64, len: usize) -> u64 {
    match len {
      8 => {
        let data: u8 = val as u8;
	...
      }
      16 => {
        let data: u16 = val as u16;
        ...
      }
      32 => {...}
      ...
    }
}
```



## 读写实现：泛型

使用泛型接口，并定义一个``trait WriteRead<T>``，实现具体控制器时，为控制器实现具体类型的``WriteRead<T>``。

```rust
trait InterruptController {
    fn send_irq(source_id: u32, level: bool);

    // 泛型的写接口
    fn write<T>(&self, addr: usize, val: T)
    where
        Self: WriteRead<T>,
    {
        Self::write_impl(addr, val);
    }

    // 泛型的读接口
    fn read<T>(&self, addr: usize) -> T
    where
        Self: WriteRead<T>,
    {
        Self::read_impl(addr)
    }
}

trait WriteRead<T> {
    fn write_impl(addr: usize, val: T);
    fn read_impl(addr: usize) -> T;
}
```

例子：

```rust
// 自定义控制器
pub struct myic {
    base: u64,
    size: u32,
    data: [u32; 10]
}

// 控制器自有方法
impl myic {
    pub fn new(base: u64, size: u32) -> Self {
        myic{
            base,
            size,
            data: [0; 10],
        }
    }
}

// 实现接口
impl InterruptController for myic {
    fn send_irq(&self, source_id: u32, level: bool) {
        println!("myic send irq");
    }
    ...
}

// 实现对u8的读写
impl WriteRead<u8> for myic{
    fn read_impl(addr: usize) -> u8 {
        println!("myic read u8");
        8
    }

    fn write_impl(addr: usize, val: u8) {
        println!("myic write u8");
    }
}

// 实现对u32的读写
impl WriteRead<u32> for myic {
    fn read_impl(addr: usize) -> u32 {
        println!("myic read u32");
        32
    }

    fn write_impl(addr: usize, val: u32) {
        println!("myic write u32");
    }
}

// main.rs
fn main() {
    let ic = myic;

    ic.send_irq(1, true); // myic send irq
    
    ic.write(0x100, 5 as u32); // myic write u32
    ic.write::<u8>(0x100, 4); // myic write u8

    ic.read::<u8>(0x100); // myic read u8
    ic.read::<u32>(0x100); // myic read u32
}

```
