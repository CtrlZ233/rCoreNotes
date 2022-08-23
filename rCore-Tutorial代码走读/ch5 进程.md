# rCore代码笔记-ch5
[[2022-08-12]]

到目前为止，我们对每一个应用程序和内核都构造了一个地址空间，同时各个应用程序可以在操作系统的管理下有序地执行，但就人机交互性而言，我们当前的系统几乎为零。应用程序的编写者只能编写一些简单的计算程序，然后听天由命地等待操作系统进行调度。

回想一下我们曾经用过的linux系统或Windows系统等具有高交互性的系统，都提供了一个人机交互界面（shell），能让用户在一定程度上影响程序运行调度。shell究其本质也只是一个用户程序，它可以根据我们的输入创建其他任务提供给操作系统进行调度，而在其他任务运行的时候，shell自身也不会中止，而是接受创建任务的返回值（或者等待创建其他任务）。本章就要使得内核为用户程序提供创建其他用户程序的能力，以支持用户程序并发编程和人机交互性的需求。

参考资料：[第五章：进程 — rCore-Tutorial-Book-v3 3.6.0-alpha.1 文档 (rcore-os.github.io)](http://rcore-os.cn/rCore-Tutorial-Book-v3/chapter5/)

## 进程相关的系统调用
进程的概念不必细说，其实在此之前我们已经实现了进程的基本数据结构，之前章节的Task本质上就是进程，可以看出，进程在内核看来就是一个任务，是可以程序的执行过程，本章需要为用户程序提供几个系统调用，以实现在Task中再创建新的Task。在创建的同时，我们内核会给每个Task分配一个唯一的编号（PID）。

- fork：创建一个和当前完全相同的Task，父进程返回子进程PID，子进程返回0。
- wait_pid：父进程通过调用该系统调用来等待子进程结束并回收部分资源。
- exec：创建一个新的Task替换掉当前的Task（PID不变，但地址空间、数据代码都是新的程序）。

值得一提的是，我们需要有一个初始进程，在操作系统高刚起的时候就被加载到任务队列中，并且fork出一个shell用户程序来等待用户输入。

## 代码走读
老规矩从main开始看：
```rust
// src/main.rs

pub fn rust_main() -> ! {
	...
	task::add_initproc();
	...
	task::run_tasks();

}
```

主要关注这两个函数。
add_initproc 函数将名为initproc的用户程序加载运行。以下是add_initproc的代码：

```rust
fn main() -> i32 {
    if fork() == 0 {
        exec("user_shell\0");
    } else {
        loop {
            let mut exit_code: i32 = 0;
            let pid = wait(&mut exit_code);
            if pid == -1 {
                yield_();
                continue;
            }
            println!(
                "[initproc] Released a zombie process, pid={}, exit_code={}",
                pid, exit_code,
            );
        }
    }
    0
}

```

它首先调用fork系统调用创建了一个进程，然后根据返回值来判断父进程和子进程，对于子进程（返回值为0），调用exec系统调用执行user_shell程序等待用户输入；对于父进程，调用wait系统调用，等待所有挂载在init_proc的子进程退出并回收资源。


### fork
在内核态对fork的处理，会创建一个新的task，并把新的task的trap_cx存储的a0寄存器（系统调用返回值）置为0，并把新进程的PID作为函数返回值返回。

```rust
// src/syscall/process.rs
pub fn sys_fork() -> isize {
    let current_task = current_task().unwrap();
    let new_task = current_task.fork();
    let new_pid = new_task.pid.0;
    // modify trap context of new_task, because it returns immediately after switching
    let trap_cx = new_task.inner_exclusive_access().get_trap_cx();
    // we do not have to move to next instruction since we have done it before
    // for child process, fork returns 0
    trap_cx.x[10] = 0;
    // add new task to scheduler
    add_task(new_task);
    new_pid as isize
}
```

```rust
// src/task/task.rs
pub struct TaskControlBlock {  
    // immutable  
    pub pid: PidHandle,  
    pub kernel_stack: KernelStack,  
    // mutable  
    inner: UPSafeCell<TaskControlBlockInner>,  
}  
  
pub struct TaskControlBlockInner {  
    pub trap_cx_ppn: PhysPageNum,  
    pub base_size: usize,  
    pub task_cx: TaskContext,  
    pub task_status: TaskStatus,  
    pub memory_set: MemorySet,  
    pub parent: Option<Weak<TaskControlBlock>>,  
    pub children: Vec<Arc<TaskControlBlock>>,  
    pub exit_code: i32,  
}
```

```rust
// src/task/task.rs
pub fn fork(self: &Arc<Self>) -> Arc<Self> {
	// ---- access parent PCB exclusively
	let mut parent_inner = self.inner_exclusive_access();
	// copy user space(include trap context)
	let memory_set = MemorySet::from_existed_user(&parent_inner.memory_set);
	let trap_cx_ppn = memory_set
		.translate(VirtAddr::from(TRAP_CONTEXT).into())
		.unwrap()
		.ppn();
	// alloc a pid and a kernel stack in kernel space
	let pid_handle = pid_alloc();
	let kernel_stack = KernelStack::new(&pid_handle);
	let kernel_stack_top = kernel_stack.get_top();
	let task_control_block = Arc::new(TaskControlBlock {
		pid: pid_handle,
		kernel_stack,
		inner: unsafe {
			UPSafeCell::new(TaskControlBlockInner {
				trap_cx_ppn,
				base_size: parent_inner.base_size,
				task_cx: TaskContext::goto_trap_return(kernel_stack_top),
				task_status: TaskStatus::Ready,
				memory_set,
				parent: Some(Arc::downgrade(self)),
				children: Vec::new(),
				exit_code: 0,
			})
		},
	});
	// add child
	parent_inner.children.push(task_control_block.clone());
	// modify kernel_sp in trap_cx
	// **** access children PCB exclusively
	let trap_cx = task_control_block.inner_exclusive_access().get_trap_cx();
	trap_cx.kernel_sp = kernel_stack_top;
	// return
	task_control_block
	// ---- release parent PCB automatically
	// **** release children PCB automatically
}
```

task的复制实现挂载到了任务控制块上。任务控制块存储了当前进程的PID、内核栈、trap上下文、任务上下文、父子进程、地址空间等信息。因此在赋值Task控制块时，我们先复制了地址空间里的数据（包括各个代码段、数据段、用户栈、和Trap上下文），然后请求分配了一个新的PID，根据PID分配新的内核栈，最后把新进程控制块添加到当前进程的子进程队列中。

当调用 add_task(new_task) 后，新的任务控制块就进入调度队列中了，由于地址空间里的数据相同，trap上下文也相同，因此当调度到子进程时，Trap的返回地址也正好是用户程序完成fork系统调用的地址。

现在总结一下，调用fork后父子进程的相同和不同：
- 相同：代码段、数据段、bss段等、任务上下文、Trap上下文（除返回值）、用户栈内容。
- 不同：fork返回值、PID、父子进程关系不被继承、内核栈。

### exec
```rust
// src/syscall/process.rs
pub fn sys_exec(path: *const u8) -> isize {  
    let token = current_user_token();  
    let path = translated_str(token, path);  
    if let Some(data) = get_app_data_by_name(path.as_str()) {  
        let task = current_task().unwrap();  
        task.exec(data);  
        0  
    } else {  
        -1  
    }  
}
```
内核态处理exec系统调用时，会根据传入的path去找到用户程序对应的elf_data数据，调用挂载在任务控制块上的exec方法替换任务控制块中的内容。

```rust

// src/task/task.rs
pub fn exec(&self, elf_data: &[u8]) {
	// memory_set with elf program headers/trampoline/trap context/user stack
	let (memory_set, user_sp, entry_point) = MemorySet::from_elf(elf_data);
	let trap_cx_ppn = memory_set
		.translate(VirtAddr::from(TRAP_CONTEXT).into())
		.unwrap()
		.ppn();

	// **** access inner exclusively
	let mut inner = self.inner_exclusive_access();
	// substitute memory_set
	inner.memory_set = memory_set;
	// update trap_cx ppn
	inner.trap_cx_ppn = trap_cx_ppn;
	// initialize base_size
	inner.base_size = user_sp;
	// initialize trap_cx
	let trap_cx = inner.get_trap_cx();
	*trap_cx = TrapContext::app_init_context(
		entry_point,
		user_sp,
		KERNEL_SPACE.exclusive_access().token(),
		self.kernel_stack.get_top(),
		trap_handler as usize,
	);
	// **** release inner automatically
}
```

在exec中，我们会根据elf data重新生成地址空间，然后将同时替换当前任务控制块中的地址空间、Trap物理地址，并重新设置Trap上下文，将Trap上下文中的返回地址设置为新程序的入口地址，当从当前的内核态返回时，就会跳转到新的用户程序入口执行。

总结一下，执行exec后：
- 发生变化的：用户地址空间（代码段、数据段、bss段等）、用户栈、Trap上下文。
- 不发生变化的：PID、内核栈、父子进程的关系。

### wait
用户态提供了wait的两个接口：
```rust
// user/src/lib.rs 

pub fn wait(exit_code: &mut i32) -> isize {
    loop {
        match sys_waitpid(-1, exit_code as *mut _) {
            -2 => {
                yield_();
            }
            // -1 or a real pid
            exit_pid => return exit_pid,
        }
    }
}

pub fn waitpid(pid: usize, exit_code: &mut i32) -> isize {
    loop {
        match sys_waitpid(pid as isize, exit_code as *mut _) {
            -2 => {
                yield_();
            }
            // -1 or a real pid
            exit_pid => return exit_pid,
        }
    }
}

```

waitpid是等待特定的pid子进程返回，wait不指定pid则是等待任意一个子进程返回。对于wait而言，可以传入一个通配PID=-1来重用系统调用接口。

而内核态对系统调用的处理如下：

```rust

// os/src/syscall/process.rs
pub fn sys_waitpid(pid: isize, exit_code_ptr: *mut i32) -> isize {  
    let task = current_task().unwrap();  
    // find a child process  
  
    // ---- access current TCB exclusively    let mut inner = task.inner_exclusive_access();  
    if !inner  
        .children  
        .iter()  
        .any(|p| pid == -1 || pid as usize == p.getpid())  
    {        
	    return -1;  
        // ---- release current PCB  
    }  
    let pair = inner.children.iter().enumerate().find(|(_, p)| {  
        // ++++ temporarily access child PCB lock exclusively  
        p.inner_exclusive_access().is_zombie() && (pid == -1 || pid as usize == p.getpid())  
        // ++++ release child PCB  
    });  
    if let Some((idx, _)) = pair {  
        let child = inner.children.remove(idx);  
        // confirm that child will be deallocated after removing from children list  
        assert_eq!(Arc::strong_count(&child), 1);  
        let found_pid = child.getpid();  
        // ++++ temporarily access child TCB exclusively  
        let exit_code = child.inner_exclusive_access().exit_code;  
        // ++++ release child PCB  
        *translated_refmut(inner.memory_set.token(), exit_code_ptr) = exit_code;  
        found_pid as isize  
    } else {  
        -2  
    }  
    // ---- release current PCB lock automatically  
}
```

首先获取当前任务控制块并且遍历所有的子进程。
- 如果要等待的pid 不等于 -1（代表需要等待某一个特定PID进程）且没有子进程的PID等于传入的PID，代表要等待的PID已经不是当前任务的子进程了（输入非法），返回-1。
- 遍历所有子进程，如果子进程中存在僵尸进程（执行完毕），如果入参pid = -1（等待任意一个子进程执行完毕）或入参pid == 僵尸进程pid，对该进程进行回收。
- 回收过程中，只需要将子进程从children数组中移除，就会自动出发任务控制块中实现Drop Trait成员的Drop方法，回收部分资源（PID、Kernel Stack），而存储地址空间的物理页框则是在进程执行完成、变成僵尸进程之前就回收掉的，后面会进行介绍。

### 资源回收
当进程执行完毕，我们需要将之前分配给他的资源进行回收，当前章节所涉及到的主要是：地址空间中的物理页框、PID、内核栈、子进程。在此，我们分为两步进行回收：
```rust
pub fn exit_current_and_run_next(exit_code: i32) {
    // take from Processor
    let task = take_current_task().unwrap();
    // **** access current TCB exclusively
    let mut inner = task.inner_exclusive_access();
    // Change status to Zombie
    inner.task_status = TaskStatus::Zombie;
    // Record exit code
    inner.exit_code = exit_code;
    // do not move to its parent but under initproc

    // ++++++ access initproc TCB exclusively
    {
        let mut initproc_inner = INITPROC.inner_exclusive_access();
        for child in inner.children.iter() {
            child.inner_exclusive_access().parent = Some(Arc::downgrade(&INITPROC));
            initproc_inner.children.push(child.clone());
        }
    }
    // ++++++ release parent PCB

    inner.children.clear();
    // deallocate user space
    inner.memory_set.recycle_data_pages();
    drop(inner);
    // **** release current PCB
    // drop task manually to maintain rc correctly
    drop(task);
    // we do not have to save task context
    let mut _unused = TaskContext::zero_init();
    schedule(&mut _unused as *mut _);
}
```

当进程执行完毕，回调用exit系统调用，最后进入这个内核态中的函数进行处理，在函数中，它将即将结束的进程的所有子进程都挂载到init_proc下，然后将地址空间清空，在此会出发地址空间中存放的所有FrameTracker的Drop方法，将之前申请的物理内存全部归还。

第二步释放则是这个即将结束的进程的父进程wait它时进行释放，主要释放了PID和内核，上一节已经讲过。可能有人会觉得，不是所有的父进程都会等待子进程结束的，假如创建这个进程的父进程没有等待子进程结束，那么当父进程结束的时候，会把子进程挂载init_proc中，而init_proc则是一直在循环等待它的任意一个子进程结束，这时候init_proc就起到了一个资源回收兜底的作用。

### 进程调度
在本章中，任务的调度将处理器和任务进行了分离：
```rust
pub struct Processor {  
    ///The task currently executing on the current processor  
    current: Option<Arc<TaskControlBlock>>,  
    ///The basic control flow of each core, helping to select and switch process  
    idle_task_cx: TaskContext,  
}
```
Processor中保存了当前运行的任务控制块current和上一次调度的时候被调离的任务上下文idle_task_cx。

```rust
pub fn run_tasks() {
    loop {
        let mut processor = PROCESSOR.exclusive_access();
        if let Some(task) = fetch_task() {
            let idle_task_cx_ptr = processor.get_idle_task_cx_ptr();
            // access coming task TCB exclusively
            let mut task_inner = task.inner_exclusive_access();
            let next_task_cx_ptr = &task_inner.task_cx as *const TaskContext;
            task_inner.task_status = TaskStatus::Running;
            drop(task_inner);
            // release coming task TCB manually
            processor.current = Some(task);
            // release processor manually
            drop(processor);
            unsafe {
                __switch(idle_task_cx_ptr, next_task_cx_ptr);
            }
        }
    }
}
```

在操作系统初始化完成后会调用run_tasks函数，这时候的任务队列中应该只有init_proc一个进程，这个时候init_proc任务控制块中的任务上下文会被选中（此时返回地址被设置为trap_return)，调用__switch之后会把当前的任务上下文存到idle_task_cx_ptr，然后去执行被选中的任务。

```rust
pub fn schedule(switched_task_cx_ptr: *mut TaskContext) {
    let mut processor = PROCESSOR.exclusive_access();
    let idle_task_cx_ptr = processor.get_idle_task_cx_ptr();
    drop(processor);
    unsafe {
        __switch(switched_task_cx_ptr, idle_task_cx_ptr);
    }
}
```

然后每次任务调度的时候（时间片用完、主动yield、进程结束等等），都会调用__switch重新把之前保存在idle_task_cx_ptr的任务上下文切换回来执行，而上一次切换则是在run_tasks刚刚调用完__switch，因此调用完schedule中的__switch之后就返回到了run_tasks中的__switch，然后在run_tasks中选择一个可用的Task进行任务切换。本质上是一次任务切换调用了两次__switch。

> A_task_cx  --> idle_task_cx --> B_task_cx 

至此，拥有进程管理的操作系统内核就完成了。