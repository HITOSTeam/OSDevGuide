# LoongArch64 启动记录（调试 + 修复）

本文总结了排查 LoongArch64 启动卡死（initproc 已入队但无用户输出）过程中所做的改动。目标是实现首次干净进入用户态并让 shell 正常工作。

## 已定位的根因

- 内核栈与 TRAMPOLINE 的虚拟地址重叠：TRAMPOLINE 位于 VA 顶端（Sv39 风格），与 LoongArch 的 PGDL/PGDH 分裂和用户低地址范围冲突。
- 直接映射窗口（DMW0/DMW1）导致低 VA（0x3ffff...）在 PLV0 下被当作物理地址，执行 trampoline/restore 路径时发生异常。

## 地址/映射修复

- 将 TRAMPOLINE/SIGRETURN/TRAP_CONTEXT 移到低规范地址范围：0x0000_003f_ffff_f000（及其以下）。
- 为 LoongArch 增加独立的内核栈顶地址（0xffff_ffff_ffff_f000），并在内核栈分配（config + task/id）中使用。
- 在 mm/address.rs 中为 LoongArch 使用 48 位 VA/PA 宽度，避免截断与错误符号扩展。
- 将 UART MMIO 加入 LoongArch 的 MMIO 列表，以便在禁用 DMW 后控制台仍可用。

## Trap/restore 路径修复

- 确保 trap_loongarch64.S 中 SAVE0 与 sp 处理正确（SAVE0 保留 TrapContext 指针；sp 从上下文恢复）。
- 在 restore 中切换 PGDL 到用户页表，并在 PGD 更改后使用 invtlb 0x3（非全局失效）。
- 在内核态保持内核 EENTRY；切到用户页表后，将 EENTRY 设置为 alltraps，使 restore 中的异常走用户态陷入路径。
- 在 loongarch 的 rust_main 中标记启动 hart 为 online，保证调度在活动 hart 上继续。

## DMW 处理

- 新增 disable_direct_map_windows()（清除 DMW0/DMW1 并失效 TLB），并在 LoongArch 的 mm::init() 后调用，防止低 VA 的 trampoline 页面被当作物理地址。

## 调试信息（可选）

- trap_return 打印 trampoline/entry/trapcx/stack 的 PTE 以验证映射。
- idle 调度器打印 ready 队列与首次切换信息。
- restore 内部曾加入临时 UART 标记用于定位故障，用户态进入正常后已移除。

## 结果

- 成功进入用户态，init_proc 有输出，LoongArch64 上 shell 正常工作。

## OSComp 双盘与 /musl 路径

- 当前设计是“双盘”：disk0 为主盘（含 `/user`、`/extra`），disk1 为 sdcard（含 `/musl`、`/glibc`）。
- 根选择策略：含 `/user` 的盘为主盘；绝对路径仅在主盘查找失败时回退到次盘。
- 因此对 `/musl`、`/glibc` 的访问需要用绝对路径（或 `cd /musl` 后用相对路径），否则不会自动落到第二块盘。

## sdcard 镜像格式注意事项

- OSComp 提供的 sdcard 镜像通常启用了 `has_journal` / `metadata_csum` / `metadata_csum_seed`。
- 当前 ext4 驱动未完整支持这些特性，写入后可能导致元数据校验不一致。
- 建议重新生成 sdcard 镜像时关闭相关特性：
  - `mkfs.ext4 -O ^has_journal,^metadata_csum,^metadata_csum_seed sdcard-la.img`
  - 对应的构建入口已在 `testsuits-for-oskernel/Makefile` 中更新。

## LoongArch IO 性能定位与优化

- 增加轻量性能计数器（`/proc/perf`）：统计 UART 字节/flush 周期、块设备读写次数/字节/周期、ext4 cache 命中率，方便定位瓶颈。
- ext4 块缓存优化：LoongArch 下扩大缓存到 2048 条，并在命中时移动到队尾形成 LRU 行为；cache miss 时做 4 块读预取（read-ahead）。
- 块设备读写合并：`BlockDevice` 新增 `read_blocks`/`write_blocks`，virtio 驱动一次读/写多个块，降低 I/O 请求次数。
- virtio DMA：LoongArch PCI 走直接 DMA（缓冲位于物理内存范围时跳过 bounce），并强制 DMA 页为 cacheable，减少内存映射/拷贝成本。
- ext4 元数据延迟同步：移除热路径中的立即 `sync`，改为在 cache 淘汰或 `sync_all` 时回写，避免频繁小写放大。

