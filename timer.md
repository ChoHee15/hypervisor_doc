
# data structure

timer_list：一个小根堆，用于维护多个定时器及其回调函数。

通过``timer_list.set(deadline: u64, handler: TimerEventFn)``来设置一个到期时间为deadline的事件，其回调函数为handler

通过``timer_list.expire_one(now: TimeValue)``来获得时间最值最小的过期事件，若没有则返回``None``

>一次处理多个到期事件？
>```rust
>while if let ev = list.expire_one(now) {
>	ev.callback()
>	...
>}
>```



# design

## set timer

task_loop -> vm.run_vcpu() -> vcpu.run() -> _run_guest -> sbi::set_timer -> self.vmexit_handler -> return Reason::SetTimer(callback) to task_loop -> register_timer(callback)




## timer injection

task_loop -> vm.run_vcpu() -> vcpu.run() -> _run_guest -> timer irq -> self.vmexit_handler -> return Reason::TimerIrq to task_loop -> check_events() & scheduler_next_event() -> callback::hvip::set_vstip()


>```rust
>//  vcpu是否能提供某种统一接口来保证各种情况下中断的发生？
>callback {
>	self.assert_irq(IRQ_TIMER); // vcpu assert irq
>}
>```




# situation

## case1

timer到期时，vcpu未运行（可能与timer_list在同一核也可能在不同核）。

此时只需直接修改vcpu即可。

## case2

timer到期时，vcpu正在运行，和timer_list在同一核。

>这种情况应该不存在？在同一核的情况下，既然在运行timer_list事件处理，就不会在运行vcpu

## case3

timer到期时，vcpu正在运行，和timer_list在不同核。



## case4

arceos时钟中断。

TODO











# tmp

同arm进行分类讨论

[Virtual GIC Design · Issue #27 · arceos-hypervisor/arceos-umhv · GitHub](https://github.com/arceos-hypervisor/arceos-umhv/issues/27)

### case1

单vcpu固定在单个物理核上，回调中直接注入没有问题。

### case2

多vcpu在单个物理核上，回调中使用``vcpu.assert_irq()``可以为不在运行的vcpu设置中断。

## case3

单/多vcpu在多个物理核上调度执行。

>当vcpu在hart0上运行时设置了定时器，然后vcpu被调度到hart1上执行，此时hart0上的定时器到期，该如何通知hart1上正在运行的vcpu中断到来并退出虚拟机？
>
>1. 完全不管，hart0处理时钟事件时，若vcpu不在自己这照样``vcpu.assert_irq()给vcpu标记``。当vcpu在hart1上退出一次虚拟机后，下次执行自然会注入中断。但是这样会使得中断不够及时。
>2. 执行回调时检查vcpu是否在当前核心上？hart0处理时钟事件时发现vcpu不在自己这后，不做处理，也不额外采取行动通知hart1。等hart1处理时钟事件时再让hart1执行回调。
>3. hart0处理时钟事件时发现vcpu不在自己这后，自己不做处理，查找得知vcpu在hart1上后，向hart1发送ipi，hart1收到后检查timer_list并执行回调（需要在软件中断里增加复杂逻辑，但会让中断及时一些）
>4. 每个hart都有自己的timer_list，当vcpu迁移时，连带时钟事件一起被迁移到新的timer_list里。
>5. ……




















