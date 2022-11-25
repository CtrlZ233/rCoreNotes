由于当前的rCore-N仅支持进程级任务的创建和调度，而在共享调度器有着同时使用多个CPU来执行协程的需求，因此在rCore-N中引入对线程的支持。

## 内核线程还是用户线程？
共享调度器的线程是为了提供**并行**执行同一个进程中不同协程的能力。

-   从概念上看，需要提供用户栈来保存当前协程的函数栈，这一点，内核线程和用户线程都能够支持。
-   同时，当某个用户协程阻塞在内核中（不考虑异步系统调用的情况），也需要保存当前线程在内核中的函数栈，这需要内核线程的支持。
-   此外，单纯的用户线程（指类似绿色线程的实现）并不能利用多核CPU的优势，无法仅仅依靠用户态的调度来进行多核的并行。

基于上述原因，我们参考 [rCore-Tutorial ch8](http://rcore-os.cn/rCore-Tutorial-Book-v3/chapter8/index.html) ，实现了rCore-N对内核线程的支持。在共享调度器的概念中，线程只是作为栈的提供者，因此仅仅将内核线程的接口提供给共享调度器，而不提供给用户程序。

此外，未来在完全实现了异步系统调用，协程不会阻塞在内核态之后，会考虑只对每个进程提供一个内核栈，达到更小的资源开销。

## 几个适配点
内核线程的结构和 [rCore-Tutorial ch8](http://rcore-os.cn/rCore-Tutorial-Book-v3/chapter8/index.html) 基本相似，但是rCore-N中对用户态中断的处理是进程级别的；此外，[rCore-Tutorial ch8](http://rcore-os.cn/rCore-Tutorial-Book-v3/chapter8/index.html) 没有考虑到多核环境下的Race Condition问题，因此就几个主要适配点简单说明一下。

### 1. 适配用户态中断
仍然保持进程级别的用户态中断处理，每个进程只有一个 `UserTrapQueue`, 为了简便处理，我们只使用进程下的一个线程来处理中断，在线程从内核返回的 `trap_return` 中，判断当前的线程 id是否为指定的处理中断的线程id：

```rust
pub fn trap_return() -> ! {  
    unsafe {  
        sstatus::clear_sie();  
    }    
    current_process()  
        .unwrap()  
        .acquire_inner_lock()  
        .restore_user_trap_info();
    ...
}

pub fn restore_user_trap_info(&mut self) {  
    use riscv::register::{uip, uscratch};  
    if self.is_user_trap_enabled() && sys_gettid() as usize == self.user_trap_handler_tid {  
        if let Some(trap_info) = &mut self.user_trap_info {  
            if !trap_info.get_trap_queue().is_empty() {  
                trace!("restore {} user trap", trap_info.user_trap_record_num());  
                uscratch::write(trap_info.user_trap_record_num());  
                unsafe {  
                    uip::set_usoft();  
                }            
            }        
        }    
    }
}
```

### 2.多核环境下的Race Condition
在多核环境下，往往存在多个CPU同时执行同一个进程中的不同线程的情况，它们可能同时进入内核执行相关代码。因此可能会产生很多Race Condition，以下是我们开开发过程中遇到的几个典型的问题和一些心得。

-   为了避免死锁，我们约定在获取 `task_inner` 之前要先获取 `process_inner`，以相同的顺序获取锁，防止死锁发生。
-   只有在将每个线程的所有上下文都设置完成后再加入就绪队列中进行调度。
-   与 `waitpid` 类似，参考rCore-N的，在 `waittid` 时需要设置一个读写锁，防止某个线程exit时修改自己的状态，而主线程在 `waittid` 时读到脏数据。
-   某些实现了Drop的线程级资源（如内核栈）在释放时会获取 `process_inner` 的锁，因此在释放时请确保当前没有获取 `process_inner`。