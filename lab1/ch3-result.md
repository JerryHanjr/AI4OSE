# Ch3: 多道程序与分时多任务

## 核心知识点

- **多道程序**：多个用户程序同时驻留在内存中
- **任务控制块（TCB）**：管理每个任务的上下文、状态和资源
- **协作式调度**：任务通过 `yield` 系统调用主动让出 CPU
- **抢占式调度**：通过时钟中断强制切换任务，实现时间片轮转
- **系统调用扩展**：新增 `yield`、`clock_gettime`

## 与 Ch2 的关键区别

| 特性 | Ch2 批处理 | Ch3 多道程序 |
|------|-----------|-------------|
| 程序加载 | 同一地址，顺序执行 | 不同地址，并发执行 |
| 调度方式 | 无调度 | 时间片轮转 |
| 内存布局 | 单个 app 区域 | 多个 app 区域(间隔0x200000) |
| 内核栈 | 小 (8页) | 大 (272 KiB) |
| 时钟中断 | 无 | 有 (12500 cycles) |

## 编译运行

```bash
$ cd ch3
$ cargo build --target riscv64gc-unknown-none-elf
$ qemu-system-riscv64 -machine virt -nographic -bios none \
    -kernel target/riscv64gc-unknown-none-elf/debug/tg-ch3
```

## 运行结果

```
   ______                       __
  / ____/___  ____  _________  / /__
 / /   / __ \/ __ \/ ___/ __ \/ / _ \
/ /___/ /_/ / / / (__  ) /_/ / /  __/
\____/\____/_/ /_/____/\____/_/\___/
====================================
[ INFO] LOG TEST >> Hello, world!
[ WARN] LOG TEST >> Hello, world!
[ERROR] LOG TEST >> Hello, world!

[ INFO] load app0 to 0x80400000
[ INFO] load app1 to 0x80600000
[ INFO] load app2 to 0x80800000
...
[ INFO] load app11 to 0x81a00000

Hello, world from user mode program!
[ INFO] app0 exit with code 0
Into Test store_fault...
[ERROR] app1 was killed by StoreFault
...
AAAAAAAAAA [1/5]
BBBBBBBBBB [1/2]
CCCCCCCCCC [1/3]
...（power计算穿插执行）...
AAAAAAAAAA [2/5]
BBBBBBBBBB [2/2]
CCCCCCCCCC [2/3]
AAAAAAAAAA [3/5]
Test write B OK!
CCCCCCCCCC [3/3]
AAAAAAAAAA [4/5]
Test write C OK!
AAAAAAAAAA [5/5]
Test write A OK!
Test sleep OK!
```

## 运行分析

### 多任务交替执行的证据

```
AAAAAAAAAA [1/5]    <- A任务第1次输出
BBBBBBBBBB [1/2]    <- 时间片到期，切换到B
CCCCCCCCCC [1/3]    <- 切换到C
...（power计算中间穿插）...
AAAAAAAAAA [2/5]    <- 又回到A
BBBBBBBBBB [2/2]    <- B完成
CCCCCCCCCC [2/3]    <- C继续
AAAAAAAAAA [3/5]    <- A继续
```

- 12个应用加载到不同内存地址（0x80400000起，间隔0x200000）
- A任务分5次、B任务分2次、C任务分3次完成，三者交替执行
- 时间片机制工作正常：`set_timer(time::read64() + 12500)` 每12500 cycles触发一次

### 关键代码：抢占式调度

```rust
// 设置时钟中断
tg_sbi::set_timer(time::read64() + 12500);
// 执行用户程序
unsafe { tcb.execute() };

match scause::read().cause() {
    // 时钟中断：时间片用完
    Trap::Interrupt(Interrupt::SupervisorTimer) => {
        tg_sbi::set_timer(u64::MAX);  // 清除时钟中断
        false  // 不结束任务，切换到下一个
    }
    // yield 系统调用：主动让出
    Event::Yield => false,
    ...
}
// 轮转到下一个任务
i = (i + 1) % index_mod;
```

## 组件依赖

- 继承 Ch2 所有组件
- 新增 `tg_syscall::Scheduling`（yield 系统调用）
- 新增 `tg_syscall::Clock`（clock_gettime 系统调用）
- 新增 `tg_syscall::Trace`（练习题接口）
