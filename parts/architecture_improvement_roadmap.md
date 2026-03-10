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