## LoongArch 计时频率修复

- 使用 `cpucfg` 读取 CPUCFG4/5 的基频与倍频/分频系数，动态计算并设置 `CLOCK_FREQ`。
- `time` 与相关 syscall 使用运行时的 `clock_freq()`，避免固定常量导致 `sleep/kill` 等测试时序异常。
- LoongArch 定时器改为按 TCFG 相对计数配置（不再以 `rdtime` 计算绝对触发值），避免计数源不一致导致的
  定时中断过稀、抢占失效（例如 `sleep 5 & kill $!` 延迟）。
- 若 QEMU/硬件上的计时器频率与 `rdtime` 不一致，会自动进行一次校准，保证时钟中断周期接近
  `TICKS_PER_SEC`（避免后台作业被 `sleep` 长时间独占 CPU）。

Files:

- `os/src/time.rs`
- `os/src/arch/loongarch64/mod.rs`
- `os/src/arch/loongarch64/trap/handler.rs`

## 清理建议

- 稳定后可禁用 DEBUG_TRAP 或移除 trap_return 的 PTE 打印。

## LoongArch64 相关概念补充

- PGDL/PGDH：LoongArch 将页表根地址拆分为低半区 PGDL（Page Global Directory Low）和高半区 PGDH（Page Global Directory High）。通常用户空间映射位于低地址区，内核映射位于高地址区。恢复到用户页表时需要切换 PGDL，否则会继续使用内核页表的低半区。
- DMW（Direct Map Window）：硬件提供的直接映射窗口（DMW0/DMW1），可将某些 VA 区间直接映射到 PA，主要用于内核高性能访问。若启用 DMW，低 VA 可能被解释为物理地址，从而绕过页表导致异常或安全问题。
- PLV（Privilege Level）：LoongArch 的特权等级（PLV0 为内核态，PLV3 为用户态）。某些地址转换规则（如 DMW）只在特定 PLV 生效。
- EENTRY：异常入口地址寄存器，指向当前特权级下的异常处理入口。内核保持内核 EENTRY；切换用户页表后若发生异常，必须确保 EENTRY 指向正确的陷入入口（例如 alltraps）。
- TRAMPOLINE：用户态与内核态之间的跳板页，通常放在固定地址以完成陷入/返回。位置与映射必须避免与内核栈或 DMW 冲突。
- TrapContext：保存用户态寄存器现场的结构，陷入时写入，返回时恢复。
- invtlb 0x3：TLB 失效指令中的非全局失效操作，用于在切换页表后刷新与当前 ASID 相关的条目，避免旧映射干扰。

# LoongArch Busybox/Shell Fixes

This note summarizes the changes made to get the LoongArch busybox tests
running cleanly without touching the testsuite scripts.

## Execve shell fallbacks

- Detect system shell paths (`/bin/sh`, `/bin/dash`, etc.) in `execve` and
  reroute them to `busybox sh` if available. This avoids the LoongArch dash
  crash and keeps `#!/bin/sh` scripts working.
- Apply the same fallback when a script's shebang points to `/bin/sh` or
  `/bin/dash`, by rewriting the interpreter to `busybox sh`.

Files:

- `os/src/syscall/process.rs`

## Fork behavior on LoongArch

- Add a full-copy user address space clone helper (no COW) and use it for
  LoongArch fork. This avoids COW edge cases observed when busybox `sh`
  forks and execs `./busybox`.
- Keep COW fork enabled for non-LoongArch targets.

Files:

- `os/src/mm/memory_set.rs`
- `os/src/task/process_block.rs`

## Enable FPU for busybox

- LoongArch `ecode=15` is FloatingPointUnavailable. Busybox/musl may emit
  base FP instructions even for simple commands, so enable EUEN.FPE during
  bootstrap to keep user code running.

Files:

