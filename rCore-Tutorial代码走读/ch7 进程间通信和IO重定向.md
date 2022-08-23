# rCore代码笔记-ch7
[[2022-08-13]]

到目前为止，我们引入了进程的概念，然而，进程与进程之间几乎是相互隔离的，进程间无法方便地“沟通”，导致建成之间无法相互协作来处理某些事物，如果能让不同进程实现数据共享与交互，就能把不同程序的功能组合在一起，实现更加强大和灵活的功能。同时，当某个事件发生时，如何能便捷地通知某个进程，该事件发生了也是我们本章需要关注的话题。

本章中，我们将实现管道（Pipe）来支持进程之间的数据交互，将实现信号（Signal）来实现进程间的消息通知。

参考资料：[第七章：进程间通信和IO重定向 — rCore-Tutorial-Book-v3 3.6.0-alpha.1 文档 (rcore-os.github.io)](http://rcore-os.cn/rCore-Tutorial-Book-v3/chapter7/)

## 进程间通信
在Linux中，进程间通信主要分为以下几种方式：
- 管道通信
- 消息队列
- 信号量通信
- 信号
- 共享存储
- 套节字通信

在本节中，我们将实现管道通信（Pipe）和信号通信（Signal）。

### 管道
一个管道分为读端和写端，一个进程在写端写入数据，另一个进程在读端读出数据，由于管道是一个队列，读取数据的时候会从队头读取并弹出数据，而写入数据的时候则会把数据写入到队列的队尾。由于管道的缓冲区大小是有限的，一旦整个缓冲区都被填满就不能再继续写入，就需要等到读端读取并从队列中弹出一些数据之后才能继续写入。当缓冲区为空的时候，读端自然也不能继续从里面读取数据，需要等到写端写入了一些数据之后才能继续读取。

我们先来简单地分析以下用户程序是如何使用管道进行通信的：
```rust
// user/src/bin/pipetest.rs
pub fn main() -> i32 {  
    // create pipe  
    let mut pipe_fd = [0usize; 2];  
    pipe(&mut pipe_fd);  
    // read end  
    assert_eq!(pipe_fd[0], 3);  
    // write end  
    assert_eq!(pipe_fd[1], 4);  
    if fork() == 0 {  
        // child process, read from parent  
        // close write_end        
        close(pipe_fd[1]);  
        let mut buffer = [0u8; 32];  
        let len_read = read(pipe_fd[0], &mut buffer) as usize;  
        // close read_end  
        close(pipe_fd[0]);  
        assert_eq!(core::str::from_utf8(&buffer[..len_read]).unwrap(), STR);  
        println!("Read OK, child process exited!");  
        0  
    } else {  
        // parent process, write to child  
        // close read end        
        close(pipe_fd[0]);  
        assert_eq!(write(pipe_fd[1], STR.as_bytes()), STR.len() as isize);  
        // close write end  
        close(pipe_fd[1]);  
        let mut child_exit_code: i32 = 0;  
        wait(&mut child_exit_code);  
        assert_eq!(child_exit_code, 0);  
        println!("pipetest passed!");  
        0  
    }  
}
```

上面的用户程序，先使用 `pipe`系统调用注册了一个管道（两个fd，一个读端，一个写端）。然后调用 `fork` 系统调用创建一个子进程，在子进程中关闭写端，从读端读数据，在父进程中关闭读端，在写端写入数据，这样，父进程写入的数据就能在子进程中读出。这就是管道通信。

用法示例如上，我们现在来看一下内核是如何对管道通信提供支持的。

`pipe` 系统调用返回管道读端和写端的文件描述符。

```rust
// os/src/syscall/fs.rs
pub fn sys_pipe(pipe: *mut usize) -> isize {  
    let task = current_task().unwrap();  
    let token = current_user_token();  
    let mut inner = task.inner_exclusive_access();  
    let (pipe_read, pipe_write) = make_pipe();  
    let read_fd = inner.alloc_fd();  
    inner.fd_table[read_fd] = Some(pipe_read);  
    let write_fd = inner.alloc_fd();  
    inner.fd_table[write_fd] = Some(pipe_write);  
    *translated_refmut(token, pipe) = read_fd;  
    *translated_refmut(token, unsafe { pipe.add(1) }) = write_fd;  
    0  
}

/// Return (read_end, write_end)  
pub fn make_pipe() -> (Arc<Pipe>, Arc<Pipe>) {  
    let buffer = Arc::new(unsafe { UPSafeCell::new(PipeRingBuffer::new()) });  
    let read_end = Arc::new(Pipe::read_end_with_buffer(buffer.clone()));  
    let write_end = Arc::new(Pipe::write_end_with_buffer(buffer.clone()));  
    buffer.exclusive_access().set_write_end(&write_end);  
    (read_end, write_end)  
}

// impl TaskControlBlockInner 
pub fn alloc_fd(&mut self) -> usize {  
    if let Some(fd) = (0..self.fd_table.len()).find(|fd| self.fd_table[*fd].is_none()) {  
        fd  
    } else {  
        self.fd_table.push(None);  
        self.fd_table.len() - 1  
    }  
}
```

从用户态传入一个用户空间里的地址，用户保存内核态分配的两个文件描述符，而两个文件描述符很明显就是当前进程文件描述符表所存的两个 `Pipe` 的下标。

`Pipe` 是实现了 `File Trait` 结构体，当然，可以看作读写受限的 `File` :
```rust
pub struct Pipe {  
    readable: bool,  
    writable: bool,  
    buffer: Arc<UPSafeCell<PipeRingBuffer>>,  
}  
  
impl Pipe {  
    pub fn read_end_with_buffer(buffer: Arc<UPSafeCell<PipeRingBuffer>>) -> Self {  
        Self {  
            readable: true,  
            writable: false,  
            buffer,  
        }    
    }    
    pub fn write_end_with_buffer(buffer: Arc<UPSafeCell<PipeRingBuffer>>) -> Self {  
        Self {  
            readable: false,  
            writable: true,  
            buffer,  
        }    
    }
}
```

`Pipe`所提供的两个方法，分别返回了只可读的 `Pipe` 和只可写的 `Pipe` 。

在操作系统中，`Pipe` 提供了一个共享缓存区 `PipeRingBuffer`，读端从这里读出数据，写端从这里写入数据：
```rust
pub struct PipeRingBuffer {  
    arr: [u8; RING_BUFFER_SIZE],  
    head: usize,  
    tail: usize,  
    status: RingBufferStatus,  
    write_end: Option<Weak<Pipe>>,  
}
```
`PipeRingBuffer` 保存读端的弱引用，这是由于在某些情况下需要确认该管道所有的写端是否都已经被关闭了，通过这个字段很容易确认这一点，后面我们会介绍到。

同时为 `PipeRingBuffer` 实现一些简单的方法：
```rust
impl PipeRingBuffer {
	// 设置写端的弱引用
	pub fn set_write_end(&mut self, write_end: &Arc<Pipe>);
	// 写入一个byte
	pub fn write_byte(&mut self, byte: u8);
	// 读出一个byte
	pub fn read_byte(&mut self) -> u8;
	// 可读的byte数
	pub fn available_read(&self) -> usize;
	// 可写的byte数
	pub fn available_write(&self) -> usize;
	// 读端是否关闭
	pub fn all_write_ends_closed(&self) -> bool
}
```

然后我们在用户态对 `Pipe` 的读写端调用 `Read` 和 `Write` 时，最后会调用到为 `Pipe` 实现的 `File Trait` ：
```rust
impl File for Pipe {
	fn read(&self, buf: UserBuffer) -> usize {
        assert!(self.readable());
        let mut buf_iter = buf.into_iter();
        let mut read_size = 0usize;
        loop {
            let mut ring_buffer = self.buffer.exclusive_access();
            let loop_read = ring_buffer.available_read();
            if loop_read == 0 {
                if ring_buffer.all_write_ends_closed() {
                    return read_size;
                }
                drop(ring_buffer);
                suspend_current_and_run_next();
                continue;
            }
            // read at most loop_read bytes
            for _ in 0..loop_read {
                if let Some(byte_ref) = buf_iter.next() {
                    unsafe {
                        *byte_ref = ring_buffer.read_byte();
                    }
                    read_size += 1;
                } else {
                    return read_size;
                }
            }
        }
    }
    fn write(&self, buf: UserBuffer) -> usize;
}
```

在读端的时候，我们会循环对 `ring_buffer` 进行读操作，直到传入的 `UserBuffer` 读满或写端已经全部关闭为止，如果 `ring_buffer` 暂时没有数据，则将读任务暂时挂起，等待写端写入。

而写端也是类似的实现，在此不再赘述。

### 信号
信号（Signals）是类UNIX操作系统中实现进程间通信的一种异步通知机制，用来提醒某进程一个特定事件已经发生，需要及时处理。

信号的发送方可以是进程或操作系统内核，进程通过系统调用 `kill` 向其他进程发送信号；而内核在碰到特定事件的时候，如用户对当前进程按下 `Ctrl+C` 按键时，内核收到包含 `Ctrl+C` 按键的外设中断和按键信息，并会向正在运行的当前进程发送 `SIGINT` 信号，将其终止。

信号的接收方是一个进程，进程会根据用户程序的设置来对信号进行相应的处理：
- 忽略
- 捕获：调用相应的处理函数进行处理。
- 默认：使用内核默认的处理方式，大多数情况下是杀死或忽略。

同样的，我们还是用一个用户程序例子来了解以下信号的用法：
```rust
// user/src/bin/sig_sample.rs
fn func() {  
    println!("user_sig_test succsess");  
    sigreturn();  
}  
  
#[no_mangle]  
pub fn main() -> i32 {  
    let mut new = SignalAction::default();  
    let old = SignalAction::default();  
    new.handler = func as usize;  
  
    println!("signal_simple: sigaction");  
    if sigaction(SIGUSR1, &new, &old) < 0 {  
        panic!("Sigaction failed!");  
    }    
    println!("signal_simple: kill");  
    if kill(getpid() as usize, SIGUSR1) < 0 {  
        println!("Kill failed!");  
        exit(1);  
    }    
    println!("signal_simple: Done");  
    0  
}

// user/src/lib.rs
#[repr(C)]  
#[derive(Debug, Clone, Copy)]  
pub struct SignalAction {  
    pub handler: usize,  
    pub mask: SignalFlags,  
}
```

我们为新的 `SignalAction` 设置了一个新的信号处理函数 `func`，在函数例用 `sigreturn` 返回，然后用 `sigaction` 系统调用将`SignalAction`注册到信号`SIGUSR1` 对应的处理操作中，注册完成后，调用 `kill` 系统调用向当前进程发送 `SIGUSR1` 信号，此时当前进程收到信号之后就会调用 `func` 函数，函数执行结束后用 `sigreturn` 返回被信号打断之前的处理流继续执行。这就是整个信号的处理流程。

其中主要涉及到的系统调用有：
- `sigaction` ：设置信号处理动作。
- `kill` ：发送信号。
- `sigreturn` ：清除堆栈帧，从信号处理例程返回。
- `sigprocmask`：设置要屏蔽的信号。

进程控制块所需要新增维护的变量有：
```rust
pub struct TaskControlBlockInner {  
	...
	// 要响应的信号
    pub signals: SignalFlags,  
    // 要屏蔽的信号
    pub signal_mask: SignalFlags,  
    // 正在处理的信号
    pub handling_sig: isize,  
    // 信号处理例程表
    pub signal_actions: SignalActions,  
    // 任务是否已经被杀死了 
    pub killed: bool,  
    // 任务是否已经被暂停了 
    pub frozen: bool,  
    // //被打断的trap上下文
    pub trap_ctx_backup: Option<TrapContext>,   
}
```
#### sigaction
设置信号处理动作：
```rust
pub fn sys_sigaction(
    signum: i32,
    action: *const SignalAction,
    old_action: *mut SignalAction,
) -> isize {
    let token = current_user_token();
    if let Some(task) = current_task() {
        let mut inner = task.inner_exclusive_access();
        if signum as usize > MAX_SIG {
            return -1;
        }
        if let Some(flag) = SignalFlags::from_bits(1 << signum) {
            if check_sigaction_error(flag, action as usize, old_action as usize) {
                return -1;
            }
            let old_kernel_action = inner.signal_actions.table[signum as usize];
            if old_kernel_action.mask != SignalFlags::from_bits(40).unwrap() {
                *translated_refmut(token, old_action) = old_kernel_action;
            } else {
                let mut ref_old_action = *translated_refmut(token, old_action);
                ref_old_action.handler = old_kernel_action.handler;
            }
            let ref_action = translated_ref(token, action);
            inner.signal_actions.table[signum as usize] = *ref_action;
            return 0;
        }
    }
    -1
}
```
内核会先检查传入的处理例程和对应的信号是否非法，然后会将之前的处理例程写回用户空间，将新的处理例程写入内核的`signal_actions` 中。
#### kill
向特定 `pid` 的进程发送信号：
```rust
pub fn sys_kill(pid: usize, signum: i32) -> isize {  
    if let Some(task) = pid2task(pid) {  
        if let Some(flag) = SignalFlags::from_bits(1 << signum) {  
            // insert the signal if legal  
            let mut task_ref = task.inner_exclusive_access();  
            if task_ref.signals.contains(flag) {  
                return -1;  
            }            task_ref.signals.insert(flag);  
            0  
        } else {  
            -1  
        }  
    } else {  
        -1  
    }  
}
```
内核会根据 `pid` 拿到对应进程的控制块，然后会将信号插入控制块中的要响应的信号队列 `signals` 中。`sigpromask` 也是类似的操作这里不再赘述。

值得一提的是，进程是在什么时候相应信号的呢？事实上，在我们的实现中，进程并没有立即响应信号，而是在下一次从内核态返回用户态之前来统一相应信号：
```rust
#[no_mangle]  
pub fn trap_handler() -> ! {
	set_kernel_trap_entry();  
	let scause = scause::read();  
	let stval = stval::read();  
	match scause.cause() {
		...
	}

	handle_signals();
	if let Some((errno, msg)) = check_signals_error_of_current() {  
	    println!("[kernel] {}", msg);  
	    exit_current_and_run_next(errno);  
	}  
	trap_return();
}
```

当处理完信号之后，我们会检查待响应信号队列 `signals` 中是否还存在一些致命的信号，如果存在，则结束该进程。

现在我们来看以下内核是如何处理信号的：

```rust
pub fn handle_signals() {  
    check_pending_signals();  
    loop {  
        let task = current_task().unwrap();  
        let task_inner = task.inner_exclusive_access();  
        let frozen_flag = task_inner.frozen;  
        let killed_flag = task_inner.killed;  
        drop(task_inner);  
        drop(task);  
        if (!frozen_flag) || killed_flag {  
            break;  
        }        check_pending_signals();  
        suspend_current_and_run_next()  
    }
}

fn check_pending_signals() {
    for sig in 0..(MAX_SIG + 1) {
        let task = current_task().unwrap();
        let task_inner = task.inner_exclusive_access();
        let signal = SignalFlags::from_bits(1 << sig).unwrap();
        if task_inner.signals.contains(signal) && (!task_inner.signal_mask.contains(signal)) {
            if task_inner.handling_sig == -1 {
                drop(task_inner);
                drop(task);
                if signal == SignalFlags::SIGKILL
                    || signal == SignalFlags::SIGSTOP
                    || signal == SignalFlags::SIGCONT
                    || signal == SignalFlags::SIGDEF
                {
                    // signal is a kernel signal
                    call_kernel_signal_handler(signal);
                } else {
                    // signal is a user signal
                    call_user_signal_handler(sig, signal);
                    return;
                }
            } else {
                if !task_inner.signal_actions.table[task_inner.handling_sig as usize]
                    .mask
                    .contains(signal)
                {
                    drop(task_inner);
                    drop(task);
                    if signal == SignalFlags::SIGKILL
                        || signal == SignalFlags::SIGSTOP
                        || signal == SignalFlags::SIGCONT
                        || signal == SignalFlags::SIGDEF
                    {
                        // signal is a kernel signal
                        call_kernel_signal_handler(signal);
                    } else {
                        // signal is a user signal
                        call_user_signal_handler(sig, signal);
                        return;
                    }
                }
            }
        }
    }
}
```
`handle_signals` 函数会调用 `check_pending_signals` 函数来检查发送给该进程的信号。信号分为必须由内核处理的信号和可由用户进程处理的信号两类。内核处理的信号有
-   `SIGSTOP` : 暂停该进程
-   `SIGCONT` : 继续该进程
-   `SIGKILL` : 杀死该进程
-   `SIGDEF` : 缺省行为：杀死该进程
主要由 `call_kernel_signal_handler` 函数完成，如果是 `SIGKILL` 或 `SIGDEF` 信号，该函数，会把进程控制块中的 `killed` 设置位 `true`。

而其它信号都属于可由用户进程处理的信号，由 `call_user_signal_handler` 函数进行进一步处理。
```rust
fn call_user_signal_handler(sig: usize, signal: SignalFlags) {  
    let task = current_task().unwrap();  
    let mut task_inner = task.inner_exclusive_access();  
  
    let handler = task_inner.signal_actions.table[sig].handler;  
    if handler != 0 {  
        // user handler  
  
        // change current mask        task_inner.signal_mask = task_inner.signal_actions.table[sig].mask;  
        // handle flag  
        task_inner.handling_sig = sig as isize;  
        task_inner.signals ^= signal;  
  
        // backup trapframe  
        let mut trap_ctx = task_inner.get_trap_cx();  
        task_inner.trap_ctx_backup = Some(*trap_ctx);  
  
        // modify trapframe  
        trap_ctx.sepc = handler;  
  
        // put args (a0)  
        trap_ctx.x[10] = sig;  
    } else {  
        // default action  
        println!("[K] task/call_user_signal_handler: default action: ignore it or kill process");  
    }
}
```
从 `call_user_signal_handler` 的实现可以看到，我们先把信号清掉，把进程之前的``trap``上下文保存在进程控制块的 `trap_ctx_backup` 中，然后把 `trap` 上下文的返回地址设置为注册的 `handler`，`a0` 寄存器（该函数的第一个参数）设置为 `sig` 。

#### sigreturn
```rust
pub fn sys_sigretrun() -> isize {  
    if let Some(task) = current_task() {  
        let mut inner = task.inner_exclusive_access();  
        inner.handling_sig = -1;  
        // restore the trap context  
        let trap_ctx = inner.get_trap_cx();  
        *trap_ctx = inner.trap_ctx_backup.unwrap();  
        0  
    } else {  
        -1  
    }  
}
```
将 `trap` 上下文更新为被信号打断前的状态，返回用户态继续执行。

## IO重定向
为了进一步增强shell程序使用文件系统时的灵活性，我们需要新增标准输入输出重定向功能。在shell中通过 `>` 我们可以将应用 的输出重定向到其他文件中。同理，通过 `<` 则可以将一个应用的输入重定向到某个指定文件而不是从键盘输入。

为了实现重定向功能，我们需要引入一个新的系统调用 `sys_dup` ：
```rust
pub fn sys_dup(fd: usize) -> isize {  
    let task = current_task().unwrap();  
    let mut inner = task.inner_exclusive_access();  
    if fd >= inner.fd_table.len() {  
        return -1;  
    }    
    if inner.fd_table[fd].is_none() {  
        return -1;  
    }    
    let new_fd = inner.alloc_fd();  
    inner.fd_table[new_fd] = Some(Arc::clone(inner.fd_table[fd].as_ref().unwrap()));  
    new_fd as isize  
}
```

该系统调用的功能很简单，将传入的文件描述符 `fd` 所对应的 `dyn File` 复制一份到新的 `fd` 上，如何用它来实现重定向呢？
```rust
let input_fd = open(input.as_str(), OpenFlags::RDONLY);  
if input_fd == -1 {  
    println!("Error when opening file {}", input);  
    return -4;  
}  
let input_fd = input_fd as usize;  
close(0);  
assert_eq!(dup(input_fd), 0);  
close(input_fd);
```

我们先使用系统调用 `open` 打开输入文件，获得输入文件的 `input_fd` ，然后关闭标准输入，调用 `dup` ，在处理 `dup` 过程中，由于标准输入刚刚关闭，此时在内部的 `fd` 分配器中 0 号位置空缺，因此就将复制的 `input_file` 存到 0号 `fd` 处然后返回，最后关闭 `input_fd` ，这个时候标准输入 0 号 `fd` 对应的 `dyn File` 就是我们刚刚打开的文件了，因此从标准输入读入的数据就会改为从文件中读入。

标准输出也同理。