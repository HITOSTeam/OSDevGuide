# 代码结构审计报告

> 审计时间：2026-04-13  
> 审计范围：`os/src/` 全量 + exampleOs 三个参考实现对比  
> 参考实现：arceos、starry-mix、oskernel2025-rocketos

---

## 一、问题清单

### 🔴 严重问题

#### 1. `syscall/filesystem.rs` — 单文件 10930 行

- **286 个函数**全部堆在一个文件，涵盖 open/read/write/mkdir/stat/mount/xattr/fallocate 等所有文件系统 syscall
- 无任何子系统分层，无法按功能独立阅读或维护
- 与 `fs/`、`task/`、`mm/` 多达 15 个模块直接耦合
- 错误码以 `const EBADF: isize = -9` 形式散落在文件顶部（40+ 个）

其余 syscall 文件也偏大：

| 文件 | 行数 |
|---|---|
| `syscall/filesystem.rs` | **10930** |
| `syscall/misc.rs` | 2958 |
| `syscall/net.rs` | 2787 |
| `syscall/process.rs` | 2372 |
| `syscall/signal.rs` | 1642 |
| `syscall/memory.rs` | 1597 |
| `syscall/time_sys.rs` | 1338 |
| `syscall/sysv_ipc.rs` | 1334 |

#### 2. `ProcessControlBlockInner` — 115 个字段压一把 `SpinMutex`

- **所有进程状态**（身份/文件表/信号/内存/namespace/调度/SysV IPC/ptrace）全部在同一把自旋锁下
- SMP 场景下任何 syscall 都可能竞争同一把锁，严重影响并发吞吐
- `process_block.rs` 共 2152 行

#### 3. 无统一错误类型

- 每个 syscall 文件各自定义相同的 `const EBADF: isize = -9` 等常量
- syscall 返回值统一是裸 `isize`，无类型安全保障
- 错误语义（如什么场景返回 `EFAULT` vs `EINVAL`）散落各处，难以统一审查

---

### 🟡 中等问题

#### 4. `fs/procfs.rs` 2492 行 / `fs/cgroupfs.rs` 2276 行

- 所有 `/proc` 和 `/sys/fs/cgroup` 的虚拟文件逻辑堆在单文件
- 与 `fs/pseudo.rs` 的 trait 抽象方向一致，但实现层没有按路径/功能拆分子模块

#### 5. `fs/dummy.rs` — 多种不相关文件类型混在一起

- `DummyFile`、`EventFdFile`、`TimerFdFile`、`NamespaceFile`、`PidFdFile`、`UserfaultfdFile` 都在同一个 1145 行文件里
- 每种文件类型应独立成文件

---

### 🟢 结构较好的部分

- **`mm/`** — 分层清晰，各文件职责单一（address / frame_allocator / heap / memory_set / page_table）
- **`fs/` 的 `File` trait 抽象** — 方向正确，trait 接口定义合理
- **`utils/`** — 精简，`RecycleAllocator` 设计合理

---

## 二、参考实现对比

| 维度 | arceos | **starry-mix** | oskernel2025 | CongCore 现状 |
|---|---|---|---|---|
| syscall 文件大小 | ~200 行/文件 | **100–300 行/文件** | 几万行/文件 | 最大 10930 行 |
| PCB 字段数 | ~10（极简） | ~50（分层） | ~300 | **115（单锁）** |
| 文件抽象 | `VfsNodeOps` trait | `FileLike` trait | `FileOp` trait | `File` trait ✓ |
| 子系统分层 | 15 个独立 crate | syscall 按语义分目录 | 无 | 无 |
| 错误类型 | `LinuxError` enum | `LinuxError` enum | 裸 isize | **裸 isize** |

**最佳参考：starry-mix** — 在保持 Linux 兼容性的同时，syscall 按语义子目录组织，文件大小可控，PCB 分层，错误类型统一。

### starry-mix 的 syscall 组织方式（目标参考）

```
api/src/syscall/
├─ fs/
│  ├─ fd_ops.rs   # dup, fcntl, close
│  ├─ io.rs       # read, write, pread, pwrite, readv, writev
│  ├─ ctl.rs      # ioctl, chdir, fchdir, sync
│  ├─ stat.rs     # stat, fstat, statx, xattr
│  ├─ mount.rs    # mount, umount, fsopen, fsmount
│  ├─ pipe.rs
│  └─ dir.rs      # mkdir, rmdir, getdents64
├─ io_mpx/        # select, poll, epoll
├─ net/
├─ mm/
└─ task/
```

---

## 三、改进优先级

### P0 — 代码组织（最高优先，影响所有后续维护）

1. **拆分 `syscall/filesystem.rs`** → 按语义建子目录 `syscall/fs/`，每文件 200–500 行
2. **建立统一 `SyscallError` 枚举** → 替换散落的 `isize` 常量，提供类型安全的错误转换

### P1 — 锁粒度

3. **`ProcessControlBlockInner` 字段分组** → 按功能域拆出子结构（identity / files / signals / memory / namespaces），各自独立加锁

### P2 — 虚拟文件系统

4. **`fs/procfs.rs` 目录化** → 拆成 `fs/procfs/` 子模块（stat.rs / fd.rs / task.rs 等）
5. **`fs/cgroupfs.rs` 目录化** → 同上
6. **`fs/dummy.rs` 拆分** → 每种文件类型独立文件

### P3 — 长期架构

7. 参考 arceos 的模块化思路，把 `ext4-fs`、`fs/`、`mm/` 逐步向独立 crate 演进
8. 参考 starry-mix 的 `FileLike` 模式，增强 `File` trait 的可组合性

---

## 四、相关文件

- `OSGuide/parts/architecture_improvement_roadmap.md` — 现有架构风险清单
- `OSGuide/roadmap.md` — 当前阶段目标
- `exampleOs/starry-mix/` — 最佳参考实现
- `exampleOs/arceos/` — 模块化设计参考