- `os/src/arch/loongarch64/mod.rs`

## brk/musl test fix

- Lower the interpreter base address on LoongArch to keep the initial stack/heap
  below 2GiB, so `brk(0)` and subsequent `brk(new)` stay in the 32-bit range
  expected by the oscomp musl tests.

Files:

- `os/src/mm/memory_set.rs`

## mkdir/chdir on /musl

- Repair missing inode type bits using the directory entry file type when
  resolving paths, so `mkdir`/`chdir` tests on the ext4 sdcard work even when
  the inode mode field is zeroed.
- If the directory entry says "directory" but the inode data does not look
  like a directory, treat it as a regular file to avoid `open` returning
  `EISDIR` for files such as `test_close.txt`.

Files:

- `ext4-fs/src/vfs.rs`

## musl/basic 调试输出（可选）

- 在 `DEBUG_SYSCALL` 下为 `brk`、`mkdir`、`chdir` 增加详细日志，方便定位
  heap 范围和 inode 类型缺失导致的错误。

Files:

- `os/src/syscall/memory.rs`
- `os/src/syscall/filesystem.rs`

---

- ext4 块缓存改为索引 LRU + LoongArch 更大缓存/预读：ext4-fs/src/block_cache.rs:145、ext4-fs/src/block_cache.rs:183
- VirtIO share OOM 改成堆上 bounce：os/src/drivers/block/virtio_blk.rs:110、os/src/drivers/block/virtio_blk.rs:241

## glibc libc-test 全量通过（LoongArch OOM 修复）

- execve 走“懒加载 ELF”路径，避免把动态测试入口 `entry-dynamic.exe`/glibc 相关 ELF 全部读入内核堆：
  - ELF 头解析与段映射改为 reader 方式：`os/src/mm/memory_set.rs`
  - `execve` 使用 inode + reader 构建 `MemorySet`：`os/src/syscall/process.rs`
  - 新增 `exec_with_memory_set/exec_dyn_with_memory_set` 复用安装逻辑：`os/src/task/process_block.rs`
- ext4 块缓存 LRU 队列做 compact，避免长期运行时队列无限膨胀导致 48MiB 级别堆分配失败：
  - `ext4-fs/src/block_cache.rs`

## LoongArch libcbench OOM 修复（低端内存回收）

- LoongArch64 物理内存起始为 `0x8000_0000`，内核镜像加载到 `0x9000_0000` 之后，
  原先 frame allocator 只使用内核镜像之后的高端区间，导致 pthread/bench 触发 OOM。
- 解决方式：
  - 在 kernel 地址空间中 **额外映射** `phys_mem_start .. stext` 的低端物理内存；
  - 在 frame allocator 初始化时 **回收** 该低端区间加入可分配列表。

Files:

- `os/src/mm/memory_set.rs`
- `os/src/mm/frame_allocator.rs`

## cyclictest 失败定位：LoongArch musl `sched_getparam` ABI/实现异常

现象：

- `cyclictest` 报错：`unable to get scheduler parameters`（`rt-utils.c: check_privs()` 中的 `sched_getparam(0, &old_param)` 失败）。
- syscall 追踪显示 **只有** `sched_getaffinity(123)`，**没有** `sched_getparam(121)` / `sched_getscheduler(120)` / `sched_setscheduler(119)`。

对比验证（排除内核问题）：

- `busybox chrt -p 1` 能正确打印 `policy/priority`，说明内核 `sched_getparam/getscheduler` syscall 正常。

二进制证据（musl libc）：

- libc 路径：`loongarch_img_info/mnt/musl/lib/libc.so`
- 函数机器码对比（每条 4 字节指令）：
  - `sched_getaffinity`（正常 syscall stub 形式）：
    - `02ffc063 29c02061 0281ec0b 002b0000 ...`

  - `getpid`（正常 syscall stub 形式）：
    - `0282b00b 002b0000 00408084 4c000020`
  - `sched_getparam/sched_getscheduler/sched_setparam/sched_setscheduler`：
    - 中间指令包含 `54bf83ff / 54bf63ff / 54bf1fff / 54beffff`
    - 明显 **不符合** 正常 syscall stub 编码模式

结论：

