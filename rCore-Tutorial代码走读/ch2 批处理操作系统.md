# rCore代码笔记-ch2
[[2022-05-21]]


## 批处理系统
我们要实现的批处理系统是比较古老的操作系统模型，它将多个用户程序加载到内存中，并依次执行，在本章中，我们要涉及到用户程序的编写和操作系统的编写，以及用户程序和操作系统的交互、CPU模式的转换等。

参考资料：[第二章：批处理系统 — rCore-Tutorial-Book-v3 3.6.0-alpha.1 文档 (rcore-os.github.io)](https://rcore-os.github.io/rCore-Tutorial-Book-v3/chapter2/)

## 初始化Trap机制

我们先把眼光聚焦到批处理系统的代码上来，用户程序请求操作系统的服务，是通过Trap的方式，在U态调用 ecall 指令，把参数通过寄存器传输。处理器在U态执行到 ecall 指令后，会通过 stvec 寄存器找到所有traps处理程序的入口，然后根据trap的上下文调用不同的trap服务程序。因此我们在系统启动时要对 stvec 寄存器进行赋值，注册所有traps的处理入口。

```rust
// os/src/main.rs

#[no_mangle]  
pub fn rust_main() -> ! {  
    clear_bss();  
    println!("[kernel] Hello, world!");  
    trap::init();  
    ...  
}  
  
// os/trap/mod.rs  
  
global_asm!(include_str!("trap.S"));  
/// initialize CSR `stvec` as the entry of `__alltraps`  
pub fn init() {  
    extern "C" {  
        fn __alltraps();  
    }  
    unsafe {  
        stvec::write(__alltraps as usize, TrapMode::Direct);  
    }  
}  
  
// os/trap/context.rs  
pub struct TrapContext {  
    /// general regs[0..31]  
    pub x: [usize; 32],  
    /// CSR sstatus        
    pub sstatus: Sstatus,  
    /// CSR sepc  
    pub sepc: usize,  
}
```

从init函数可以，我们在trap.S中定义了一个__alltraps作为traps的处理入口。在__alltraps构造Trap的上下文，填充一个TrapContext 结构体。因此我们来详细看一下这个__alltraps：

```mipsasm
# os/src/trap/trap.S

.macro SAVE_GP n
    sd x\n, \n*8(sp)
.endm

.align 2
__alltraps:
    csrrw sp, sscratch, sp
    # now sp->kernel stack, sscratch->user stack
    # allocate a TrapContext on kernel stack
    addi sp, sp, -34*8
    # save general-purpose registers
    sd x1, 1*8(sp)
    # skip sp(x2), we will save it later
    sd x3, 3*8(sp)
    # skip tp(x4), application does not use it
    # save x5~x31
    .set n, 5
    .rept 27
        SAVE_GP %n
        .set n, n+1
    .endr
    # we can use t0/t1/t2 freely, because they were saved on kernel stack
    csrr t0, sstatus
    csrr t1, sepc
    sd t0, 32*8(sp)
    sd t1, 33*8(sp)
    # read user stack from sscratch and save it on the kernel stack
    csrr t2, sscratch
    sd t2, 2*8(sp)
    # set input argument of trap_handler(cx: &mut TrapContext)
    mv a0, sp
    call trap_handler
```

-   第 7 行我们使用 .align 将 \_\_alltraps 的地址 4 字节对齐，这是 RISC-V 特权级规范的要求；   
 
-   第 9 行的 csrrw 原型是 csrrw rd, csr, rs，可以将 CSR 当前的值读到通用寄存器 rd 中，然后将通用寄存器 rs 的值写入该 CSR 。因此这里起到的是交换 sscratch 和 sp 的效果。在这一行之前 sp 指向用户栈，sscratch 指向内核栈（这是硬件实现的），现在 sp 指向内核栈， sscratch 指向用户栈。
    
-   第 12 行，我们准备在内核栈上保存 Trap 上下文，于是预先分配 34×8字节的栈帧，为什么是34？可以看一下结构体TrapContext的内存大小。这里改动的是 sp ，sp之前在第9行已经指向内核栈了，说明TrapContext确实是保存在内核栈上。
    
-   第 13~24 行，保存 Trap 上下文的通用寄存器 x0~x31，跳过 x0 （始终为0）和 tp(x4)（线程寄存器一般不会被用到）。我们在这里也不保存 sp(x2)，因为我们要基于它来找到每个寄存器应该被保存到的正确的位置。实际上，在栈帧分配之后，我们可用于保存 Trap 上下文的地址区间为 [sp,sp+8×34) ，按照 TrapContext 结构体的内存布局，基于内核栈的位置（sp所指地址）来从低地址到高地址分别按顺序放置 x0~x31这些通用寄存器，最后是 sstatus 和 sepc 。因此通用寄存器 xn 应该被保存在地址区间 [sp+8n,sp+8(n+1))[sp+8n,sp+8(n+1)) 。为了简化代码，x5~x31 这 27 个通用寄存器我们通过类似循环的 .rept 每次使用 SAVE_GP 宏来保存，其实质是相同的。注意我们需要在 trap.S 开头加上 .altmacro 才能正常使用 .rept 命令。
    
-   第 25~28 行，我们将 CSR sstatus 和 sepc 的值（当从用户态陷入S态时，硬件会自动将Trap 处理完成后默认会执行的下一条指令的地址填入sepc 寄存器，sstatus的SPP字段会被修改为 CPU 当前的特权级）分别读到寄存器 t0 和 t1 中然后保存到内核栈对应的位置上。指令 csrr rd, csr的功能就是将 CSR 的值读到寄存器 rd中。这里我们不用担心 t0 和 t1 被覆盖，因为它们刚刚已经被保存了。
    
-   第 30~31 行专门处理 sp 的问题。首先将 sscratch 的值读到寄存器 t2 并保存到内核栈上，注意： sscratch 的值是进入 Trap 之前的 sp 的值，指向用户栈。而现在的 sp 则指向内核栈。
    
-   第 33 行令 a0←sp，让寄存器 a0 指向内核栈的栈指针也就是我们刚刚保存的 Trap 上下文的地址，这是由于我们接下来要调用 trap_handler 进行 Trap 处理，它的第一个参数 cx 由调用规范要从 a0 中获取。而 Trap 处理函数 trap_handler 需要 Trap 上下文的原因在于：它需要知道其中某些寄存器的值，比如在系统调用的时候应用程序传过来的 syscall ID 和对应参数。我们不能直接使用这些寄存器现在的值，因为它们可能已经被修改了，因此要去内核栈上找已经被保存下来的值。

然后恢复上下文的代码如下：

```riscvasm
# os/src/trap/trap.S
.macro LOAD_GP n
    ld x\n, \n*8(sp)
.endm

__restore:
    # case1: start running app by __restore
    # case2: back to U after handling trap
    mv sp, a0
    # now sp->kernel stack(after allocated), sscratch->user stack
    # restore sstatus/pc
    ld t0, 32*8(sp)
    ld t1, 33*8(sp)
    ld t2, 2*8(sp)
    csrw sstatus, t0
    csrw sepc, t1
    csrw sscratch, t2
    # restore general-purpuse registers except sp/tp
    ld x1, 1*8(sp)
    ld x3, 3*8(sp)
    .set n, 5
    .rept 27
        LOAD_GP %n
        .set n, n+1
    .endr
    # release TrapContext on kernel stack
    addi sp, sp, 34*8
    # now sp->kernel stack, sscratch->user stack
    csrrw sp, sscratch, sp
    sret
```

-   第 10 行比较奇怪我们暂且不管，假设它从未发生，那么 sp 仍然指向内核栈的栈顶。
    
-   第 13~26 行负责从内核栈顶的 Trap 上下文恢复通用寄存器和 CSR 。注意我们要先恢复 CSR 再恢复通用寄存器，这样我们使用的三个临时寄存器才能被正确恢复。
    
-   在第 28 行之前，sp 指向保存了 Trap 上下文之后的内核栈栈顶， sscratch 指向用户栈栈顶。我们在第 28 行在内核栈上回收 Trap 上下文所占用的内存，回归进入 Trap 之前的内核栈栈顶。第 30 行，再次交换 sscratch 和 sp，现在 sp 重新指向用户栈栈顶，sscratch 也依然保存进入 Trap 之前的状态并指向内核栈栈顶。
    
-   在应用程序控制流状态被还原之后，第 31 行我们使用 sret 指令回到 U 特权级继续运行应用程序控制流。

然后CPU的控制权交给 trap_handler ：

```rust
// os/src/trap/mod.rs
  
#[no_mangle]  
/// handle an interrupt, exception, or system call from user space  
pub fn trap_handler(cx: &mut TrapContext) -> &mut TrapContext {  
    let scause = scause::read(); // get trap cause  
    let stval = stval::read(); // get extra value  
    match scause.cause() {  
        Trap::Exception(Exception::UserEnvCall) => {  
            cx.sepc += 4;  
            cx.x[10] = syscall(cx.x[17], [cx.x[10], cx.x[11], cx.x[12]]) as usize;  
        }  
        Trap::Exception(Exception::StoreFault) | Trap::Exception(Exception::StorePageFault) => {  
            println!("[kernel] PageFault in application, kernel killed it.");  
            run_next_app();  
        }  
        Trap::Exception(Exception::IllegalInstruction) => {  
            println!("[kernel] IllegalInstruction in application, kernel killed it.");  
            run_next_app();  
        }  
        _ => {  
            panic!(  
                "Unsupported trap {:?}, stval = {:#x}!",  
                scause.cause(),  
                stval  
            );  
        }  
    }  
    cx  
}
```

从a0寄存器获取得到的sp指针作为trap_handler的入参，然后将sp指针指向的那一片内存重新解释为一个TrapContext结构体。然后根据scause寄存器的值（在发生Trap时由硬件自动填充）选择对应的处理策略。

-   对于系统调用，spec加4，指向系统调用完成后的下一条指令，方便后续回复上下文的时候对spec寄存器赋值。然后根据TrapContext保存的x17、x10、x11、x12作为参数来调用服务。
    
-   对于存储错误和非法的指令的处理，仅仅只是通过第一章实现的println!宏打印出相关信息后就加载下一个用户程序了。

## 系统调用
系统调用有专门的系统调用号来表示我们申请的服务种类，而系统调用号和相应的调用参数规则需要用户程序和操作系统事先约定号，我们此处采用的是Linux标准的系统调用规则。

```rust
// os/src/syscall/mod.rs

const SYSCALL_WRITE: usize = 64;
const SYSCALL_EXIT: usize = 93;
/// handle syscall exception with `syscall_id` and other arguments
pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize {
    match syscall_id {
        SYSCALL_WRITE => sys_write(args[0], args[1] as *const u8, args[2]),
        SYSCALL_EXIT => sys_exit(args[0] as i32),
        _ => panic!("Unsupported syscall_id: {}", syscall_id),
    }
}
```

可以看出我们通过第一个参数，也就是TrapContext中的x17去匹配不同的服务函数。这些函数在我们第一章已经实现过，这里不再赘述。

## 加载用户程序
初始化Trap的相关数据结构后，我们需要加载用户程序，这个流程总的分两步，第一步，先把所有的用户程序镜像加载到数据段的某一片内存中，然后开始一个接一个地把每一个用户程序加载到某一片内存中并执行它。我们先看把所有应用程序镜像加载到数据段的代码：

```rust
// os/src/batch.rs
pub fn init() {
    print_app_info();
}

/// print apps info
pub fn print_app_info() {
    APP_MANAGER.exclusive_access().print_app_info();
}

struct AppManager {
    num_app: usize,
    current_app: usize,
    app_start: [usize; MAX_APP_NUM + 1],
}

lazy_static! {
    static ref APP_MANAGER: UPSafeCell<AppManager> = unsafe {
        UPSafeCell::new({
            extern "C" {
                fn _num_app();
            }
            let num_app_ptr = _num_app as usize as *const usize;
            let num_app = num_app_ptr.read_volatile();
            let mut app_start: [usize; MAX_APP_NUM + 1] = [0; MAX_APP_NUM + 1];
            let app_start_raw: &[usize] =
                core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1);
            app_start[..=num_app].copy_from_slice(app_start_raw);
            AppManager {
                num_app,
                current_app: 0,
                app_start,
            }
        })
    };
}
```

在这里我们不过多纠结rust的static mut变量的使用方法，我们只关心内核的初始化逻辑。可以看出我们声明了一个全局静态的AppManager结构体，里面存放了最重要的所有应用程序镜像在数据段的起始地址。应用程序镜像数据是通过build.rs文件写到link_app.S这个文件中的，我们可以看一下这个文件：

```asmasm
    .align 3
    .section .data
    .global _num_app
_num_app:
    .quad 5
    .quad app_0_start
    .quad app_1_start
    .quad app_2_start
    .quad app_3_start
    .quad app_4_start
    .quad app_4_end

    .section .data
    .global app_0_start
    .global app_0_end
app_0_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/00hello_world.bin"
app_0_end:

    .section .data
    .global app_1_start
    .global app_1_end
app_1_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/01store_fault.bin"
app_1_end:

    .section .data
    .global app_2_start
    .global app_2_end
app_2_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/02power.bin"
app_2_end:

    .section .data
    .global app_3_start
    .global app_3_end
app_3_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/03priv_inst.bin"
app_3_end:

    .section .data
    .global app_4_start
    .global app_4_end
app_4_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/04priv_csr.bin"
app_4_end:
```

这个汇编文件用.data标记将所有的用户镜像代码放到数据段，然后用链接脚本将该文件和内核链接起来，linker_app.S里的二进制流作为os的数据段依次序存到了0x8020xxxx，然后需要加载的时候在把相应的数据段里的指令拷贝到某一个位置再来执行。同时这个文件还声明了各个镜像的起始和终止地址以方便AppManager管理。

然后我们看一下如何执行用户程序：

```rust
// os/src/batch.rs

pub fn run_next_app() -> ! {
    let mut app_manager = APP_MANAGER.exclusive_access();
    let current_app = app_manager.get_current_app();
    unsafe {
        app_manager.load_app(current_app);
    }
    app_manager.move_to_next_app();
    drop(app_manager);
    // before this we have to drop local variables related to resources manually
    // and release the resources
    extern "C" {
        fn __restore(cx_addr: usize);
    }
    unsafe {
        __restore(KERNEL_STACK.push_context(TrapContext::app_init_context(
            APP_BASE_ADDRESS,
            USER_STACK.get_sp(),
        )) as *const _ as usize);
    }
    panic!("Unreachable in batch::run_current_app!");
}

impl AppManager {
    unsafe fn load_app(&self, app_id: usize) {
        if app_id >= self.num_app {
            panic!("All applications completed!");
        }
        println!("[kernel] Loading app_{}", app_id);
        // clear icache
        asm!("fence.i");
        // clear app area
        core::slice::from_raw_parts_mut(APP_BASE_ADDRESS as *mut u8, APP_SIZE_LIMIT).fill(0);
        let app_src = core::slice::from_raw_parts(
            self.app_start[app_id] as *const u8,
            self.app_start[app_id + 1] - self.app_start[app_id],
        );
        let app_dst = core::slice::from_raw_parts_mut(APP_BASE_ADDRESS as *mut u8, app_src.len());
        app_dst.copy_from_slice(app_src);
    }
}
```

我们先看run_next_app函数的整体流程，它先用AppManager加载了当前的用户程序到某个地方，然后先是使用app_init_context函数构造一个用户态的TrapContext结构体（sepc指向应用程序入口点[0x80400000中]， sp指向用户栈）返回，然后把这个TrapContext压入内核栈的栈顶，然后调用__restore方法的时候就会把我们构造的Context进行恢复，最后调用sret进入到用户态，spec中的值赋值给PC开始执行用户程序。

我们仔细看一下load_app中所作的事情，实际上就是把AppManager存放的当前用户程序镜像的地址指向的一篇内存拷贝到APP_BASE_ADDRESS = [0x80400000中]，在拷贝之前先对目标内存清0。

## 编写用户程序
我们如何在用户程序中使用系统调用，答案是ecall指令，当CPU处于U态时，调用ecall指令会发起系统调用请求，因此，我们在U态向S态发起一个写文件的请求如下：

```rust
// user/src/syscall.rs

const SYSCALL_WRITE: usize = 64;
const SYSCALL_EXIT: usize = 93;

fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret: isize;
    unsafe {
        asm!(
            "ecall",
            inlateout("x10") args[0] => ret,
            in("x11") args[1],
            in("x12") args[2],
            in("x17") id
        );
    }
    ret
}

pub fn sys_write(fd: usize, buffer: &[u8]) -> isize {
    syscall(SYSCALL_WRITE, [fd, buffer.as_ptr() as usize, buffer.len()])
}
```

值得注意的是，这里定义的两个服务号需要和内核定义的一致。我们进一步封装以实现用户态自己的println!宏，方法和第一张类似，需要区分的是，第一章是向SBI发起请求，而第二章这里则是向我们自己实现的操作系统内核发起请求。

```rust
pub fn write(fd: usize, buf: &[u8]) -> isize {
    sys_write(fd, buf)
}
pub fn exit(exit_code: i32) -> isize {
    sys_exit(exit_code)
}


struct Stdout;

const STDOUT: usize = 1;

impl Write for Stdout {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        write(STDOUT, s.as_bytes());
        Ok(())
    }
}

pub fn print(args: fmt::Arguments) {
    Stdout.write_fmt(args).unwrap();
}

#[macro_export]
macro_rules! print {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!($fmt $(, $($arg)+)?));
    }
}

#[macro_export]
macro_rules! println {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
    }
}
```

到此为止，批处理系统就完成了。