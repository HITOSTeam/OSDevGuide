# OS架构风险与改进路线图（与LTP并行）

> 目标：在持续推进 LTP 通过率的同时，逐步把系统演进为更接近 Linux 语义和工程质量的 OS。
> 执行原则：遵循 `70% LTP 推进 + 30% 架构治理`，避免 case-specific hack 固化。

## 当前架构速览

- 启动链路：`main.rs` 完成 `mm/log/trap/time/fs/task` 初始化后进入调度。
- 核心调度：`task/processor.rs` + `task/manager.rs`（每 hart processor + 全局任务管理器）。
- 存储与文件：`ext4-fs` + `fs/*` + `syscall/filesystem.rs`（syscall 层逻辑较重）。
- 内存管理：`mm/*` 负责页表、COW、lazy fault、用户拷贝等。

## 架构级风险清单（优先级从高到低）

1. 全局 ext4 大锁导致 SMP 扩展性与卡顿风险  
   - 现象：`ext4_lock()` 为全局串行锁，所有 ext4 操作互斥。  
   - 影响：多核并发文件操作吞吐受限，锁竞争路径复杂时可能放大阻塞。  
   - 方向：按 inode/目录/元数据热点拆锁；高冲突路径改为可睡眠等待而非忙等。

2. `/proc` 当前实现与真实 procfs 语义耦合不清  
   - 现象：`procfs` 初始化/同步路径与 ext4 目录实体有耦合。  
   - 影响：虚拟文件系统边界模糊，后续权限/一致性/维护成本升高。  
   - 方向：把 `/proc` 明确为纯内存 pseudo-fs（由进程状态实时生成内容），减少对磁盘实体依赖。

3. syscall 期间可抢占性不足（中断策略偏保守）  
   - 现象：trap/syscall 路径为了规避锁问题，对中断开放较保守。  
   - 影响：长 syscall 的延迟上升，影响实时性与系统响应。  
   - 方向：分层梳理“可中断/不可中断”临界区；为高频路径提供中断安全内存/队列机制。

4. `exec` 中存在 glibc 私有偏移修补逻辑（脆弱）  
   - 现象：依赖特定动态链接器内部布局的魔法偏移。  
   - 影响：glibc 版本/构建差异时易失效，长期不可维护。  
   - 方向：回到 ELF/auxv/ld.so 规范语义修复，移除私有结构补丁。

5. 调度路径存在全局锁竞争热点  
   - 现象：任务入队/取队依赖全局任务管理器锁。  
   - 影响：核数升高后锁竞争明显，吞吐和尾延迟恶化。  
   - 方向：每 hart 就绪队列细粒度化 + 轻量 steal 策略，降低全局锁占用。

6. stat/statx 可见大小语义实现有全局扫描成本  
   - 现象：为覆盖未 flush 缓冲可见性，引入跨进程 fd 扫描。  
   - 影响：路径查询类 syscall 开销上升，且与 fs 锁路径耦合。  
   - 方向：建立 inode 级“脏写上界”元数据缓存，替代全表扫描。

## 近期并行治理记录

- `mkdir03` 超时根因：`syscall_mkdirat` 使用 `translated_str()` 读取用户路径，遇到坏地址会直接杀进程，偏离 Linux `EFAULT` 语义。  
  已改为 `read_user_cstring()`，并正确返回 `EFAULT/ENAMETOOLONG`。
- `mkdir03` 语义偏差：`mkdir("/dev/null")` 之前返回 `EROFS`，与 Linux 期望 `EEXIST` 不一致。  
  已在 `syscall_mkdirat` 的 pseudo 路径分支中对“目标已存在”返回 `EEXIST`，仅对不存在的 pseudo 目标返回 `EROFS`。
- `link08/linkat01` 跨挂载语义偏差：旧实现在部分跨挂载 hardlink 场景返回 `EROFS` 或直接成功。  
  已在 `syscall_linkat` 中补齐“跨挂载优先 `EXDEV`”判定，并区分 pseudo 源路径（返回 `EXDEV`）与 pseudo 目标路径（返回 `EROFS`）。
