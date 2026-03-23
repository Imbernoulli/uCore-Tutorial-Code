# Lab1 实验报告

## 实现功能简述

本次实验在 ch3 多道程序与分时多任务系统的基础上，实现了 `sys_trace` 系统调用（ID: 410）。该系统调用支持三种功能模式：模式 0 读取指定地址处一个字节的值；模式 1 向指定地址写入一个字节；模式 2 查询当前任务对指定系统调用的累计调用次数（含本次）。实现方式为：在进程控制块 `struct proc` 中新增 `syscall_count` 数组记录每个系统调用的调用次数，在 `syscall()` 入口处统一递增计数器，并在 switch 分发中添加 `SYS_trace` 的处理分支。

## 问答题

### 1. U 态程序使用 S 态特权指令的行为

使用 RustSBI v0.3.0-alpha.2。在 U 态下：

- `__ch2_bad_register`：尝试通过 `csrr` 读取 `sstatus` 寄存器，触发 `IllegalInstruction` 异常，内核捕获后终止该进程。
- `__ch2_bad_instruction`：尝试执行 `sret` 等 S 态特权指令，同样触发 `IllegalInstruction` 异常被终止。
- `__ch2_bad_address`：访问非法内存地址，触发 `StorePageFault` 或 `LoadPageFault` 异常被终止。

这体现了 RISC-V 特权级机制对用户态程序的保护。

### 2. trampoline.S 问答

#### 2.1 L79: 刚进入 `userret` 时，`a0`、`a1` 分别代表了什么值？

- `a0`：当前进程的 trapframe 地址。
- `a1`：用户页表的 satp 值（用于切换到用户地址空间）。

这两个值由 `usertrapret()` 函数作为参数传入。

#### 2.2 L87-L88: `sfence.vma` 指令的作用？删掉会导致错误吗？

```assembly
csrw satp, a1
sfence.vma zero, zero
```

`sfence.vma zero, zero` 用于刷新 TLB（Translation Lookaside Buffer），确保页表切换后，后续的地址翻译使用新的页表。在当前章节（ch3）中，由于所有任务和内核共享同一地址空间，实际上没有切换页表，因此删掉该指令不会导致错误。但在后续章节引入虚拟地址空间后，删掉会导致 TLB 中残留旧映射，引发错误。

#### 2.3 L96-L125: 为何注释说要除去 `a0`？哪一个地址代表 `a0`？`a0` 的值存在何处？

因为此时 `a0` 正被用作 trapframe 的基地址指针，不能在恢复过程中被覆盖，否则后续的 `ld` 指令将无法正确寻址。`a0` 在 trapframe 中的偏移是 112（即 `112(a0)` 处）。用户态的 `a0` 值此时已经通过 L92-L93 被存入了 `sscratch` 寄存器中。

#### 2.4 `userret` 中发生状态切换的指令？

L132 的 `sret` 指令。执行 `sret` 后，CPU 将 `sepc` 的值赋给 `pc`，同时将 `sstatus` 中的 `SPP` 位对应的特权级恢复（设为 U 态），从而进入用户态。`usertrapret()` 在调用 `userret` 之前已将 `sstatus.SPP` 设置为 User。

#### 2.5 L29: 执行后 `a0` 和 `sscratch` 中各是什么值？

```assembly
csrrw a0, sscratch, a0
```

执行后：
- `a0` = 原 `sscratch` 的值，即 trapframe 的地址。
- `sscratch` = 原 `a0` 的值，即用户程序的 `a0`（系统调用参数/返回值）。

`csrrw` 是原子交换指令，将 `a0` 与 `sscratch` 的值互换。这样 `a0` 就可以用作 trapframe 基地址来保存其他寄存器。

#### 2.6 L32-L61: 从 trapframe 第几项开始保存？为什么？是否保存了所有值？

从 trapframe 的第 6 项开始保存（偏移 40，即 `ra`）。前 5 项（偏移 0-32）分别是 `kernel_satp`、`kernel_sp`、`kernel_trap`、`epc`、`kernel_hartid`，这些是内核信息，不需要在此处保存用户寄存器值。

没有保存所有用户寄存器——`a0` 没有在这里保存，因为 `a0` 当前存放的是 trapframe 地址而非用户态的 `a0`。用户态 `a0` 在 L64-L65 中通过 `csrr t0, sscratch` 从 `sscratch` 中取出后单独保存到 `112(a0)` 处。

#### 2.7 进入 S 态是哪一条指令发生的？

`ecall` 指令（在用户程序中执行）。当用户态执行 `ecall` 时，硬件自动将特权级切换到 S 态，将 PC 保存到 `sepc`，并跳转到 `stvec` 指向的 `uservec` 处。

#### 2.8 L75-L76: `ld t0, 16(a0)` 执行后 `t0` 的值是什么？

```assembly
ld t0, 16(a0)
jr t0
```

`t0` 的值是 `usertrap` 函数的地址。trapframe 偏移 16 处存放的是 `kernel_trap` 字段，该字段在 `usertrapret()` 中被设置为 `usertrap` 的地址。`jr t0` 随后跳转到 `usertrap` 函数执行具体的陷入处理。

## 荣誉准则

1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与 **以下各位** 就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：

    无

2. 此外，我也参考了 **以下资料** ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：

    uCore-Tutorial-Guide 教程文档

3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按"-100"分计。
