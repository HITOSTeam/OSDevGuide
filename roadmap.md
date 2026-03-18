# Project Roadmap

> 目标：持续推进 LTP 覆盖、Linux 语义一致性与内核工程质量，不把项目收缩成“为测试过关而修补”的短期工程。
> 适用对象：当前维护者以及后续所有接手者。

## 核心原则

- 使用 **LTP-driven development with architecture improvement in parallel**。
- 维持大致 `70% LTP 推进 + 30% 架构/治理改进` 的节奏。
- 优先做可复用、Linux-like、可维护的语义修复，不接受只为某个 case 生效的 hack。
- 每一轮都允许并鼓励继续优化、重构、抽象和治理已有实现。
- 后来人不应把“当前已经能跑”当作终点；只要设计、扩展性、语义一致性仍有明显缺口，就应持续改进。

## 当前阶段判断

项目已经越过“只有最基本 syscall 能跑”的阶段，正在进入：

1. 大规模 LTP 继续推进。
2. 把已有功能逐步修成更接近 Linux 的系统语义。
3. 把临时可用实现逐步替换成更稳定的内核机制。

近期已经完成的一类代表性工作：

- `poll/select/epoll` 基础语义已经稳定。
- `epoll_wait` 已经从纯短睡眠 fallback，推进到 pipe、FIFO、socketpair、Unix socket、netlink、loopback net socket、eventfd 的事件驱动唤醒。
- `epoll_wait` 的空 epoll / `epoll_ctl` 自唤醒路径也已收口到同一套 waiter 机制，不再单独依赖 fallback 短睡眠。
- `pidfd` 与 `userfaultfd` 已接入 waiter registration，并完成 riscv64 聚焦回归验证。
- `UnixSocketFile` 的 stream 建链态与 `NetSocketFile` 的本地状态迁移也已补到同一套 waiter 路径里，减少 mixed-support fallback。
- `MqDescriptor` 现已暴露 Linux 风格的 poll mask / waiter registration，为后续 POSIX MQ 的 epoll 覆盖打基础。
- `timerfd` 已从 dummy syscall 路径提升为真实 waitable file，并完成 musl+glibc 聚焦回归；同时调度器开始用上下文切换边界做细粒度 CPU runtime accounting，减少 FAIR/yield-heavy 工作负载只靠 tick 记账的偏差。
- fd table / `CLONE_FILES` / `close_range` / `unshare` 等共享文件表语义已经开始向 Linux 靠拢。

这说明项目下一步不该只是“继续加测试名”，而应继续把这些方向做深做实。

## 近期路线图

### 当前主线：`procfs / filesystem / files` 语义重构

目标：用一条连续的语义重构线，同时降低后续 LTP 推进成本与系统结构债。

判断：

- 当前阶段不宜把重点放在“纯性能优化”或“继续零散补 syscall”。
- 更优策略是 **LTP 驱动的小步重构**：让 `procfs`、`filesystem` 公共层、`files` 生命周期三者一起向 Linux 语义收口。
- 纯优化（例如调度锁拆分、`ext4_lock()` 细化）仍然重要，但应放在这条语义主线之后，或与相关 LTP 子组绑定推进，而不是独立成为当前第一优先级。

建议顺序：

1. 先收 `procfs` 边界与语义源头。
2. 再拆 `syscall/filesystem.rs` 的公共逻辑。
3. 再推进 `files` 对象与 `CLONE_FILES` 生命周期。
4. 最后结合对应回归，再做 ext4/调度热点优化。

### Phase 0: 先收 `procfs` 边界

目标：把 `/proc` 从 ext4 overlay 过渡实现继续推进到真实 pseudo-fs。

优先项：

- 让 `/proc/<pid>`、`/proc/self`、`/proc/<pid>/fd`、`/proc/<pid>/task` 的 lookup / readdir / open 优先走 pseudo 层，而不是依赖 ext4 上是否存在对应实体目录。
- 让 `/proc/sys` 的值逐步回到内核变量 / handler，而不是把 ext4 文件内容当作状态源头。
- 停止继续扩展“sync `/proc/<pid>` 到 ext4 目录实体”的过渡路径。

完成标准：

- `/proc` 与 `/proc/<pid>` 的基础遍历和打开不再依赖 ext4 目录同步。
- `/proc/sys` 的新增/修改语义优先落在 typed kernel state，而不是 `OSInode` 文件内容。
- `proc01`、`sysctl01-04`、一小组 `mountns/setns` follow-up 不退化。

### Phase 1: 收口等待与事件驱动框架

目标：尽量让 `epoll_wait` 的无限等待路径摆脱 fallback 轮询。

优先项：

- 保留并扩展嵌套 `epoll` 回归覆盖；当前 `parent epoll -> child epoll -> pipe/HUP` 基础链路已通过手工 smoke，后续重点转向更复杂 corner case 与等待拓扑。
- 继续把同一套 waiter 机制扩到其他仍未开放真实 epoll wait registration 的 `File` 类型。
- 继续减少 mixed-support 场景下的短睡眠 fallback 覆盖面。

