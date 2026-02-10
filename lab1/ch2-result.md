# Ch2: 批处理系统

## 核心知识点

- **批处理系统**：将多个用户程序打包，内核依次自动执行
- **特权级切换**：U-mode（用户态）与 S-mode（内核态）之间的切换
- **Trap 处理**：用户程序通过 `ecall` 触发系统调用，或因异常陷入内核
- **上下文保存与恢复**：进入/退出 Trap 时保存/恢复用户寄存器状态
- **系统调用**：`write`（输出）和 `exit`（退出）

## 编译过程

```bash
$ cd ch2
$ cargo build --target riscv64gc-unknown-none-elf
   Compiling tg-sbi v0.1.0-preview.1
   Compiling tg-kernel-context v0.1.0-preview.1
   Compiling tg-syscall v0.1.0-preview.2
   Compiling tg-console v0.1.0-preview.2
   Compiling tg-ch2 v0.3.1-preview.3
    Finished `dev` profile in 23.70s
```

## 运行命令

```bash
$ qemu-system-riscv64 -machine virt -nographic -bios none \
    -kernel target/riscv64gc-unknown-none-elf/debug/tg-ch2
```

## 运行结果

```
   ______                       __
  / ____/___  ____  _________  / /__
 / /   / __ \/ __ \/ ___/ __ \/ / _ \
/ /___/ /_/ / / / (__  ) /_/ / /  __/
\____/\____/_/ /_/____/\____/_/\___/
====================================
[TRACE] LOG TEST >> Hello, world!
[DEBUG] LOG TEST >> Hello, world!
[ INFO] LOG TEST >> Hello, world!
[ WARN] LOG TEST >> Hello, world!
[ERROR] LOG TEST >> Hello, world!

[ INFO] load app0 to 0x80400000
Hello, world from user mode program!
[ INFO] app0 exit with code 0

[ INFO] load app1 to 0x80400000
Into Test store_fault, we will insert an invalid store operation...
Kernel should kill this application!
[ERROR] app1 was killed because of Exception(StoreFault)

[ INFO] load app2 to 0x80400000
3^10000=5079(MOD 10007)
...
3^100000=2749(MOD 10007)
Test power OK!
[ INFO] app2 exit with code 0

[ INFO] load app3 to 0x80400000
Try to execute privileged instruction in U Mode
Kernel should kill this application!
[ERROR] app3 was killed because of Exception(IllegalInstruction)

[ INFO] load app4 to 0x80400000
Try to access privileged CSR in U Mode
Kernel should kill this application!
[ERROR] app4 was killed because of Exception(IllegalInstruction)

[ INFO] load app5-app7 to 0x80400000
...power_3/5/7 计算测试均通过...
Test power_3 OK! / Test power_5 OK! / Test power_7 OK!
```

## 运行分析

共加载并执行了 8 个用户程序：

| App | 功能 | 结果 |
|-----|------|------|
| app0 | Hello World | 正常退出 (code 0) |
| app1 | 非法存储测试 | 被内核杀死 (StoreFault) |
| app2 | 幂运算 (3^n mod 10007) | 正常退出 |
| app3 | 执行特权指令 | 被内核杀死 (IllegalInstruction) |
| app4 | 访问特权CSR | 被内核杀死 (IllegalInstruction) |
| app5-7 | 大数幂运算 | 正常退出 |

关键观察：
- 所有用户程序加载到同一地址 `0x80400000`（批处理，顺序执行）
- 非法操作被内核正确捕获并处理（StoreFault、IllegalInstruction）
- 批处理模式下，下一个 app 会覆盖上一个 app 的内存区域

## 关键代码分析

### Trap 处理流程

```rust
// 用户程序触发 Trap 后回到内核
unsafe { ctx.execute() };

match scause::read().cause() {
    // 用户态系统调用
    Trap::Exception(Exception::UserEnvCall) => {
        match handle_syscall(&mut ctx) {
            Done => continue,     // 继续执行用户程序
            Exit(code) => ...,    // 用户程序退出
            Error(id) => ...,     // 不支持的系统调用
        }
    }
    // 其他异常：杀死应用
    trap => log::error!("app{i} was killed because of {trap:?}"),
}
```

### 系统调用分发

```rust
fn handle_syscall(ctx: &mut LocalContext) -> SyscallResult {
    let id = ctx.a(7).into();      // a7 寄存器存放 syscall ID
    let args = [ctx.a(0), ...];     // a0-a5 存放参数
    match tg_syscall::handle(caller, id, args) {
        Ret::Done(ret) => {
            *ctx.a_mut(0) = ret;    // 返回值写入 a0
            ctx.move_next();         // sepc += 4，跳过 ecall
        }
        ...
    }
}
```

## 组件依赖

- `tg-sbi`：SBI 调用
- `tg-kernel-context`：用户上下文保存/恢复，特权级切换
- `tg-syscall`：系统调用 ID 定义和分发框架
- `tg-console`：格式化输出（print!/println!）
- `tg-linker`：链接脚本生成、内核布局、应用元数据
