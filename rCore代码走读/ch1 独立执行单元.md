# rCore代码笔记-ch1
[[2022-05-21]]
```toc

```

五一节补一下实验报告，rust不是很熟悉（/(ㄒoㄒ)/~~），尽量从操作系统的角度理解来代码。

参考资料：[第一章：应用程序与基本执行环境 — rCore-Tutorial-Book-v3 3.6.0-alpha.1 文档 (rcore-os.github.io)](https://rcore-os.github.io/rCore-Tutorial-Book-v3/chapter1/)
## qemu-riscv的启动流程
1. 第一阶段：由固化在Qemu内的一小段汇编程序负责。将必要的文件加载到物理内存后，PC（程序计数器）会被设置为0x1000，因此Qemu实际运行的第一条指令位于物理地址[0x1000]，接下来执行几条指令后就跳转到[0x80000000]（硬编码固化到Qemu中）并进入第二阶段。
2. 第二阶段：由bootloader负责。我们需要将bootloader（该功能由RustSBI提供）放到以物理地址[0x80000000]开头的内存中，并跳转到该处执行。在这一阶段，bootloader负责对计算机进行一些初始化工作，并跳转到内核镜像加载的地址（该地址不固定，可能事先约定好，也可能是动态获取的）。我们选用的RustSBI则是将下一阶段的入口地址预先约定为[0x80200000]。
3. 由内核镜像负责。为了正确与第二层对接，我们需要保证内核的第一条指令位于物理地址[0x80200000]。为此我们需要将内核镜像预先加载到Qemu物理地址的[0x80200000]开头的区域中。

## rust编写移除标准库依赖的risc-v的可执行程序
需要清楚地认识到我们编写的是操作系统内核程序，它运行在几乎赤裸的计算机环境上，与我们平时的应用程序不同，我们将不会得到任何标准系统库的支持（因为此刻我们写的就是系统）。因此，我们需要先完成一个不需要任何标准系统库支持的rust程序。

移除标准库，我们需要使用#![no_std]标注来告诉编译器我们不使用标准库。在移除标准库后，Rust语言迫切需要一个Panic处理函数来对运行时的Panic进行处理（这本来应该是标准库所做的事情，但是我们把它干掉了）。因此我们自己用#![panic_handle]标注编写自己的Panic处理函数：
```rust
// src/lang_items.rs
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    if let Some(location) = info.location() {
        println!(
            "[kernel] Panicked at {}:{} {}",
            location.file(),
            location.line(),
            info.message().unwrap()
        );
    } else {
        println!("[kernel] Panicked: {}", info.message().unwrap());
    }
    shutdown()

}
```

这里有同学会说 println! 函数也是标准库里的宏，别急，我们后面会自己实现一个功能类似的 println! 宏，这里只是意思一下。

类似的，移除标准库后还有一个影响，语言标准库和三方库作为应用程序的执行环境，需要负责在执行应用程序之前进行一些初始化工作，然后才跳转到应用程序的入口点（也就是跳转到我们编写的main函数）开始执行。事实上start语义项代表了标准库 std 在执行应用程序之前需要进行的一些初始化工作。由于我们禁用了标准库，编译器也就找不到这项功能的实现了。因此我们有两个方法——重写 start 语义项和删除 main 函数。其实在我的理解来看，只有一个选项，那就是重写start，然后在start执行完毕后使用一个跳转指令进入真正的main函数，这个时候的main函数你可以随便给他起个啥名（ctrlz233_main都可以）。回顾上一节的risc-v启动流程，start对应的是内核的第一条指令，它将放到0x80200000的物理地址。

```riscvasm
.section .text.entry
    .globl _start
_start:
    la sp, boot_stack_top
    call rust_main

    .section .bss.stack
    .globl boot_stack
boot_stack:
    .space 4096 * 16
    .globl boot_stack_top
boot_stack_top:
```

\_start 这个被声明为全局的标识符就是我们需要重写的，在_start中我们将开辟好的、由boot_stack_top指向栈顶的栈内存赋值给sp寄存器，然后调用rust_main函数。

> la指令是一条为指令，为load address的缩写，相当于auipc和addi两条指令一起执行的效果。

而我们的rust_main函数则是在main.rs文件中用高级编程语言rust实现，当然，我们需要在main.rs中包含如下代码以保证被我们重写的_start能被编译器正确找到解析并链接。

```rust
global_asm!(include_str!("entry.asm"));
```

由于我们编写的是需要在裸机上运行的镜像文件，因此我们最好能够对我们编译出来的各个段（section）的分布了如指掌，因此这里需要用到链接脚本（链接脚本的语法不过多赘述，需要自己简单学习）：

```link
OUTPUT_ARCH(riscv)
ENTRY(_start)
BASE_ADDRESS = 0x80200000;

SECTIONS
{
    . = BASE_ADDRESS;
    skernel = .;

    stext = .;
    .text : {
        *(.text.entry)
        *(.text .text.*)
    }

    . = ALIGN(4K);
    etext = .;
    srodata = .;
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }

    . = ALIGN(4K);
    erodata = .;
    sdata = .;
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }

    . = ALIGN(4K);
    edata = .;
    .bss : {
        *(.bss.stack)
        sbss = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
    }

    . = ALIGN(4K);
    ebss = .;
    ekernel = .;

    /DISCARD/ : {
        *(.eh_frame)
    }
}
```

链接脚本规定了从_start的符号开始编址且起始地址为0x80200000，后面依次递增，规定了链接后的ELF文件的各个section的由来。声明了几个外部符号stext、etext、srodata、erodata、sdata、edata、sbss、ebss供rust代码使用。这里的ALIGN(4K)在我的理解是：防止一个物理页（4K）包含多个不同section的数据。最后丢弃不需要的东西。

最后，我们使用rust-objcopy命令工具将编译出来的ELF文件的无用元数据drop掉，得到镜像文件。

## 基于 SBI 服务完成输出和关机
现在我们要借助SBI实现在屏幕上打印字符，以及安全的关机。在此之前，我们需要简单了解SBI是啥。

>SBI (Supervisor Binary Interface) is an interface between the Supervisor Execution Environment (SEE) and the supervisor. It allows the supervisor to execute some privileged operations by using the ecall instruction. Examples of SEE and supervisor are: M-Mode and S-Mode on Unix-class platforms, where SBI is the only interface between them, as well as the Hypervisor extended-Supervisor (HS) and Virtualized Supervisor (VS). 

简单来说，SBI是一个运行在M态的一个程序，它将硬件接口进行封装并为上层操作系统（运行在S态）提供服务接口，以实现模块间的解耦。因此，我们不需要自己手动编写M态的代码并调用BIOS的服务，只需要借助ecall指令调用SBI提供的服务即可。SBI提供的服务和对应的服务号有专门的标准，有兴趣的同学可以了解一下。

```rust
// src/sbi.rs
#[inline(always)]
fn sbi_call(which: usize, arg0: usize, arg1: usize, arg2: usize) -> usize {
    let mut ret;
    unsafe {
        asm!(
            "ecall",
            inlateout("x10") arg0 => ret,
            in("x11") arg1,
            in("x12") arg2,
            in("x17") which,
        );
    }
    ret
}
```

该函数运行在S态，执行ecall指令，向SBI发出请求，x10、x11、x12、x17寄存器作为输入，x10作为输出。

我们所需要的向屏幕打印字符的服务、关机的服务都由SBI进行提供：

```rust
const SBI_CONSOLE_PUTCHAR: usize = 1;
const SBI_CONSOLE_GETCHAR: usize = 2;
const SBI_SHUTDOWN: usize = 8;

pub fn console_putchar(c: usize) {
    sbi_call(SBI_CONSOLE_PUTCHAR, c, 0, 0);
}

pub fn console_getchar() -> usize {
    sbi_call(SBI_CONSOLE_GETCHAR, 0, 0, 0)
}

pub fn shutdown() -> ! {
    sbi_call(SBI_SHUTDOWN, 0, 0, 0);
    panic!("It should shutdown!");
}
```

进一步，我们进行对console_putchar封装：

```rust
struct Stdout;

impl Write for Stdout {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        for c in s.chars() {
            console_putchar(c as usize);
        }
        Ok(())
    }
}

pub fn print(args: fmt::Arguments) {
    Stdout.write_fmt(args).unwrap();
}
```

我们声明了一个空的结构体，为它实现Write的Trait，这个Trait可以一次性输出一个字符串。然后定义一个print函数，该函数对Stdout进一步封装，提供一个格式化输出的接口。

```rust
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

然后我们把print函数封装成两个宏，这两个宏的使用方法和标准库中的使用方法相同。

然后我们看一下rust_main函数都在做些什么：

```rust
#[no_mangle]
pub fn rust_main() -> ! {
    extern "C" {
        fn stext();
        fn etext();
        fn srodata();
        fn erodata();
        fn sdata();
        fn edata();
        fn sbss();
        fn ebss();
        fn boot_stack();
        fn boot_stack_top();
    }
    clear_bss();
    println!("Hello, world!");
    println!(".text [{:#x}, {:#x})", stext as usize, etext as usize);
    println!(".rodata [{:#x}, {:#x})", srodata as usize, erodata as usize);
    println!(".data [{:#x}, {:#x})", sdata as usize, edata as usize);
    println!(
        "boot_stack [{:#x}, {:#x})",
        boot_stack as usize, boot_stack_top as usize
    );
    println!(".bss [{:#x}, {:#x})", sbss as usize, ebss as usize);
    panic!("Shutdown machine!");
}
```

rust_main先把bss段清零，然后调用我们编写的println!宏打印了各个段和boot_stack的起始地址和终止地址。最后调用核心库的panic!宏，把CPU权限交给我们之前注册的Panic处理函数。

```rust
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    if let Some(location) = info.location() {
        println!(
            "Panicked at {}:{} {}",
            location.file(),
            location.line(),
            info.message().unwrap()
        );
    } else {
        println!("Panicked: {}", info.message().unwrap());
    }
    shutdown()
}
```

该函数打印出panic的位置，然后调用SBI的shutdown服务进行关机。

到此为止，一个支持输出和关机的裸机程序就完成了。