- `symlinkat01` 路径解析偏差：相对符号链接目标包含 `..` 时，解析无法越过“起始目录”，导致跟随链接时错误 `ENOENT`。  
  已在 `resolve_ext4_path()` 中增强 `..` 处理：当栈深为 1 时通过目录 `..` 项向上回溯，确保相对链接按 Linux 语义解析。
- `readlink03/readlinkat02` 参数语义偏差：`bufsiz == 0` 时旧实现返回成功。  
  已在 `syscall_readlinkat()` 统一返回 `EINVAL`，与 Linux/LTP 预期一致。
- `readlinkat01` 空路径语义缺失：旧实现对空 pathname 一律 `ENOENT`，不支持“按 `dirfd` 指向的符号链接读目标”。  
  已补齐空 pathname 分支（基于 fd 直接读取 symlink 目标），并配套修正 `openat(O_PATH|O_NOFOLLOW)` 的终端符号链接处理。
- `close_range02` 缺失：内核此前未接入 `close_range(436)`，导致 `ENOSYS`。  
  已在 syscall 分发层接入 `close_range`，并实现 Linux 兼容的核心语义：参数校验（`first > last`、非法 flags 返回 `EINVAL`）、区间 close、`CLOSE_RANGE_CLOEXEC` 标记设置。
- fd 表治理（30% lane）：`fcntl/close/close_range/dup/dup3/pipe2` 里存在重复的 `fd_flags` 对齐、fd 有效性判断、slot 清理逻辑。  
  已下沉到 `ProcessControlBlockInner::{ensure_fd_flags_len,is_fd_open,clear_fd}`，减少重复分支与维护点，保持行为不变并便于后续 `CLONE_FILES` 语义演进。
- `open13`（`O_PATH`）语义偏差：`fchmod/fchown` 在 `O_PATH` fd 上曾错误成功，`fgetxattr` 缺失且返回 `ENOSYS`。  
  已补齐 fd 级别 `EBADF` 语义，并接入 `fgetxattr(10)` 分发；同时修复 musl 在 `fchmod/fchown` 遇到 `EBADF` 后经 `/proc/self/fd/N` 回退到 `fchmodat/fchownat` 的路径，避免误报 `ENOENT`。
- 测试编排治理：将 open 子组按语义风险拆分（`OPEN_ADV=open13`、`OPEN_LARGEFILE=open12`、`OPEN_TMPFILE=open14`），默认回归只保留稳定子组；新增并验证 `ACCESS_TASKS` 与 `CLOSE_TASKS`，保持“通过率推进”与“语义正确性”并行。
- `umask` 语义治理：此前 `umask` 使用全局原子变量，导致跨进程状态泄漏（非 Linux 语义），在长回归链中会放大权限类测试偶发风险。  
  已改为进程级状态（PCB `inner.umask`），并在 `fork` 继承、`exec` 保持；`current_umask/syscall_umask` 改为访问当前进程字段，文件创建路径通过 `apply_umask()` 自动获得正确语义。
- LTP 推进：在稳定基线上继续扩展并通过 `FACCESSAT_TASKS`、`DUP_CORE_TASKS`、`DUP_FCNTL_TASKS`、`CWD_DIR_TASKS`、`UMASK_TASKS`、`CHMOD_TASKS`、`CHOWN_TASKS`（musl+glibc），同时保留高风险 `openat`/`O_TMPFILE` 子组隔离策略。
- `openat03`（`O_TMPFILE`）语义治理：旧实现依赖目录 `..` 逐级回溯定位临时池目录，导致在多目录场景出现残留隐藏目录/文件，触发 `rmdir(...): ENOTEMPTY`。  
  已改为按 `device_id` 选择对应文件系统根 inode，统一在同一 fs 的隐藏池创建匿名临时文件，并在 fd drop 时清理；`openat03` 已在 musl+glibc 通过。
- LTP 扩展：在现有稳定组上新增并验证 `POSIX_TIMER_TASKS`、`CLOCK_GETTIME_TASKS`、`CLOCK_RES_TASKS`、`CLOCK_SETTIME_TASKS`、`GETITIMER_TASKS`、`SETITIMER_TASKS`、`TIME_MISC_TASKS`、`GETRUSAGE_TASKS`（musl+glibc 全通过）。  
  `openat04` 仍因环境侧 `tst_acquire_device` 失败（`TBROK: Failed to acquire device`）保留在独立设备组，不纳入默认稳定回归。