完成标准：

- `epoll_wait`/`epoll_pwait` 相关聚焦回归稳定通过。
- 基础嵌套 `epoll` smoke 持续可复现通过，不因 pipe/HUP 或 waiter 改动退化。
- 新接入对象的 wakeup 生命周期清晰，没有悬挂 waiter 或明显 race。
- `pidfd_open[01-04]` 与 `userfaultfd01` 已在 riscv64 聚焦回归中完成复验；`pidfd_getfd*` / `pidfd_send_signal*` 的当前 `TCONF` 归因于 riscv64 缺少对应 syscall 号，而非 readiness 回归。

### Phase 2: 继续治理 fd / files 生命周期

目标：把当前共享文件表模型继续向 Linux `files_struct` 风格推进。

优先项：

- 继续清理 `dup/dup3/fcntl/close/close_range/openat/pipe2` 中残留的 owner-specific 假设。
- 降低“owner+sharer 重绑定”模型的脆弱性。
- 逐步沉淀为更独立的 files 对象与引用计数语义。
- 让 `fork` / `clone(CLONE_FILES)` / `exec` / `unshare` / `close_range(..., UNSHARE)` 的状态转移更明确。

完成标准：

- `CLONE_FILES`、`exec`、`fork`、`close_range(..., UNSHARE)`、`FD_CLOEXEC` 行为更统一。
- 后续 fd 相关 syscall 不再需要到处复制共享表修补逻辑。

### Phase 3: 继续拆大 syscall 与公共语义层

目标：把重逻辑从 syscall 大文件中下沉，降低重复与维护成本。

优先项：

- 继续从 `syscall/filesystem.rs` 下沉公共路径解析、errno 映射、权限判断逻辑。
- 继续清理仍使用 `translated_str()` 直接杀进程的路径参数读取点。
- 对高频对象抽公共 helper，而不是在 syscall 层重复分支。
- 统一 pseudo/ext4/procfs 路径分流，不再让多个 syscall 各自维护一套分支规则。

完成标准：

- 新语义修复优先落在共享层，不再优先落在 syscall 分支内部。
- 参数错误与地址错误更多按 Linux errno 返回，而不是异常退出。

### 当前建议批次

优先批次：

1. `proc01` + `sysctl01-04`
2. `getdents/readdir/openat/readlinkat` 的 proc/pseudo follow-up
3. `unshare01-02`、`close_range01-02`、`execve/execveat` 的 files 生命周期 follow-up
4. `mountns01-04` 与小组 bind mount follow-up

不建议当前优先做的事项：

- 不建议先独立做 `ext4_lock()` 大拆分。
- 不建议先独立重写调度器。
- 不建议一上来整体重写 `procfs.rs`。

原因：这些动作跨度过大，容易打散当前稳定基线；更合适的方式是绑定到上面几轮语义批次里逐步推进。

## 中期路线图

### Procfs / pseudo-fs 分层

- 把 `/proc` 继续从 ext4 实体耦合中剥离，向纯内存 pseudo-fs 方向推进。
- 让 `/proc/sys` 的可读写项逐步切到 typed handler，而不是 ext4 文件内容即状态。
- 明确真实文件系统、伪文件系统、进程视图文件系统的边界。

### 调度与锁模型优化

- 降低全局调度锁热点。
- 继续评估 ext4 全局大锁的拆分路径。
- 明确哪些路径允许睡眠，哪些路径必须保持短临界区。

### Exec / 动态链接语义治理

- 去掉脆弱的 glibc 私有补丁和魔法偏移兼容逻辑。
- 回到 ELF、auxv、ld.so 规范语义来修复用户态加载路径。

## 长期目标

- 持续提高 LTP 覆盖，而不是把剩余测试永久搁置。
- 把主要子系统逐步改造成“真实 OS 风格”的结构，而不是测试导向脚手架。
- 保证新贡献既能推进通过率，也能留下更好的架构状态给后继者。

## 对后续维护者的约定

- 可以，也应该继续优化与重构。
- 可以，也应该替换当前的过渡实现，只要新实现更符合 Linux 语义且更易维护。
- 不要因为某个实现“已经让测试通过”就停止改进。
- 做完一轮后，更新：
  - `OSGuide/ltp_test_summary.md`
  - `OSGuide/parts/architecture_improvement_roadmap.md`
  - 本文档（如果阶段判断或优先级发生变化）

## 每轮工作建议

1. 先读 `OSGuide/ltp_test_summary.md`。
2. 再读 `OSGuide/parts/architecture_improvement_roadmap.md`。
3. 最后看本文，确认当前阶段最值得推进的方向。
4. 选择 5 到 20 个同类测试，优先能推动一个底层语义簇的批次。
5. 修复后必须做 focused 回归，并同步更新文档。
