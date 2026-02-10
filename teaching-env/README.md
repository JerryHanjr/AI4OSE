# AI辅助OS内核教学实验环境

## 设计理念

本教学环境以"AI辅助+问题驱动"为核心方法，区别于传统的"先看教材再做实验"模式。

核心思路：**带着问题学，用AI解答，通过实验验证**。

## 环境特点

### 1. AI驱动的问题式学习

每个实验章节提供"核心问题清单"，学生可以：
- 先尝试自己思考答案
- 带着问题与 AI（如 Claude、ChatGPT）对话
- 通过运行实验代码验证 AI 的回答是否正确
- 在理解中发现新问题，形成"问题→AI解答→实验验证→新问题"的循环

### 2. 分层递进的知识结构

```
第一层：概念理解（问题清单 + AI对话）
    ↓
第二层：代码阅读（关键源码 + 注释分析）
    ↓
第三层：实验运行（编译、运行、观察输出）
    ↓
第四层：深度思考（扩展问题 + 对比分析）
```

### 3. 组件化理解

基于 tg-* 系列 Crate 理解模块化 OS 设计：

```
tg-sbi ─────────────► SBI 接口抽象
tg-kernel-context ──► 上下文切换（寄存器保存/恢复）
tg-syscall ─────────► 系统调用框架
tg-kernel-alloc ────► 内存分配（Buddy Allocator）
tg-kernel-vm ───────► 虚拟内存管理（Sv39 页表）
tg-task-manage ─────► 进程/线程管理
tg-console ─────────► 控制台 I/O
tg-linker ──────────► 链接脚本和内存布局
```

## 学习路径

| 阶段 | 章节 | 核心目标 | 学习指南 |
|------|------|---------|---------|
| 1 | Ch1 | 理解裸机启动过程 | [guides/ch1-guide.md](guides/ch1-guide.md) |
| 2 | Ch2 | 理解特权级和 Trap | [guides/ch2-guide.md](guides/ch2-guide.md) |
| 3 | Ch3 | 理解多任务调度 | [guides/ch3-guide.md](guides/ch3-guide.md) |
| 4 | Ch4 | 理解虚拟内存 | [guides/ch4-guide.md](guides/ch4-guide.md) |
| 5 | Ch5 | 理解进程管理 | [guides/ch5-guide.md](guides/ch5-guide.md) |

## 对比分析

详细的与参考仓库的对比分析见 [comparison.md](comparison.md)。
效率评估报告见 [efficiency-eval.md](efficiency-eval.md)。

## 仓库地址

- 本仓库：https://github.com/JerryHanjr/AI4OSE
- 参考仓库：https://github.com/rcore-os/rCore-Tutorial-in-single-workspace/tree/test
