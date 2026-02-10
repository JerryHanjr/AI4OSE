# Ch1 学习指南：应用程序与基本执行环境

## 学习目标

理解一个最简程序如何在 RISC-V 裸机上运行。

## 核心问题清单

在阅读代码前，先思考以下问题（可向 AI 提问获取线索）：

### 基础问题
1. 为什么要用 `#![no_std]`？标准库依赖了什么操作系统功能？
2. `#![no_main]` 意味着什么？正常的 Rust 程序 main 函数之前发生了什么？
3. 裸函数 (naked function) 和普通函数有什么区别？为什么 `_start` 需要是裸函数？

### 硬件相关
4. RISC-V 的 M态、S态、U态分别是什么？为什么需要特权级？
5. SBI 是什么？为什么 S态程序不能直接操作硬件，要通过 SBI？
6. 地址 0x80000000 和 0x80200000 分别对应什么？为什么 S态入口不从 0x80000000 开始？

### 深度问题
7. 链接脚本中各个段（.text, .rodata, .data, .bss）的作用是什么？
8. 栈为什么从高地址向低地址增长？如果栈溢出会发生什么？
9. `shutdown(false)` 和 `shutdown(true)` 的区别是什么？QEMU 如何响应？

## AI 对话示例

```
你：为什么 _start 函数需要 naked 属性？

AI：naked 函数不生成函数序言（prologue）和尾声（epilogue）。普通函数编译器会
自动生成 push/pop 栈帧的代码，但 _start 是程序的第一个执行点，此时栈指针还
未设置，如果编译器生成了操作栈的代码就会崩溃。

你：那如果我不用 naked，会怎样？

AI：编译器会在函数开头生成 "addi sp, sp, -N" 来分配栈帧，但此时 sp 可能是
0 或未定义值，写入未知地址会导致 StoreAccessFault 异常。
```

## 实验步骤

1. 阅读 `ch1/src/main.rs`（96行，很短）
2. 阅读 `ch1/build.rs` 中的链接脚本
3. 编译：`cargo build --target riscv64gc-unknown-none-elf`
4. 运行：`qemu-system-riscv64 -machine virt -nographic -bios none -kernel ...`
5. 观察输出 "Hello, world!"

## 扩展思考

- 尝试修改输出内容，重新编译运行
- 如果把 `console_putchar` 改成直接写 UART 寄存器会怎样？
- tg-sbi 的 `nobios` 特性做了什么？查看 tg-sbi 的源码