- LTP 继续推进：新增并稳定通过 `MKNODAT_TASKS`、`MKNOD_CORE_TASKS`、`CREAT_SGID_TASKS`、`SYMLINK_LEGACY_TASKS`（musl+glibc）。  
  `creat07` 在当前镜像中因缺少资源 `creat07_child` 触发 `TBROK`，已明确保持在独立非默认组，避免将环境噪声误判为内核语义回归。
- LTP 继续推进（并发/随机语义）：新增并稳定通过 `GETRANDOM_TASKS` 与 `ROBUST_TID_TASKS`（含 `get_robust_list01`，musl+glibc）。  
  这部分已并入默认稳定回归链，用于持续覆盖随机源行为与 robust-list/tid-address 语义。
- LTP 继续推进（凭据细节语义）：新增并稳定通过 `CRED_FS_TASKS`（`setfsuid/setfsgid`）与 `CRED_EGID_TASKS`（musl+glibc），并纳入默认稳定回归。  
  该扩展提升了权限路径与 fsuid/fsgid 生效链路的长期回归覆盖。
- 设备依赖子组推进（第一阶段）：在 `submit_script` 为 LTP 用例注入 `LTP_DEV=/dev/root` 与 `LTP_SINGLE_FS_TYPE=tmpfs`，并在内核 `ioctl` 中补齐 `/dev/root` 的 `BLKGETSIZE64/BLKGETSIZE/BLKSSZGET/BLKPBSZGET`（含 musl 的符号扩展请求兼容）。  
  同时补齐 `/proc/sys/kernel/tainted`（返回 `0`）与 `capget/capset(90/91)`，并完成 `CLONE_FILES` 共享 fd-table 基础语义：  
  - `clone(CLONE_FILES)` 的 fork-like 路径共享文件表所有者；  
  - `close_range(..., CLOSE_RANGE_UNSHARE)` 会先私有化当前进程 fd-table，再执行 close/cloexec；  
  - 相关 fd 访问路径（`fcntl/dup/dup3/close/close_range/openat/pipe2` 及常用 fd helper）已切到共享文件表视角；  
  - owner 退出时增加共享 fd-table 交接：迁移到存活 sharer 并重绑定其余 sharer，避免 `CLONE_FILES` 场景因 owner 退出导致共享 fd 状态丢失。  
  同时接入 `unshare(97)` 的最小 Linux 语义（`CLONE_FILES` 实际解绑、`CLONE_FS` no-op 成功、`CLONE_NEWNS` root-only no-op），`unshare01/unshare02` 已并入默认回归并稳定通过（musl+glibc）。
  结果：`chdir01` 与 `close_range01` 均已稳定并入默认回归（musl+glibc）。
- LTP 提效推进（单轮 10+ 子组）：新增并稳定并入 13 个进程组/优先级相关用例（`setpgid01/02/03`、`setsid01`、`setpgrp01/02`、`getpgrp01`、`gettid01/02`、`getpriority01/02`、`setpriority01/02`，musl+glibc）。  
  该轮提升了作业控制、tid 一致性与 nice/priority 路径的持续覆盖密度，满足“每轮新增 10-20 tests”节奏。
- LTP 提效推进（单轮 ~50 tests）：在默认稳定链新增并入凭据/UTS/资源限制/告警子组：`GETRES_TASKS(6)`、`CRED_SET_TASKS(26)`、`UTS_NAME_TASKS(6)`、`UTS_QUERY_TASKS(2)`、`SETRLIMIT_TASKS(6)`、`GETRLIMIT_TASKS(3)`、`ALARM_TASKS(5)`，并追加 glibc 边界用例 `gethostname02`（总计 55 个用例名）。  
  本轮按“相关回归优先”仅针对该批次执行回归验证，musl+glibc 路径均为 `FAIL LTP CASE ... : 0`，用于在保证语义稳定的同时提高吞吐。