- `cyclictest` 失败不是 testsuits 缺失，也不是内核 syscall 缺失，
  而是 **LoongArch musl 的 `sched_getparam` 家族函数在用户态未正确发起 syscall**（ABI/实现问题）。

## LTP（riscv64）acct/工具相关修复记录

为解决 LTP `acct02`/`abort01`/`access01` 等问题与工具缺失做了以下改动：

- 增加 `/proc/config.gz` 伪文件：提供最小 gzip 配置内容（`CONFIG_BSD_PROCESS_ACCT=y`、`CONFIG_BSD_PROCESS_ACCT_V3 is not set`），并加入 `/proc` 目录条目。
  - `os/src/fs/procfs.rs`
  - `os/src/fs/pseudo.rs`
- 修复伪文件 `st_size=0` 导致 `zcat /proc/config.gz` 阻塞：`fstat/newfstatat` 对 `PseudoFile::Static` 返回实际长度。
  - `os/src/syscall/filesystem.rs`
- 实现 `acct(2)` + 退出写账：`syscall_acct` 设置输出文件，进程退出时写入 `struct acct` 记录到文件末尾。
  - `os/src/syscall/filesystem.rs`
  - `os/src/task/processor.rs`
- SA_RESTART + wait4 EINTR 处理：支持被信号中断后可重启的 syscall，避免 `abort01` 触发的 `waitpid` EINTR 直接失败。
  - `os/src/syscall/process.rs`
  - `os/src/task/task_block.rs`
- busybox 兜底收紧：新增 applet allowlist（默认只允许 `zcat`），仅允许白名单命令走 busybox fallback，修复 `access01` 误判。
  - `os/src/syscall/mod.rs`
  - `os/src/syscall/process.rs`
  - `os/src/syscall/filesystem.rs`
- 默认 `PATH` 增补 LTP 目录：加入 `/musl/ltp/testcases/bin` 与 `/glibc/ltp/testcases/bin`，确保 `acct02_helper` 能被 `tst_get_path()` 解析到。
  - `os/src/syscall/process.rs`

镜像/工具层面（不涉及源码）：

- 在基础镜像中更新静态 busybox，并单独放置真实 `zcat`（来自 busybox applet），保证 `zcat` 可用且不依赖 shell fallback。

## LTP（riscv64）waitpid/信号/进程组修复记录

为解决 `waitpid04` 期望 errno 以及 `waitpid_common` 的 job-control 测试（`waitpid06~13`）相关问题，新增/修复了：

- `wait4` flags 校验：只允许 `WNOHANG|WUNTRACED|WCONTINUED`，非法 flags 返回 `EINVAL`；对 `pid==INT_MIN` 返回 `ESRCH`（与 `waitpid04` 预期一致）。
  - `os/src/syscall/process.rs`
- 进程组语义：新增 `pgid` 字段，子进程继承父进程 pgid；支持 `setpgid/getpgid`，`pid==0` 与 `pid<0` 的 waitpid 进程组匹配。
  - `os/src/task/process_block.rs`
  - `os/src/syscall/misc.rs`
  - `os/src/syscall/mod.rs`
- 停止/继续（job-control）信号处理：`SIGSTOP/SIGTSTP/SIGTTIN/SIGTTOU` 默认动作为停止，`SIGCONT` 默认继续；停止/继续时唤醒父进程 waiters，并让 `wait4` 产生 `WIFSTOPPED/WIFCONTINUED` 状态。
  - `os/src/syscall/signal.rs`
  - `os/src/syscall/process.rs`
- /proc 状态同步：`/proc/<pid>/stat` 与 `/proc/<pid>/status` 输出 `pgrp` 与 `T (stopped)` 状态。
  - `os/src/fs/procfs.rs`
- 退出码编码修正：`exit/exit_group` 按 8-bit 规范截断，避免 `exit(-1)` 被误当作 signal 退出（导致 waitpid 测试错误）。
  - `os/src/syscall/flow.rs`
- futex 共享 key 修复：共享 futex 以 `(0,uaddr)` 作为 key 入队，避免 checkpoint futex 超时（`waitpid_common` 依赖）。
  - `os/src/syscall/futex.rs`
