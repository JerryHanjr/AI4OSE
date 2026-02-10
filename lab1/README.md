# rCore-Tutorial 基础实验记录 (Ch1-Ch5)

基于参考仓库 [rCore-Tutorial-in-single-workspace](https://github.com/rcore-os/rCore-Tutorial-in-single-workspace/tree/test) 完成的5个基础实验。

## 环境

- Rust: 1.93.0 (stable)
- Target: riscv64gc-unknown-none-elf
- QEMU: qemu-system-riscv64 (virt machine)
- Host: macOS (Darwin)

## 实验列表

| 实验 | 名称 | 状态 | 结果文件 |
|------|------|------|---------|
| Ch1 | 应用程序与基本执行环境 | 已完成 | [ch1-result.md](ch1-result.md) |
| Ch2 | 批处理系统 | 已完成 | [ch2-result.md](ch2-result.md) |
| Ch3 | 多道程序与分时多任务 | 已完成 | [ch3-result.md](ch3-result.md) |
| Ch4 | 地址空间 | 已完成 | [ch4-result.md](ch4-result.md) |
| Ch5 | 进程管理 | 已完成 | [ch5-result.md](ch5-result.md) |

## 快速复现

```bash
# 1. 克隆参考仓库
git clone -b test https://github.com/rcore-os/rCore-Tutorial-in-single-workspace.git
cd rCore-Tutorial-in-single-workspace

# 2. 安装工具链
rustup target add riscv64gc-unknown-none-elf
rustup component add rust-src

# 3. 编译运行（以Ch1为例）
cd ch1
cargo build --target riscv64gc-unknown-none-elf
qemu-system-riscv64 -machine virt -nographic -bios none \
    -kernel target/riscv64gc-unknown-none-elf/debug/tg-ch1
```

## 源码

`src/` 目录下保存了 Ch1-Ch5 的 main.rs 关键源码副本，来自参考仓库。