- poll/epoll 语义治理：此前 `ppoll/pselect/epoll` 分散在 syscall 层各自计算 readiness，缺少统一的 Linux poll mask 语义，`epoll_pwait2(441)` 也未接入。
  已将 `poll_mask()/supports_poll()` 下沉到 `File` 抽象，补齐 pipe/socket/FIFO/pidfd/userfaultfd 的 `POLLHUP/POLLERR/POLLRDHUP` 可见性；`ppoll` 现在对 `sigmask` 正确忽略被屏蔽信号，`epoll` 接入 `epoll_pwait2` 并统一复用等待/信号掩码逻辑。
  结果：`epoll_ctl01-05`、`epoll_wait01-07`、`epoll_pwait01-05` 已在 musl+glibc 聚焦回归中稳定通过；并且 `epoll_wait` 已为 pipe、FIFO、socketpair、Unix socket、netlink、loopback 网络 socket、`pidfd`、`userfaultfd` 接入真实 waiter 注册和事件唤醒，避免无限等待路径继续只靠短睡眠轮询。
  2026-03-10 又在 riscv64 上完成聚焦复验：`pidfd_open01-04`、`userfaultfd01` 均通过；`pidfd_getfd01-02` 与 `pidfd_send_signal01-03` 仍是架构级 `TCONF`，因为 riscv64 LTP 缺少对应 syscall 号，不属于 readiness 回归。
  同日继续补齐了 `UnixSocketFile` stream 建链态的 waiter 注册，以及 `NetSocketFile` 在 `listen/accept/shutdown_read/connect` 等本地状态迁移上的主动唤醒；聚焦 `select/poll/epoll` 回归继续保持通过。
  同时新增了用户态 `nested_epoll_smoke` 手工回归，覆盖 `parent epoll -> child epoll -> pipe` 的基础嵌套链路，并确认读空后仍因 `EPOLLHUP` 保持 ready 的 Linux 语义；该 smoke 已在 riscv64 shell 镜像手工通过。
  2026-03-11 继续收口 `epoll_wait` 自身的 waiter 路径：无限等待现在会通过 `EpollFile::register_poll_waiter_internal()` 同时挂上 epoll 自身与子对象等待队列，覆盖空 epoll / `epoll_ctl` 变更唤醒场景；配套新增 `epoll_ctl_wakeup_smoke`，在 riscv64 shell 镜像上验证“先阻塞于空 epoll，再由另一进程 `EPOLL_CTL_ADD` 一个已 readable 的 pipe fd”能够正确唤醒返回。
  同日又把 `eventfd` 从 dummy fd 提升为真实计数对象：补齐计数器读写、阻塞/非阻塞等待、poll mask 与 poll waiter 唤醒链路，并新增 `eventfd_epoll_smoke` 验证 `EPOLLOUT -> EPOLLIN -> EPOLLOUT` 的 readiness 迁移；`eventfd01-06`、`eventfd2_01-03` 也在 musl+glibc 聚焦回归中继续保持通过。
  同日还收口了 `timerfd` 批次：`timerfd01-02`、`timerfd_create01`、`timerfd_gettime01`、`timerfd_settime01-02` 在 musl+glibc 聚焦回归中继续通过，`timerfd04` 则稳定表现为 `/proc/config.gz` 缺少 `CONFIG_TIME_NS=y` 的配置级 `TCONF`。其中 `timerfd_settime02` 是带 `.max_runtime = 150` 的 race-stress 用例，预期行为就是持续打满压力窗口后再 `TPASS`，因此后续应把它视为稳定性/长期运行观察点，而不是短时结束型功能回归。为避免 yield-heavy LTP 路径长期只靠 tick 做粗粒度统计，调度器现在也在 schedule-in / yield / block / exit / tick 这些共享边界上统一结算线程 CPU runtime，并让 `CLOCK_*CPUTIME*`、`getrusage()`、legacy cpu cgroup 统计共享同一套累计值。
  同日继续清理 `timerfd` 内部结构：把常态 tick 路径改为按 monotonic/realtime deadline heap 只处理最早到期的一批 timerfd，并用 schedule generation 失效旧条目；`clock_settime/settimeofday` 也直接复用 realtime heap 收集 live timerfd。这样 `timerfd` 不再依赖单独的全局 `Weak<TimerFdFile>` 扫描表，同时保住 `read/gettime/poll/settime` 主动推进 deadline 后的重装语义。
  同时也把 POSIX timer 的 wall-clock 路径收成了 monotonic/realtime deadline heap：`timer_settime/gettime/getoverrun/delete` 相关 LTP 用例在 musl+glibc 聚焦回归中继续通过，而 `CLOCK_PROCESS_CPUTIME_ID` / `CLOCK_THREAD_CPUTIME_ID` 仍保留按 CPU-time 动态判断的扫描兜底，避免一次性把两类时钟语义混在一起重写。后续又给这两条 heap 增加了 live armed 计数，避免旧 schedule 条目残留时仅凭 `heap.is_empty()` 误判“还有 pending timer”，把 idle 路径长期拖在自旋轮询里；同时把 timer 实体管理收成 `Vec + (pid,timer_id)->index` 索引，避免 wall-clock heap 命中后再退回一次全表线性查找。对 `CLOCK_*CPUTIME*` 这条扫描兜底，也进一步按 `pid` / `(pid,tid)` 分桶，只对 bucket 级最早 deadline 已到的对象取一次当前 CPU time，再扫描这个 bucket 内的 timer。
  `MqDescriptor` 当前也已暴露 `poll_mask()` / `register_poll_waiter()` 并在消息入队/出队时主动唤醒 poll waiters，为 POSIX MQ 后续 epoll 覆盖和 targeted regression 做好基础设施。
  2026-03-14 又补了一组 POSIX MQ readiness follow-up：新增 `mq_epoll_smoke`、`mq_unlink_epoll_smoke`、`mq_notify_signal_smoke` 三个用户态 smoke，在 riscv64 shell 镜像上确认 `mqueue -> epoll` 可见性、`unlink` 后存活队列的 waiter 语义，以及 `mq_notify(SIGEV_SIGNAL)` 的首条消息触发/一次性注册/`EBUSY`/取消注册语义都能稳定通过；同时也修正了用户态 `sigaction` wrapper，使其按 `rt_sigaction` ABI 传递 sigset 大小，避免手工 smoke 与 Linux 用户态接口脱节。
  同日继续沿着嵌套等待拓扑补覆盖：新增 `nested_epoll_ctl_wakeup_smoke`，在 riscv64 shell 镜像上验证“父 epoll 阻塞等待子 epoll，另一进程通过 `EPOLL_CTL_ADD` 把一个已 readable 的 pipe fd 动态加入子 epoll”能够跨层唤醒父 epoll 返回。这样 `nested_epoll_smoke` 的基础链路与 `epoll_ctl_wakeup_smoke` 的 ctl-driven 唤醒路径就被串到了同一条嵌套拓扑里。
  同时又补了 `nested_epoll_ctl_del_smoke`，覆盖删除路径的 ready 去抖：当 `pipe -> child epoll -> parent epoll` 链路已经处于可读状态时，若对子 epoll 执行 `EPOLL_CTL_DEL` 移除 pipe interest，父 epoll 不应继续保留陈旧 ready；重新 `EPOLL_CTL_ADD` 后，未消费的数据又应立即重新变成 ready。
  同时再补了 `nested_epoll_oneshot_smoke`，确认 `EPOLLONESHOT` 在 `pipe -> child epoll -> parent epoll` 链路上仍保持 Linux 风格的一次性触发语义：首次可读会穿透到父 epoll，第二次写入在未 rearm 前不会重复上报，只有对子 epoll 执行 `EPOLL_CTL_MOD` 重新 arm 后才再次把 ready 传播回父 epoll。
  同日继续补上 `nested_epoll_et_smoke`，把 `EPOLLET` 也纳入 `pipe -> child epoll -> parent epoll` 拓扑覆盖：首次写入后父/子 epoll 都只上报一次，读空后 readiness 归零，第二次写入再触发新的 edge。这个 smoke 同时暴露出内核里一个 ET bookkeeping 缺口：过去 `epoll_wait` 只有在返回 ready events 时才刷新 `last_ready`，导致 ready -> not-ready -> ready 的下一条边可能丢失；现已改为即使本轮返回空事件也同步 apply updates，避免嵌套 ET 链路错过后续 edge。
  同日又补了 `nested_epoll_et_maxevents_smoke`，覆盖 `maxevents` 截断下的嵌套 ET 收口：当父 epoll 只返回本轮前一个 ready child 时，后续 child interest 即使没被返回，也必须同步刷新 `last_ready`。否则另一个 child epoll 在经历 `ready -> not-ready -> ready` 后会因为陈旧快照错过下一条 edge。对应内核实现现已改为“事件返回数可截断，但 updates 仍遍历全量 interest”，把 batch 截断与内部 ET bookkeeping 解耦。
  同日还补了 `nested_epoll_parent_oneshot_smoke`，把 `EPOLLONESHOT` 提到 `parent epoll -> child epoll` 这一层做单独覆盖：首次可读穿透到父 epoll 后，父侧 interest 必须进入 one-shot disabled 状态；即便 child epoll 先回到 not-ready、再重新 ready，父侧在未 rearm 前也不能重复上报，直到对父侧 interest 执行 `EPOLL_CTL_MOD` 才恢复一次新的传播。这样父层 one-shot 与子层 one-shot 语义就被拆开覆盖，不再混在同一条 smoke 里。
  当前剩余长期缺口是缺少覆盖全部 `File` 类型的通用 wait-queue / event registration，以及嵌套 `epoll` 更复杂 corner case 的系统化覆盖；在这些对象补齐前，`epoll_wait` 仍对 mixed-support 场景保留短睡眠轮询兜底。
