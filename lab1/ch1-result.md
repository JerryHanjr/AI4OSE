# Ch1: 应用程序与基本执行环境

## 核心知识点

- `#![no_std]` / `#![no_main]`：不使用标准库和标准入口，适合裸机编程
- **SBI (Supervisor Binary Interface)**：S态软件向M态固件请求服务的标准接口
- **RISC-V 特权级**：M态(Machine)、S态(Supervisor)、U态(User)
- **链接脚本**：控制程序各段在内存中的布局
- **裸函数 (naked function)**：不生成函数序言/尾声，可在无栈环境下执行

## 内存布局

```
0x80000000  M-mode 区域（tg-sbi 提供）
  .text.m_entry   M-mode 入口代码
  .text.m_trap    M-mode 中断处理
  .bss.m_stack    M-mode 栈空间

0x80200000  S-mode 区域（本程序）
  .text           代码段（含 .text.entry 入口）
  .rodata         只读数据段
  .data           可读写数据段
  .bss            未初始化数据段（含栈空间）
```

## 编译过程

```bash
$ cd ch1
$ cargo build --target riscv64gc-unknown-none-elf
   Compiling tg-sbi v0.1.0-preview.1
   Compiling tg-ch1 v0.3.1-preview.3
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 6.42s
```

## 运行命令

```bash
$ qemu-system-riscv64 -machine virt -nographic -bios none \
    -kernel target/riscv64gc-unknown-none-elf/debug/tg-ch1
```

## 运行结果

```
Hello, world!
```

## 关键代码分析

### 入口函数 `_start`

```rust
#[unsafe(naked)]
#[unsafe(link_section = ".text.entry")]
unsafe extern "C" fn _start() -> ! {
    const STACK_SIZE: usize = 4096;
    #[unsafe(link_section = ".bss.uninit")]
    static mut STACK: [u8; STACK_SIZE] = [0u8; STACK_SIZE];
    core::arch::naked_asm!(
        "la sp, {stack} + {stack_size}",  // 设置栈指针到栈顶
        "j  {main}",                       // 跳转到 rust_main
        stack_size = const STACK_SIZE,
        stack      =   sym STACK,
        main       =   sym rust_main,
    )
}
```

要点：
1. `naked` 属性确保不生成函数序言/尾声
2. 栈空间在 `.bss.uninit` 段中分配（4 KiB）
3. `la sp` 将栈指针设置到栈顶（栈从高地址向低地址增长）

### 主函数

```rust
extern "C" fn rust_main() -> ! {
    for c in b"Hello, world!\n" {
        console_putchar(*c);  // 通过 SBI 逐字符输出
    }
    shutdown(false)  // 正常关机
}
```

通过 SBI 的 `console_putchar` 逐字节输出字符串，然后调用 `shutdown` 退出 QEMU。

## 组件依赖

- `tg-sbi`：SBI 调用封装（console_putchar, shutdown），启用 `nobios` 特性内建 M-mode 启动代码
