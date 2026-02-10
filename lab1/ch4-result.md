# Ch4: 地址空间

## 核心知识点

- **Sv39 虚拟内存**：RISC-V 三级页表，39位虚拟地址
- **地址空间隔离**：每个进程独立页表，互相不可访问
- **异界传送门（MultislotPortal）**：跨地址空间的上下文切换
- **ELF 加载**：解析 ELF 格式用户程序并映射到独立地址空间
- **内核堆分配器**：Buddy Allocator 支持动态内存分配
- **sbrk 系统调用**：用户程序动态调整堆空间

## Sv39 页表机制

```
虚拟地址 (39位):
┌─────────┬─────────┬─────────┬──────────────┐
│ VPN[2]  │ VPN[1]  │ VPN[0]  │   Offset     │
│  9位    │  9位    │  9位    │    12位      │
└─────────┴─────────┴─────────┴──────────────┘

三级页表查找：
satp.PPN → L2页表 → L1页表 → L0页表 → 物理页号
```

## 编译运行

```bash
$ cd ch4
$ cargo build --target riscv64gc-unknown-none-elf
$ qemu-system-riscv64 -machine virt -nographic -bios none \
    -kernel target/riscv64gc-unknown-none-elf/debug/tg-ch4
```

## 运行结果

```
   ______                       __
  / ____/___  ____  _________  / /__
 / /   / __ \/ __ \/ ___/ __ \/ / _ \
/ /___/ /_/ / / / (__  ) /_/ / /  __/
\____/\____/_/ /_/____/\____/_/\___/
====================================

[ INFO] .text ----> 0x80200000..0x80217000
[ INFO] .rodata --> 0x80217000..0x80220000
[ INFO] .data ----> 0x80220000..0x812c98b0
[ INFO] .boot ----> 0x812ca000..0x812d0000
[ INFO] (heap) ---> 0x812d0000..0x81a00000

[ INFO] detect app[0]: 0x80220088..0x80365af0
[ INFO] process entry = 0x14328, heap_bottom = 0x1f000
...（13个应用检测）...
[ INFO] detect app[12]: 0x8117fdd8..0x812c95f0

Hello, world from user mode program!
Into Test store_fault...
[ERROR] unsupported trap: Exception(StorePageFault), stval = 0x0, sepc = 0x14298
...
Test power OK!
[ERROR] unsupported trap: Exception(IllegalInstruction)...
...
Test write A/B/C OK!
Test power_3/5/7 OK!
Test sleep OK!
Test sbrk start.
origin break point = 20000
one page allocated,  break point = 21000
try write to allocated page
write ok
10 page allocated,  break point = 2b000
11 page DEALLOCATED,  break point = 20000
try DEALLOCATED more one page, should be failed.
Test sbrk almost OK!
now write to deallocated page, should cause page fault.
[ERROR] unsupported trap: Exception(StorePageFault), stval = 0x20000, sepc = 0x146d2
```

## 运行分析

### 内核内存布局

| 段 | 地址范围 | 大小 |
|----|---------|------|
| .text | 0x80200000..0x80217000 | 92 KiB |
| .rodata | 0x80217000..0x80220000 | 36 KiB |
| .data | 0x80220000..0x812c98b0 | ~16.7 MiB |
| .boot | 0x812ca000..0x812d0000 | 24 KiB |
| heap | 0x812d0000..0x81a00000 | ~7.2 MiB |

### sbrk 内存管理测试

```
break: 0x20000 → 0x21000 (分配1页)
       0x21000 → 0x2b000 (分配10页)
       0x2b000 → 0x20000 (释放11页)
写入已释放页面 → StorePageFault (正确行为)
```

### 与 Ch3 的关键区别

| 特性 | Ch3 | Ch4 |
|------|-----|-----|
| 地址空间 | 共享物理内存 | 每进程独立虚拟地址空间 |
| 异常类型 | StoreFault | StorePageFault |
| 内存分配 | 静态 | 动态（Buddy Allocator） |
| 上下文切换 | 直接切换 | 通过异界传送门跨地址空间切换 |
| 新增syscall | 无 | sbrk (堆管理) |

### 关键代码：地址翻译

```rust
// Ch4 中的 write 系统调用需要地址翻译
impl IO for SyscallContext {
    fn write(&self, caller: Caller, fd: usize, buf: usize, count: usize) -> isize {
        const READABLE: VmFlags<Sv39> = build_flags("RV");
        // 通过页表将用户虚拟地址翻译为物理地址
        if let Some(ptr) = process.address_space
            .translate::<u8>(VAddr::new(buf), READABLE) {
            // 使用翻译后的物理地址访问数据
            print!("{}", unsafe { ... });
        }
    }
}
```

## 组件依赖

- 继承 Ch3 所有组件
- 新增 `tg-kernel-alloc`：Buddy Allocator 内核堆分配器
- 新增 `tg-kernel-vm`：虚拟内存管理（Sv39 页表、地址空间）
- 新增 `xmas-elf`：ELF 文件解析
- 新增 `tg_kernel_context::foreign::MultislotPortal`：跨地址空间上下文切换