- readiness 回归收口：`select01-04`、`poll01-02`、`ppoll01`、`pselect01-03` + `_64`、`epoll_create01-02`、`epoll_create1_01-02`、`epoll-ltp` 已在 musl+glibc 聚焦回归中稳定通过。  
  其中 `__NR_select`、`__NR__newselect`、独立 `pselect6_time64`、`__NR_epoll_create` 在 riscv64 上属于架构级不存在接口，LTP 对这些变体给出 `TCONF`，与 Linux/riscv64 预期一致，不视为内核语义缺口。
- 后续治理项：继续清点仍使用 `translated_str()` 读取路径参数的 syscall，统一迁移到可返回 errno 的安全读取 helper，减少“异常退出替代 errno”的风险。  
  fd-table 共享语义已覆盖 `close_range01` 主路径，下一步可继续把当前“owner+sharer 重绑定”模型演进为独立 files-struct 引用计数对象，进一步贴近 Linux 生命周期语义。

## 分阶段改进计划（建议）

### 阶段 A（立即执行，低风险高收益）

- 固化边界：新增文档和代码注释，明确 fs/procfs/调度/中断边界。
- 抽公共逻辑：继续把 `syscall/filesystem.rs` 中可复用逻辑下沉（权限、errno 映射、路径解析）。
- 约束调试开关：默认关闭噪声日志，保留按需开关，不引入常开 debug 输出。

### 阶段 B（与 LTP 子组并行推进）

- 每完成 2-3 个 LTP 子组，安排 1 次“只做治理”的迭代：
  - 拆分 syscall 大文件中的子模块；
  - 降低全局锁覆盖范围；
  - 清理临时兼容逻辑并补回标准语义。

### 阶段 C（中期目标）

- procfs/pseudo-fs 完整内存化和接口分层。
- 调度器并发模型优化（减少全局锁热点）。
- exec/动态链接语义修复，移除私有偏移补丁。

## 执行检查清单（每轮）

- 本轮目标 LTP 子组是否通过（musl + glibc）？
- 是否引入了新的全局锁/全局扫描？
- 是否新增了 case-specific hack？若有，是否已列出移除计划？
- 是否更新了本路线图与对应 `OSGuide/parts/*` 记录？

---

维护约定：此文档用于“未来改进的导航页”，每次完成关键架构治理后都应同步更新。
