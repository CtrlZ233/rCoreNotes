# rCore代码笔记-ch3
[[2022-05-21]]
```toc

```

今天继续开冲，本章主要介绍了多道程序操作系统和分时操作系统，他们都是可以将多个用户程序同时放入内存，根据一定的机制在不同的程序之间进行切换的操作系统，与之前的[[ch2 批处理操作系统]]相比，区别在于：
- 所有的用户程序都已经被加载到内存的不同位置。
- CPU可能在一个程序没有执行完的时候就暂停该程序去执行其他程序。

参考资料：[第三章：多道程序与分时多任务 — rCore-Tutorial-Book-v3 3.6.0-alpha.1 文档 (rcore-os.github.io)](https://rcore-os.github.io/rCore-Tutorial-Book-v3/chapter3/index.html#link-chapter3)
## 程序加载
要紧事就是先把所有的用户程序加载到内存中，因此我们需要事先约定好每个任务（用户程序）的内存起始地址、内存大小等。

和第二章一样，link_app.S 通过把用户代码内嵌到数据段来实现用户程序预加载，此外还标明了各个用户代码的起始存储地址（为了与后续的代码执行地址区分）和终止存储地址。然后通过 loader 模块中的 load_apps()来将代码段中的用户代码加载到内存的各个内存执行段中：

```rust
pub const APP_BASE_ADDRESS: usize = 0x80400000;
pub const APP_SIZE_LIMIT: usize = 0x20000;

// os/src/loader.rs

/// Get base address of app i.
fn get_base_i(app_id: usize) -> usize {
    APP_BASE_ADDRESS + app_id * APP_SIZE_LIMIT
}

/// Load nth user app at
/// [APP_BASE_ADDRESS + n * APP_SIZE_LIMIT, APP_BASE_ADDRESS + (n+1) * APP_SIZE_LIMIT).
pub fn load_apps() {
    extern "C" {
        fn _num_app();
    }
    let num_app_ptr = _num_app as usize as *const usize;
    let num_app = get_num_app();
    let app_start = unsafe { core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1) };
    // clear i-cache first
    unsafe {
        asm!("fence.i");
    }
    // load apps
    for i in 0..num_app {
        let base_i = get_base_i(i);
        // clear region
        (base_i..base_i + APP_SIZE_LIMIT)
            .for_each(|addr| unsafe { (addr as *mut u8).write_volatile(0) });
        // load app from data section to memory
        let src = unsafe {
            core::slice::from_raw_parts(app_start[i] as *const u8, app_start[i + 1] - app_start[i])
        };
        let dst = unsafe { core::slice::from_raw_parts_mut(base_i as *mut u8, src.len()) };
        dst.copy_from_slice(src);
    }
}
```

我们可以看出这个函数先清除了指令Cache的缓存（由于我们修改了指令内存的数据，之前的Cache已经失效，因此我们必须清除缓存），然后对每一个内存段清空，并把数据段中的用户程序代码拷贝到对应的内存段中，以实现用户程序的加载。

## 任务上下文（TaskContext）
可以想象，我们实现任务（用户程序）的切换，需要保存一些数据，以方便我们后续切换回来的时候继续执行，我们称之为任务上下文。这个上下文与Trap上下文类似，都是为了能够重新返回之前的任务继续执行。但我们需要区分任务切换和异常之前的不同，异常发生的时候，CPU处于用户程序的执行状态，因此我们需要保存很多东西：用户堆栈、所有寄存器的状态和返回地址。而任务切换发生时CPU处于内核态，由操作系统内核调用任务切换函数来进行任务切换，因此需要保存的仅仅是返回地址、内核堆栈以及需要rust函数调用规范中被调用者需要维护的寄存器的状态（s0——s11）：

```rust
pub struct TaskContext {
    /// return address ( e.g. __restore ) of __switch ASM function
    ra: usize,
    /// kernel stack pointer of app
    sp: usize,
    /// callee saved registers:  s 0..11
    s: [usize; 12],

}
```

>函数调用同样需要维护函数调用之前的寄存器信息，但是每种高级语言都有自己的调用规范，一部分寄存器是由函数调用者维护，另一部分则是由被调用者维护。rust中，寄存器s0-->s11由被调用者进行维护。正常情况下，这些维护都有rust编译器进行自动管理。

## 任务切换
假如我们拿到了任务切换流程的源TaskContext引用和目标TaskContext引用。如何进行任务切换呢？首先，我们需要把当前的任务上下文给存起来，然后，再根据目标TaskContext对目标任务的状态进行恢复。

```riscvasm
__switch:
    # __switch(
    #     current_task_cx_ptr: *mut TaskContext,
    #     next_task_cx_ptr: *const TaskContext
    # )
    # save kernel stack of current task
    sd sp, 8(a0)
    # save ra & s0~s11 of current execution
    sd ra, 0(a0)
    .set n, 0
    .rept 12
        SAVE_SN %n
        .set n, n + 1
    .endr
    # restore ra & s0~s11 of next execution
    ld ra, 0(a1)
    .set n, 0
    .rept 12
        LOAD_SN %n
        .set n, n + 1
    .endr
    # restore kernel stack of next task
    ld sp, 8(a1)
    ret
```

我们把这个函数暴露给rust程序，入参为一个源TaskContext的可变引用（保存在a0寄存器）和一个目标TaskContext的可变引用（保存在a1寄存器）。第7到14行将当前的ra寄存器、sp寄存器和s0--s11寄存器填充到源TaskContext中，第16到23行将根据TaskContext加载sp寄存器、ra寄存器、s0--s11寄存器。当该函数返回时，ra指向的返回地址时目标任务之前的切换地址（内核态），sp指向的是目标任务的内核态堆栈，s0--s11为目标任务上一次被切出时的寄存器状态。一旦ret被返回掉，整个系统的任务切换就完成了，参考图示如下：
![[Pasted image 20220521204946.png]]

## 任务管理器（TaskManager)
到目前为止，我们已经可以实现任务的切换，但是由于我们由多个任务都处在内存中，而他们又有着各自不同的状态，因此我们需要一个管理者将这些任务管理起来，当当前的任务被切出的时候，管理者选出一个可以立即运行的程序切入。
```rust
// os/src/task/mod.rs

pub struct TaskManager {
    /// total number of tasks
    num_app: usize,
    /// use inner value to get mutable access
    inner: UPSafeCell<TaskManagerInner>,
}

/// Inner of Task Manager
pub struct TaskManagerInner {
    /// task list
    tasks: [TaskControlBlock; MAX_APP_NUM],
    /// id of current `Running` task
    current_task: usize,
}

// os/src/task/task.rs
#[derive(Copy, Clone)]
pub struct TaskControlBlock {
    pub task_status: TaskStatus,
    pub task_cx: TaskContext,
}

#[derive(Copy, Clone, PartialEq)]
pub enum TaskStatus {
    UnInit,
    Ready,
    Running,
    Exited,
}
```

各个任务由任务控制块（TaskControlBlock）来标识状态和任务上下文。任务上下文在之前已经讲过，而任务的状态主要分为UnInit（未初始化）、Ready（就绪，但未得到CPU）、Runing（正在CPU上运行）、Exited（运行结束）。而TaskManager拥有所有的任务控制块以及当前正在运行的任务号。

TaskManager是一个单例，因此它在操作系统启动时就已经初始化好，在它的初始化过程中，它需要做以下几件事情：
- 初始化每个任务的内核堆栈和用户堆栈。
- 初始化每个任务的TrapContext（每个任务的启动需要通过中断返回的方式进入用户态）。
- 将TrapContext推入对应的内核堆栈（中断返回前调用__restore函数，从内核栈中取出刚刚初始化的TrapContext）。
- 初始化每个任务的上下文。
- 将每个任务的状态置为Ready。

```rust
// os/src/task/mod.rs

pub static ref TASK_MANAGER: TaskManager = {
	let num_app = get_num_app();
	let mut tasks = [TaskControlBlock {
		task_cx: TaskContext::zero_init(),
		task_status: TaskStatus::UnInit,
	}; MAX_APP_NUM];
	for (i, task) in tasks.iter_mut().enumerate() {
		task.task_cx = TaskContext::goto_restore(init_app_cx(i));
		task.task_status = TaskStatus::Ready;
	}
	TaskManager {
		num_app,
		inner: unsafe {
			UPSafeCell::new(TaskManagerInner {
				tasks,
				current_task: 0,
			})
		},
	}
};

// os/src/task/context.rs

impl TaskContext {
    /// init task context
    pub fn zero_init() -> Self {
        Self {
            ra: 0,
            sp: 0,
            s: [0; 12],
        }
    }

    /// set task context {__restore ASM funciton, kernel stack, s_0..12 }
    pub fn goto_restore(kstack_ptr: usize) -> Self {
        extern "C" {
            fn __restore();
        }
        Self {
            ra: __restore as usize,
            sp: kstack_ptr,
            s: [0; 12],
        }
    }
}
```

可以看出我们先将所有的TaskContext清零，并状态置为UnInit，然后遍历每个TaskContext，先通过 init_app 设置好每个任务的内核堆栈并把spec为各个用户程序的加载起始地址的TrapContext推入堆栈。然后调用 goto_restore 把TaskContext中的sp只为刚刚初始化过的内核栈， ra 置为__restore以实现：在__switch函数返回后立即返回用户态执行切入的任务。

此外，TaskManager还提供各种管理功能：
```rust
impl TaskManager {
    /// Run the first task in task list.
    ///
    /// Generally, the first task in task list is an idle task (we call it zero process later).
    /// But in ch3, we load apps statically, so the first task is a real app.
    fn run_first_task(&self) -> !;

    /// Change the status of current `Running` task into `Ready`.
    fn mark_current_suspended(&self);

    /// Change the status of current `Running` task into `Exited`.
    fn mark_current_exited(&self);

    /// Find next task to run and return app id.
    ///
    /// In this case, we only return the first `Ready` task in task list.
    fn find_next_task(&self) -> Option<usize>;
	
    /// Switch current `Running` task to the task we have found,
    /// or there is no `Ready` task and we can exit with all applications completed
    fn run_next_task(&self);
}
```



## 协作式调度
到目前为止，我们已经可以实现任务的切换，但是我们还需要确定任务切换的时机。我们先介绍一种看上去很美好的调度算法——协作式调度。

顾名思义，协作式调度要求用户程序要有协作互助的精神，在自己当前用不上CPU的时候主动进行切换（例如当前需要读写磁盘数据，或者等待某些事件发生）。切换任务是由用户程序主动发起，主动权在用户程序手中。

由于任务切换的对安全性的要求极高，因此真正的任务切换只能在受信任的操作系统内核中进行。而协作式调度又要求切换由用户程序发起，因此内核需要提供一个切换的系统调用给用户态。用户通过系统调用对内核请求任务切换的服务。

```rust
// user/src/syscall.rs

const SYSCALL_YIELD: usize = 124;

pub fn sys_yield() -> isize {
    syscall(SYSCALL_YIELD, [0, 0, 0])
}

// user/src/lib.rs
pub fn yield_() -> isize {
    sys_yield()
}

// user/src/bin/00write_a.rs
fn main() -> i32 {
    for i in 0..HEIGHT {
        for _ in 0..WIDTH {
            print!("A");
        }
        println!(" [{}/{}]", i + 1, HEIGHT);
        yield_();
    }
    println!("Test write_a OK!");
    0
}
```

在用户程序主动调用 yield_ 函数，进而调用第124号系统调用来请求任务切换。而在操作系统中也将处理第124号系统调用。

```rust
// os/src/syscall/mod.rs

const SYSCALL_YIELD: usize = 124;

/// handle syscall exception with `syscall_id` and other arguments
pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize {
    match syscall_id {
        SYSCALL_WRITE => sys_write(args[0], args[1] as *const u8, args[2]),
        SYSCALL_EXIT => sys_exit(args[0] as i32),
        SYSCALL_YIELD => sys_yield(),
        _ => panic!("Unsupported syscall_id: {}", syscall_id),
    }
}

// os/src/syscall/process.rs

/// current task gives up resources for other tasks
pub fn sys_yield() -> isize {
    suspend_current_and_run_next();
    0
}


// os/src/task/mod.rs

/// suspend current task, then run next task
pub fn suspend_current_and_run_next() {
    mark_current_suspended();
    run_next_task();
}
```

在 yield 系统调用处理中，我们先把当前正在运行的任务状态设置为Ready，然后选择下一个就绪的任务进行运行：

```rust
// os/src/task/mod.rs

/// Find next task to run and return app id.
///
/// In this case, we only return the first `Ready` task in task list.
fn find_next_task(&self) -> Option<usize> {
	let inner = self.inner.exclusive_access();
	let current = inner.current_task;
	(current + 1..current + self.num_app + 1)
		.map(|id| id % self.num_app)
		.find(|id| inner.tasks[*id].task_status == TaskStatus::Ready)
}

/// Switch current `Running` task to the task we have found,
/// or there is no `Ready` task and we can exit with all applications completed
fn run_next_task(&self) {
	if let Some(next) = self.find_next_task() {
		let mut inner = self.inner.exclusive_access();
		let current = inner.current_task;
		inner.tasks[next].task_status = TaskStatus::Running;
		inner.current_task = next;
		let current_task_cx_ptr = &mut inner.tasks[current].task_cx as *mut TaskContext;
		let next_task_cx_ptr = &inner.tasks[next].task_cx as *const TaskContext;
		drop(inner);
		// before this, we should drop local variables that must be dropped manually
		unsafe {
			__switch(current_task_cx_ptr, next_task_cx_ptr);
		}
		// go back to user mode
	} else {
		panic!("All applications completed!");
	}
}
```

我们会从当前的任务序号开始遍历所有的任务，找到第一个状态为Ready的任务，将它的状态设置为Running，把指向当前运行任务的指针指向该任务，然后调用__switch函数进行任务切换，当__switch函数返回时，内核栈、s0--s11均已经发生了切换，内核栈中存有该任务陷入内核态时的TrapContext，在 yield 系统调用完成后会根据这个TrapContext返回该任务的用户态执行用户程序。

这就是协同调度的基本实现。但是这种调度有着十分明显的缺陷：
-	我们把调度权交给用户程序的编写者，但用户程序并不都是如此谦让。
-	应用调用它主动交出 CPU 使用权之后，它下一次再被允许使用 CPU 的时间点与内核的调度策略与当前的总体应用执行情况有关，很有可能远远迟于该应用等待的事件（如外设处理完请求）达成的时间点。这就会造成该应用的响应延迟不稳定或者很长。比如，设想一下，敲击键盘之后隔了数分钟之后才能在屏幕上看到字符，这已经超出了人类所能忍受的范畴。

基于以上的缺点，抢占式调度和分时多任务系统应运而生。

## 抢占式调度

从上一节看来：如果应用自己很少 yield ，操作系统内核就要开始收回之前下放的权力，由它自己对 CPU 资源进行集中管理并合理分配给各应用，这就是内核需要提供的任务调度能力。我们可以将多道程序的调度机制分类成 
- 协作式调度 (Cooperative Scheduling) ：因为它的特征是：只要一个应用不主动 yield 交出 CPU 使用权，它就会一直执行下去。
- 抢占式调度 (Preemptive Scheduling) 则是应用随时都有被内核切换出去的可能。

本节使用时间片轮转法实现抢占式调度，我们给每个任务分配一个时间片，当该时间片执行完毕，由硬件发起一个时钟中断，内核接管CPU后选择一个其他Ready的程序分配给他一个新的时间片来切入执行。和上面的协同式调度的区别在于，我们引入了定时器的机制，定时器每个一段时间就将CPU交给内核来进行任务调度。

```rust
// os/src/main.rs
/// the rust entry-point of os
#[no_mangle]
pub fn rust_main() -> ! {
    ...
    trap::enable_timer_interrupt();
    timer::set_next_trigger();
    ...
    panic!("Unreachable in rust_main!");
}

// os/src/trap/mod.rs
/// timer interrupt enabled
pub fn enable_timer_interrupt() {
    unsafe {
        sie::set_stimer();
    }
}

// os/src/timer.rs
const TICKS_PER_SEC: usize = 100;
const MSEC_PER_SEC: usize = 1000;


/// read the `mtime` register
pub fn get_time() -> usize {
    time::read()
}

/// get current time in milliseconds
pub fn get_time_ms() -> usize {
    time::read() / (CLOCK_FREQ / MSEC_PER_SEC)
}

/// set the next timer interrupt
pub fn set_next_trigger() {
    set_timer(get_time() + CLOCK_FREQ / TICKS_PER_SEC);
}
```

在内核初始化时我们使能了时钟中断，然后设置了下一个时钟中断的发生时刻。这里的CLOCK_FREQ是一秒内时钟计数器的增量，除以TICKS_PER_SEC = 100就是 1/100s = 10ms 的时钟计数器的增量，这里合起来就是，当10ms后会出发一次时钟中断。

我们将这两个都设置好了之后就可以开始运行用户程序，当第一个用户程序运行10 ms后时钟中断触发了，CPU交给内核进行处理：

```rust
#[no_mangle]
// os/src/trap/mod.rs
/// handle an interrupt, exception, or system call from user space
pub fn trap_handler(cx: &mut TrapContext) -> &mut TrapContext {
    let scause = scause::read(); // get trap cause
    let stval = stval::read(); // get extra value
    match scause.cause() {
        ...
        Trap::Interrupt(Interrupt::SupervisorTimer) => {
            set_next_trigger();
            suspend_current_and_run_next();
        }
        _ => {
            panic!(
                "Unsupported trap {:?}, stval = {:#x}!",
                scause.cause(),
                stval
            );
        }
    }
    cx
}
```

在中断处理中，我们重新设置了下一个时钟中断的触发时间，然后暂停当前任务取运行下一个任务（这和上一节介绍的是完全一样的）。

为了实现我们在用户程序中的计时任务，内核需要提供一个系统调用，以返回当前系统的时间：

```rust
// user/src/syscall.rs
const SYSCALL_GET_TIME: usize = 169;

pub fn sys_get_time() -> isize {=
    syscall(SYSCALL_GET_TIME, [0, 0, 0])
}

// os/src/syscall/mod.rs
const SYSCALL_GET_TIME: usize = 169;

/// handle syscall exception with `syscall_id` and other arguments
pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize {
    match syscall_id {
        ...
        SYSCALL_GET_TIME => sys_get_time(),
        _ => panic!("Unsupported syscall_id: {}", syscall_id),
    }
}

// os/src/syscall/process.rs
/// get time in milliseconds
pub fn sys_get_time() -> isize {
    get_time_ms() as isize
}
```

在用户程序中就可以使用系统调用获取当前的系统时间来实现计时任务：
```rust
// user/src/bin/03sleep.rs
#[no_mangle]
fn main() -> i32 {
    let current_timer = get_time();
    let wait_for = current_timer + 3000;
    while get_time() < wait_for {
        yield_();
    }
    println!("Test sleep OK!");
    0
}
```

到此为止，一个分时多任务系统就基本完成了。