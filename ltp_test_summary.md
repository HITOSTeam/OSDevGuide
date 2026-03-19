# LTP Test Suite — Classification Summary

> Source: `ltp_all.md` (2822 entries total)
> Last updated: 2026-03-19

This document categorises every entry in `ltp_all.md` into functional groups.
The count in each section is approximate (some tests appear in multiple
contexts; script helpers and data files are included in the raw count).

---

## 1. Process Lifecycle (~80)

Creation, execution and termination of processes.

| Prefix / Group            | Tests                                                                                     |
| ------------------------- | ----------------------------------------------------------------------------------------- |
| `fork`                    | fork[01-14]                                                                               |
| `vfork`                   | vfork, vfork[01-02]                                                                       |
| `clone`                   | clone[01-09], clone3[01-03]                                                               |
| `exec*`                   | execl01, execle01, execlp01, execv01, execve[01-06], execveat[01-03] ✅ (`execveat03` env `TCONF`), execvp01 |
| `exit`                    | exit[01-02], exit_group01                                                                 |
| `wait / waitpid / waitid` | wait[01-02], wait4[01-03], waitpid[01-13], waitid[01-11]                                  |

✅ 2026-03-09 core100 process follow-up passed (`clone05-07`, `wait403`,
`execve05-06`, `execveat03`).

✅ 2026-03-19 files-lifecycle follow-up passed on musl+glibc
(`close_range02`, `unshare01-02`, `vfork`, `vfork01-02`, `execl01`,
`execle01`, `execv01`, `execve01-04`, `execveat01-02`; `execveat03`
remains environment-driven `TCONF` on overlayfs coverage).

✅ 2026-03-10 pidfd readiness follow-up passed on riscv64 (`pidfd_open01-04`);
`pidfd_getfd01-02` and `pidfd_send_signal01-03` remain arch-specific `TCONF`
because riscv64 LTP lacks those syscall numbers, not because of kernel
readiness regressions.

---

## 2. Process Identity & Credentials (~250)

UID/GID queries and mutations, group lists, filesystem credentials,
capability sets.

| Group                                   | Tests                                                                              |
| --------------------------------------- | ---------------------------------------------------------------------------------- |
| `getpid / getppid`                      | getpid[01-02], getppid[01-02]                                                      |
| `gettid`                                | gettid[01-02]                                                                      |
| `getpgid / getpgrp / getsid`            | getpgid[01-02], getpgrp01, getsid[01-02]                                           |
| `setpgid / setsid / setpgrp`            | setpgid[01-03], setsid01, setpgrp[01-02]                                           |
| `getuid / getgid / geteuid / getegid`   | getuid[01,03], getgid[01,03], geteuid[01-02], getegid[01-02] + `_16` variants      |
| `getresuid / getresgid`                 | getresuid[01-03], getresgid[01-03] + `_16` variants                                |
| `setuid / setgid / setreuid / setregid` | setuid[01,03-04], setgid[01-03], setreuid[01-07], setregid[01-04] + `_16` variants |
| `setresuid / setresgid`                 | setresuid[01-05], setresgid[01-04] + `_16` variants                                |
| `setfsuid / setfsgid`                   | setfsuid[01-04], setfsgid[01-02] + `_16` variants                                  |
| `setegid`                               | setegid[01-02]                                                                     |
| `getgroups / setgroups`                 | getgroups[01,03], setgroups[01-04] + `_16` variants                                |
| `capget / capset`                       | capget[01-02], capset[01-04]                                                       |
| `cap_bounds`                            | cap_bounds_r, cap_bounds_rw, cap_bset_inh_bounds                                   |
| `check_*`                               | check_keepcaps, check_simple_capset, check_pe                                      |

✅ Credential extended batches passed (`getresuid[01-03]`, `getresgid[01-03]`,
`setuid01`, `setuid03-04`, `setgid01-03`, `setreuid01-07`, `setregid01-04`,
`setresuid01-05`, `setresgid01-04`, `setfsuid01-04`, `setfsgid01-02`, `setegid01-02`).

✅ 2026-03-09 credential follow-up passed (`getgroups01`, `setgroups01-02`).

---

## 3. Scheduling & Priority (~30)

| Group                       | Tests                                                                                                                                                                                                                                                               | Status |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| `nice`                      | nice[01-05]                                                                                                                                                                                                                                                         | ✅     |
| `getpriority / setpriority` | getpriority[01-02], setpriority[01-02]                                                                                                                                                                                                                              | ✅     |
| `sched_get/set*`            | sched_getaffinity01, sched_setaffinity01, sched_getattr[01-02], sched_setattr01, sched_getparam[01,03], sched_setparam[01-05], sched_getscheduler[01-02], sched_setscheduler[01-04], sched_rr_get_interval[01-03], sched_get_priority_max/min[01-02], sched_yield01 | ✅     |
| stress                      | sched_tc[0-6], sched_stress.sh                                                                                                                                                                                                                                      | ✅ `sched_tc[0-6]` (NEXT100 sched) |

---

## 4. Signals (~60)

| Group                      | Tests                                                                   |
| -------------------------- | ----------------------------------------------------------------------- |
| `signal / sigaction`       | signal[01-06], sigaction[01-02], rt_sigaction[01-03]                    |
| `sigprocmask / sigpending` | sigprocmask01, rt_sigprocmask[01-02], sigpending02                      |
| `sigsuspend / sigwait`     | sigsuspend01, rt_sigsuspend01, sigwait01, sigwaitinfo01, sigtimedwait01 |
| `sigaltstack`              | sigaltstack[01-02]                                                      |
| `sighold / sigrelse`       | sighold02, sigrelse01                                                   |
| `signalfd`                 | signalfd[01-06] (✅ 01), signalfd4\_[01-02]                             |
| `kill / tgkill / tkill`    | kill[02-13], tgkill[01-03], tkill[01-02]                                |
| `rt_sigqueueinfo`          | rt_sigqueueinfo01                                                       |
| `pause`                    | pause[01-03]                                                            |
| `alarm`                    | alarm[02,03,05-07]                                                      |

✅ NEXT100 signal batch passed (`kill02/03/05-13`, `pause01-03`, `rt_sigprocmask01-02`,
`rt_sigqueueinfo01`, `rt_sigsuspend01`, `sighold02`, `signalfd4_01-02`, `tgkill01-03`, `tkill01-02`).

✅ 2026-03-09 core100 signal batch passed (`sigaction01-02`, `rt_sigaction01-03`,
`sigaltstack01-02`, `sigpending02`, `sigprocmask01`, `sigrelse01`,
`sigsuspend01`, `sigtimedwait01`, `sigwait01`, `sigwaitinfo01`, `signal01-05`).

---

## 5. File System — Basic Syscalls (~350)

### 5.1 Open / Create / Close

| Group                 | Tests                                      |
| --------------------- | ------------------------------------------ |
| `open`                | open[01-14], openat[01-04], openat2[01-03] ✅ 2026-03-08 0305 follow-up (`open12`, `open14`, `openat02`, `openat04`, `openat2[01-03]`) |
| `creat`               | creat[01,03-09] ✅ 2026-03-08 (`creat07`, `creat09`) |
| `close / close_range` | close[01-02], close_range[01-02]           |

### 5.2 Read / Write

| Group            | Tests                                                                                       | Status              |
| ---------------- | ------------------------------------------------------------------------------------------- | ------------------- |
| `read / write`   | read[01-04], write[01-06], readv[01-02], writev[01-07]                                      | ✅ Batch3 core pass |
| `pread / pwrite` | pread[01-02], pwrite[01-04], preadv[01-03], pwritev[01-03], preadv2[01-03], pwritev2[01-02] | ✅ NEXT100 fs-io + fs-io-v2 |
| `lseek / llseek` | lseek[01-02,07,11], llseek[01-03]                                                           | ✅ Batch3 core pass |

### 5.3 File Descriptors

| Group               | Tests                                  | Status |
| ------------------- | -------------------------------------- | ------ |
| `dup / dup2 / dup3` | dup[01-07], dup2[01-07], dup3\_[01-02] | ✅ 2026-03-09 core100 (`dup01-07`, `dup201-207`, `dup3_01-02`) |
| `fcntl`             | fcntl[01-39] + `_64` variants          | ✅ FS100 batch1-4 pass |

### 5.4 File Metadata / Permissions

| Group                            | Tests                                                                             | Status |
| -------------------------------- | --------------------------------------------------------------------------------- | ------ |
| `stat / lstat / fstat / fstatat` | stat[01-03], lstat[01-02], fstat[02-03], fstatat01, statx[01-12] + `_64` variants | ✅ A-batch core (`*_64`, `statx04-12`) |
| `statfs / fstatfs`               | statfs[01-03], fstatfs[01-02] + `_64` variants                                    | ✅ A-batch `_64` set |
| `chmod / chown`                  | chmod[01,03,05-07], chown[01-05] + `_16` variants                                 | ✅ core + `_16` (`chmod*`, `chown[01-05]`, `chown[01-05]_16`) |
| `fchmod / fchown`                | fchmod[01-06], fchmodat[01-02], fchown[01-05], fchownat[01-02] + `_16` variants   | ◐ `fchmod[01-06]`, `fchmodat[01-02]` ✅; `fchown[01-05]_16` ✅ |
| `lchown`                         | lchown[01-03] + `_16` variants                                                    | ✅ `lchown[01-03]`, `lchown[01-03]_16` |
| `access / faccessat`             | access[01-04], faccessat[01-02], faccessat2[01-02]                                | ✅ core (`access*`, `faccessat01-02`) |
| `umask`                          | umask01                                                                           | ✅ 2026-03-09 core100 |

### 5.5 Directories

| Group                     | Tests                                         | Status |
| ------------------------- | --------------------------------------------- | ------ |
| `chdir / fchdir / getcwd` | chdir[01,04], fchdir[01-03], getcwd[01-04]    | ✅ |
| `chroot`                  | chroot[01-04]                                 | ✅ 2026-03-09 core100 |
| `mkdir / mkdirat / rmdir` | mkdir[02-05,09], mkdirat[01-02], rmdir[01-03] | ✅ |
| `getdents`                | getdents[01-02]                               | ✅ |
| `readdir`                 | readdir01, readdir21                          | ✅ A-batch |

### 5.6 Links

| Group                                         | Tests                                                           | Status |
| --------------------------------------------- | --------------------------------------------------------------- | ------ |
| `link / linkat`                               | link[02,04-05,08], linkat[01-02]                                | ✅ |
| `symlink / symlinkat / readlink / readlinkat` | symlink[01-04], symlinkat01, readlink[01,03], readlinkat[01-02] | ✅ |

### 5.7 Rename / Unlink

| Group               | Tests                                          | Status |
| ------------------- | ---------------------------------------------- | ------ |
| `rename / renameat` | rename[01,03-14], renameat01, renameat2[01-02] | ✅ |
| `unlink / unlinkat` | unlink[05,07-09], unlinkat01                   | ✅ |

### 5.8 Truncate / Sync

| Group                      | Tests                                                                    | Status |
| -------------------------- | ------------------------------------------------------------------------ | ------ |
| `truncate / ftruncate`     | truncate[02-03], ftruncate[01,03-04] + `_64` variants                    | ◐ A-batch (`truncate03_64`) |
| `sync / fsync / fdatasync` | sync01, fsync[01-04], fdatasync[01-03], syncfs01, sync_file_range[01-02] | ✅ A-batch |
| `fallocate`                | fallocate[01-06]                                                         | ✅ A-batch (`05-06` TCONF) |
| `copy_file_range`          | copy_file_range[01-03]                                                   | ✅ 2026-03-09 core100 |

### 5.9 File Nodes

| Group             | Tests                        | Status |
| ----------------- | ---------------------------- | ------ |
| `mknod / mknodat` | mknod[01-09], mknodat[01-02] | ◐ A-batch (`mknod07`, `mknodat02`) |

---

## 6. Advanced / Async I/O (~60)

| Group                       | Tests                                                                                                                                                                                | Status |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------ |
| `pipe / pipe2`              | pipe[01-15], pipe2\_[01-02,04]                                                                                                                                                       | ✅ B-phase1 |
| `sendfile`                  | sendfile[02-09] + `_64` variants                                                                                                                                                     | ✅ B-phase2 (`sendfile02-08` + `_64` pass; `sendfile09/_64` TCONF: needs 5G free space) |
| `splice / tee / vmsplice`   | splice[01-09], tee[01-02], vmsplice[01-04]                                                                                                                                           | ✅ B-phase2 (`splice01-07`, `tee01-02`, `vmsplice01-04`; `splice08-09` TCONF: kernel >= 6.7) |
| `AIO`                       | aio[01-02], aiocp, aiodio_append/sparse, aio-stress, io_setup[01-02], io_submit[01-03], io_getevents[01-02], io_destroy[01-02], io_cancel[01-02], io_control01, io_pgetevents[01-02] | ✅ 2026-03-05 0302 batch (`TCONF` env skips accepted) |
| `io_uring`                  | io_uring[01-02]                                                                                                                                                                      |        |
| `posix_fadvise / readahead` | posix_fadvise[01-04] + `_64`, readahead[01-02]                                                                                                                                       | ✅ (`posix_fadvise01-04` + `readahead01-02`; readahead is `TCONF` in current env) |
| `DIO`                       | dio_append, dio_read, dio_sparse, dio_truncate, diotest[1-6], dma_thread_diotest, doio                                                                                               | ◐ (`dio_append/read/sparse/truncate`, `diotest1-6`, `doio` ✅ on musl+glibc; pending: `dma_thread_diotest`) |

---

## 7. Memory Management (~150)

| Group                        | Tests                                                                                                          | Status |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------- | ------ |
| `mmap / mremap / mprotect`   | mmap[001,01-20], mmap[1-3], mmapstress[01-10], mremap[01-06], mprotect[01-05]                                  | ✅ (`mremap01-06`, `mprotect03-05`; other listed mmap subset already covered) |
| `munmap`                     | munmap[01-03]                                                                                                  |        |
| `brk / sbrk`                 | brk[01-02], sbrk[01-03]                                                                                        |        |
| `mlock / munlock / mlockall` | mlock[01-05], mlock2[01-03], mlockall[01-03], munlock[01-02], munlockall01                                     |        |
| `madvise`                    | madvise[01-03,05-11], process_madvise01                                                                        |        |
| `mincore`                    | mincore[01-04]                                                                                                 | ✅ (`mincore02-04`) |
| `msync`                      | msync[01-04]                                                                                                   | ✅ (`msync01-04`) |
| `userfaultfd`                | userfaultfd01                                                                                                  | ✅ 2026-03-08 |
| `overcommit / page`          | overcommit_memory, page[01-02]                                                                                 | ✅ (`page01-02` earlier; `overcommit_memory` added 2026-03-08) |
| `OOM`                        | oom[01-05]                                                                                                     | ✅ 2026-03-08 (`oom01-05`; `oom02-05` env `TCONF` accepted) |
| `Huge Pages`                 | hugemmap[01-32], hugeshmat[01-05], hugeshmctl[01-03], hugeshmdt01, hugeshmget[01-03,05], hugefallocate[01-02] |        |
| `THP`                        | thp[01-04]                                                                                                     |        |

✅ 2026-03-02 MM targeted follow-up passed on musl (`mmap08`, `mmap14`,
`mlock04-05`, `madvise06`, `madvise08` with core-dump-related `TCONF`,
`mincore04`, `page01-02`, `memfd_create01-04`).

✅ 2026-03-05 0302 MM full subset (48 tests) passed on musl+glibc
(`mmap-corruption01`, `mmap001`, `mmap08`, `mmap1-3`, `mmap10-20`,
`mmapstress01-10`, `mlock04-05`, `mlock201-203`, `mlockall03`,
`madvise03,05-11`, `mincore04`, `page01-02`, `memfd_create01-04`).

✅ 2026-03-05 brk/sbrk regression fix validated on musl+glibc:
`mmapstress03` + `shmt09` both PASS (no conflict between mmap-hole growth
and SysV SHM growth blocking).

✅ 2026-03-08 MM overcommit/OOM batch passed on musl+glibc
(`overcommit_memory`, `oom01-05`; `oom02-05` are accepted env `TCONF` in the
current image because `libnuma` development support is absent).

✅ 2026-03-08 MM mmap/mremap/msync/mincore follow-up passed on musl+glibc
(`brk02`, `mprotect03-05`, `munmap01-03`, `mremap01-06`, `msync01-04`, `mincore02-03`;
`sbrk03` is arch-specific `TCONF` on riscv64 and only meaningful on s390 32-bit).

✅ 2026-03-08 special fs/mm follow-up passed on musl+glibc
(`creat07`, `creat09`, `userfaultfd01`).

✅ 2026-03-10 readiness follow-up revalidated `userfaultfd01` on musl+glibc.

---

## 8. IPC (~90)

| Group              | Tests                                                                                                     |
| ------------------ | --------------------------------------------------------------------------------------------------------- |
| SysV Messages      | msgctl[01-06,12], msgget[01-05], msgrcv[01-03,05-08], msgsnd[01-02,05-06], msgstress01, msg_comm ✅ (`msgctl01-06,12`, `msgget01-05`, `msgrcv01-03,05-08`, `msgsnd01-02,05-06`, `msgstress01`, `msg_comm`) |
| SysV Semaphores    | semctl[01-09] (✅ 01), semget[01-02,05], semop[01-05], sem_comm, sem_nstest, semtest_2ns                  |
| SysV Shared Memory | shmctl[01-08], shmget[02-06], shmat[01-04], shmdt[01-02], shm_comm, shmem_2nstest, shmt[02-10], shmnstest ✅ (`shmat02-04`, `shmctl01-08`, `shmdt01-02`, `shmget02-06`, `shmt02-10`; long `shmat1` excluded) |
| POSIX MQ           | mq*open01, mq_notify[01-03], mq_timedreceive01, mq_timedsend01, mq_unlink01, mqns*[01-04] ✅ (`mq_open01`, `mq_notify01-03`, `mq_timedreceive01`, `mq_timedsend01`, `mq_unlink01`, `mqns_01-04`) |
| Futex              | futex_wait[01-05], futex_wait_bitset01, futex_wake[01-04], futex_cmp_requeue[01-02], futex_waitv[01-03] (✅ `futex_wait05`, `futex_wait_bitset01`, `futex_wake02`, `futex_cmp_requeue01`, `futex_cmp_requeue02`) |
| eventfd            | eventfd[01-06], eventfd2\_[01-03] ✅ 2026-03-11 focused follow-up (`eventfd01-06`, `eventfd2_01-03`)     |
| timerfd            | timerfd[01-02], timerfd_create01, timerfd_gettime01, timerfd_settime[01-02] ✅ 2026-03-11 focused rerun (`timerfd01-02`, `timerfd_create01`, `timerfd_gettime01`, `timerfd_settime01-02`); `timerfd04` 当前因缺少 `CONFIG_TIME_NS` 得到内核配置级 `TCONF` |
| signalfd           | (see Signals section)                                                                                     |

✅ NEXT100 IPC batch passed (`msgctl01/02`, `msgget01/02`, `msgrcv01`, `msgsnd01`,
`semctl01/02`, `semop01/02`, `semget01`, `shmat01`, `eventfd01-06`, `eventfd2_01-03`,
`futex_wait01-05`, `futex_wait_bitset01`, `futex_wake01-04`, `futex_cmp_requeue01-02`,
`futex_waitv01-03`, `timerfd04`).

✅ 2026-03-11 eventfd epoll follow-up passed on riscv64: manual
`eventfd_epoll_smoke` validated `EPOLLOUT -> EPOLLIN -> EPOLLOUT` readiness
transitions under `epoll_wait`, and focused LTP reruns kept `eventfd01-06` and
`eventfd2_01-03` passing on musl+glibc.

✅ 2026-03-11 timerfd follow-up passed on riscv64: focused musl+glibc reruns
confirmed `timerfd01-02`, `timerfd_create01`, `timerfd_gettime01`,
`timerfd_settime01-02` continue to pass after moving timerfd to a real waitable
file object. `timerfd04` is presently `TCONF` because `/proc/config.gz` does
not advertise `CONFIG_TIME_NS=y`, so it is tracked as an environment/config gap
rather than a timerfd syscall regression. `timerfd_settime02` itself is an LTP
race-stress test with `.max_runtime = 150`, so on a healthy kernel it is still
expected to run until the test framework requests exit and then report `TPASS`,
not to terminate like a short functional case.

✅ 2026-03-14 POSIX MQ readiness follow-up passed on riscv64: manual
`mq_epoll_smoke`, `mq_unlink_epoll_smoke`, and `mq_notify_signal_smoke`
validated `mqueue -> epoll` readiness delivery, `unlink`-after-open queue
liveness, and `mq_notify(SIGEV_SIGNAL)` one-shot / `EBUSY` / unregister
semantics in the shell image. The run also exposed and fixed a userland
`sigaction` wrapper bug: syscall 134 now uses `rt_sigaction` ABI with an
explicit sigset size, matching the kernel-side interface used by Linux/riscv64.

✅ 2026-03-14 nested epoll ctl-wakeup follow-up passed on riscv64: manual
`nested_epoll_ctl_wakeup_smoke` validated the cross-layer wake path where a
task blocks in `parent epoll_wait`, another process adds an already-readable
pipe fd into `child epoll` via `EPOLL_CTL_ADD`, and readiness propagates back
through the nested epoll topology without relying on timeout polling.

✅ 2026-03-14 nested epoll ctl-del follow-up passed on riscv64: manual
`nested_epoll_ctl_del_smoke` validated that removing a readable pipe interest
from `child epoll` via `EPOLL_CTL_DEL` also clears stale readiness from the
`parent epoll` view, and that re-adding the same unread source makes the nested
chain immediately ready again.

✅ 2026-03-14 nested epoll one-shot follow-up passed on riscv64: manual
`nested_epoll_oneshot_smoke` validated `EPOLLONESHOT` propagation through the
`pipe -> child epoll -> parent epoll` chain: the first readable transition
reaches the parent, the second write stays suppressed before rearm, and a
subsequent `EPOLL_CTL_MOD` on the child interest restores one more wakeup.

✅ 2026-03-14 nested epoll edge-triggered follow-up passed on riscv64: manual
`nested_epoll_et_smoke` validated `EPOLLET` propagation through the
`pipe -> child epoll -> parent epoll` chain across two distinct ready edges.
The run also exposed and fixed a kernel bookkeeping bug: `epoll_wait` now
refreshes edge-triggered `last_ready` snapshots even on empty returns, so a
ready -> not-ready -> ready transition no longer loses the next edge.

✅ 2026-03-14 nested epoll edge-triggered maxevents follow-up passed on riscv64:
manual `nested_epoll_et_maxevents_smoke` validated that `epoll_wait(maxevents=1)`
still refreshes ET snapshots for later nested interests that were not returned
in the current batch. Without that bookkeeping, a sibling `child epoll` could
stay stuck in a stale ready state and lose its next edge after transitioning
through not-ready.

✅ 2026-03-14 nested epoll parent one-shot follow-up passed on riscv64: manual
`nested_epoll_parent_oneshot_smoke` validated that `EPOLLONESHOT` on the
`parent epoll -> child epoll` link disables only the parent-side interest after
the first wakeup, stays suppressed across a full child `not-ready -> ready`
transition, and is re-enabled by `EPOLL_CTL_MOD` rearm.

✅ 2026-03-01 SysV MSG/SEM subset passed (`msgctl12`, `msgget05`, `msgrcv05-08`,
`msgsnd06`, `semctl09`, `semget02`).

✅ 2026-03-02 POSIX MQ batch passed on musl+glibc (`mq_open01`, `mq_notify01-03`,
`mq_timedreceive01`, `mq_timedsend01`, `mq_unlink01`).

✅ 2026-03-02 SysV SHM batch passed on musl+glibc (`shmat02-04`, `shmctl01-08`,
`shmdt01-02`, `shmget02-06`, `shmt02-10`; `shmat1` intentionally excluded as a
long-running stress case).

✅ 2026-03-02 IPC namespace follow-up passed on musl+glibc (`mqns_01-04`,
`mesgq_nstest`, `msg_comm`, `sem_comm`, `shm_comm`, `shmem_2nstest`,
`shmnstest`, `sem_nstest`, `semtest_2ns`).

✅ 2026-03-09 core100 SysV MSG/SEM follow-up passed (`msgctl03-05`,
`msgsnd02`, `msgsnd05`, `msgrcv02-03`, `semctl03-05`, `semctl07-08`,
`semop03`).

---

## 9. Threading (~15)

| Group                      | Tests                                                   |
| -------------------------- | ------------------------------------------------------- |
| Robust list / TID          | set_robust_list01, get_robust_list01, set_tid_address01 |
| Thread area                | set_thread_area01                                       |
| ptrace (thread)            | ptrace[01-11]                                           |
| NPTL                       | nptl01                                                  |
| pth_str / clisrv            | pth_str[01-03], run_sched_cliserv.sh                    |

✅ 2026-03-05 0302 Threading batch passed on musl+glibc (`set_thread_area01`,
`nptl01`, `ptrace01-11`, `pth_str01-03`, `run_sched_cliserv.sh`).

---

## 10. Time & Clocks (~60)

| Group                              | Tests                                                                          |
| ---------------------------------- | ------------------------------------------------------------------------------ |
| `clock_gettime / settime / getres` | clock_gettime[01-04], clock_settime[01-03], clock_getres01                     |
| `clock_nanosleep / nanosleep`      | clock_nanosleep[01-04], nanosleep[01-02,04]                                    |
| `clock_adjtime / adjtimex`         | clock_adjtime[01-02], adjtimex[01-03]                                          |
| `getitimer / setitimer`            | getitimer[01-02], setitimer[01-02]                                             |
| `gettimeofday / settimeofday`      | gettimeofday[01-02], settimeofday[01-02]                                       |
| `times / time / stime`             | times[01,03], time01, stime[01-02]                                             |
| `utime / utimensat`                | utime[01-07] (✅ 01), utimensat01, utimes01, futimesat01                       |
| POSIX Timers                       | timer_settime[01-03], timer_gettime01, timer_delete[01-02], timer_getoverrun01 ✅ 2026-03-10 (musl+glibc; `CLOCK_BOOTTIME*`/`CLOCK_REALTIME_ALARM`/`CLOCK_TAI` remain env TCONF) |
| `leapsec`                          | leapsec01                                                                      |

✅ NEXT100 time batch passed (`adjtimex01-03`, `clock_adjtime02`, `futimesat01`,
`settimeofday01-02`, `stime01-02`, `time01`, `times03`, `utime02-07`, `utimes01`, `leapsec01`).

✅ 2026-03-09 core100 time batch passed (`clock_gettime01-04`,
`clock_settime01-03`, `clock_getres01`, `clock_nanosleep01-04`,
`nanosleep01-02,04`, `gettimeofday02`).

---

## 11. Networking — Sockets (~500+)

> The network tests form the largest single category.

| Group                            | Tests                                                                                                                                                                                                                                                                                       |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Socket basics ✅                 | socket[01-02], socketpair[01-02], socketcall[01-03], sockioctl01                                                                                                                                                                                                                            |
| Bind / connect / listen / accept ✅ | bind[01-06], connect[01-02], listen01, accept[01-03], accept4_01                                                                                                                                                                                                                            |
| Send / recv family ✅             | send[01-02], sendto[01-03], sendmsg[01-03], sendmmsg[01-02], recv01, recvfrom01, recvmsg[01-03], recvmmsg01                                                                                                                                                                                 |
| Socket options ✅                | setsockopt[01-10], getsockopt[01-02], getsockname01, getpeername01                                                                                                                                                                                                                          |
| select / poll / epoll ✅         | select[01-04], pselect[01-03] + `_64`, poll[01-02], ppoll01, epoll*create[01-02], epoll_create1*[01-02], epoll_ctl[01-05], epoll_wait[01-07], epoll_pwait[01-05], epoll-ltp ✅ 2026-03-10 full readiness follow-up (`select01-04`, `poll01-02`, `ppoll01`, `pselect01-03` + `_64`, `epoll_create01-02`, `epoll_create1_01-02`, `epoll_ctl01-05`, `epoll_wait01-07`, `epoll_pwait01-05`, `epoll-ltp`) |
| TCP (IPv4) stress                | tcp4-uni-basic[01-14], tcp4-uni-dsackoff[01-14], tcp4-uni-pktlossdup[01-14], tcp4-uni-sackoff[01-14], tcp4-uni-smallsend[01-14], tcp4-uni-tso[01-14], tcp4-uni-winscale[01-14], tcp4-multi-diffip[01-14], tcp4-multi-diffnic[01-14], tcp4-multi-diffport[01-14], tcp4-multi-sameport[01-14] |
| TCP (IPv6) stress                | tcp6-\* (same structure as IPv4, ~140 tests)                                                                                                                                                                                                                                                |
| UDP (IPv4/IPv6)                  | udp4-multi-diffip[01-07], udp4-multi-diffnic[01-07], udp4-multi-diffport[01-07], udp4-uni-basic[01-07], udp6-\* (mirror)                                                                                                                                                                    |
| ICMP                             | icmp4-multi-diffip[01-07], icmp4-multi-diffnic[01-07], icmp6-multi-\* (mirror), icmp_rate_limit01                                                                                                                                                                                           |
| SCTP                             | sctp01.sh, sctp*big_chunk, test_1_to_1*_, test_basic_, test*connect\*, test_assoc*_, test_fragments_, test*getname*, test_peeloff*, test_sockopt\*, test_sctp*\*                                                                                                                            |
| Network namespaces               | netns\__, ns-_ clients/servers                                                                                                                                                                                                                                                              |
| Network protocols                | vlan[01-03].sh, gre[01-02].sh, geneve[01-02].sh, vxlan[01-04].sh, macsec[01-03].sh, macvlan01.sh, macvtap01.sh, mpls[01-04].sh, wireguard[01-02].sh, sit01.sh                                                                                                                               |
| IPsec                            | ipsec_lib.sh, tcp_ipsec*, udp_ipsec*, sctp_ipsec*, dccp_ipsec*                                                                                                                                                                                                                              |
| VSOCK                            | vsock01                                                                                                                                                                                                                                                                                     |
| Multicast                        | mcast-\_, mc\_\_                                                                                                                                                                                                                                                                            |

✅ 2026-03-10 manual `nested_epoll_smoke` passed on riscv64 shell image
(`parent epoll -> child epoll -> pipe/HUP` basic chain), so current nested
epoll follow-up focus is broader corner-case coverage rather than a known
basic readiness failure.

✅ 2026-03-11 manual `epoll_ctl_wakeup_smoke` passed on riscv64 shell image
(`epoll_wait` blocks on empty epoll, then another process `EPOLL_CTL_ADD`s an
already-readable pipe fd), covering the epoll self-wakeup path for ctl-driven
readiness changes.

---

## 12. Filesystem Advanced (~200)

| Group               | Tests                                                                                                                                                                                                                          |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| inotify             | inotify[01-12], inotify*init1*[01-02] ✅ 2026-03-08 0305 follow-up (`inotify01-04`, `inotify10-12`, `inotify_init1_01-02` PASS; `inotify05-09` TCONF accepted)                                                                                                                                                                                          |
| fanotify            | fanotify[01-23] ✅ 2026-03-08 0306 full batch (`fanotify01-23` PASS; current env accepts fanotify-not-configured / overlayfs-on-tmpfs TCONF where applicable)                                                                 |
| Extended Attributes | getxattr[01-05], setxattr[01-03], listxattr[01-03], removexattr[01-02], fgetxattr[01-03], fsetxattr[01-02], flistxattr[01-03], fremovexattr[01-02], lgetxattr[01-02], llistxattr[01-03], lremovexattr01, getxattr[01-05], acl1 |
| Mount / umount      | mount[01-07], umount[01-03], umount2\_[01-02], mount_setattr01 ✅ 2026-03-06 phase2                                                                                                                                                |
| New mount API       | fsconfig[01-03], fsmount[01-02], fsopen[01-02], fspick[01-02], open_tree[01-02], move_mount[01-02] ✅ 2026-03-06 phase2-3 (`fsconfig01-03`, `fsopen01-02`, `fsmount01-02`, `fspick01-02`, `open_tree01-02`, `move_mount01-02`) |
| Mount namespaces    | mountns[01-04] ✅ 2026-03-19 mountns batch (`mountns01-04` PASS on musl+glibc after mount propagation/object modeling and `/proc/config.gz` namespace gate update)                                                           |
| Bind mounts         | fs_bind[01-24].sh, fs_bind_rbind[01-39].sh, fs_bind_move[01-22].sh, fs_bind_cloneNS[01-07].sh                                                                                                                                  |
| Filesystem stress   | fsstress, fs*racer*\*.sh, growfiles, fsx-linux                                                                                                                                                                                 |
| Filesystem tests    | fs_di, fs_fill, fs_inod, fs_perms, inode[01-02], ftest[01-08]                                                                                                                                                                  |
| chroot / pivot_root | chroot[01-04], pivot_root01                                                                                                                                                                                                    |
| quotactl            | quotactl[01-09]                                                                                                                                                                                                                |
| name_to_handle_at   | name_to_handle_at[01-02]                                                                                                                                                                                                       |
| open_by_handle_at   | open_by_handle_at[01-02]                                                                                                                                                                                                       |
| pkey                | pkey01                                                                                                                                                                                                                         |
| swapfile            | swapon[01-03], swapoff[01-02]                                                                                                                                                                                                  |

---

## 13. Namespaces (~50)

| Group          | Tests                                                             |
| -------------- | ----------------------------------------------------------------- |
| User NS        | userns[01-08] ✅ 2026-03-08 phase3 (`userns01-08` PASS in harness; current env accepts libcap / CONFIG_USER_NS-related TCONF where applicable) |
| PID NS         | pidns[01-06,10,12-13,16-17,20,30-32] ✅ 2026-03-08 phase4 (`pidns01-06,10,12-13,16-17,20,30-32` PASS in harness; current env accepts CONFIG_PID_NS TCONF where applicable) |
| Mount NS       | mountns[01-04] ✅ 2026-03-19 (`mountns01-04` PASS on musl+glibc) |
| Network NS     | netns\_\* (6 tests)                                               |
| Time NS        | timens01                                                          |
| UTS NS (setns) | setns[01-02], unshare[01-02] ✅ (`setns01-02`, `unshare01-02`; UTS path TCONF) |
| IPC NS         | mqns\_[01-04], shmem_2nstest, sem_nstest, mesgq_nstest, shmnstest ✅ (`mqns_01-04`, `mesgq_nstest`, `shmem_2nstest`, `sem_nstest`, `shmnstest`) |

---

## 14. cgroups (~70)

| Group      | Tests                                                        | Status |
| ---------- | ------------------------------------------------------------ | ------ |
| Core       | cgroup_core[01-03]                                           | ✅ 2026-03-09 (pass on musl+glibc) |
| Legacy v1 Generic | cgroup_fj_function.sh freezer/devices/blkio/net_cls/perf_event/net_prio/hugetlb | ✅ 2026-03-09 (pass on musl+glibc) |
| Freeze     | freeze*\*.sh, stop_freeze*\*.sh, libcgroup_freezer           | ✅ 2026-03-09 (`write_freezing.sh`, `freeze_write_freezing.sh`, `freeze_thaw.sh`, `freeze_self_thaw.sh`, `freeze_sleep_thaw.sh`, `freeze_move_thaw.sh` pass on musl+glibc) |
| CPU        | cpuctl\*_, cpuacct_, cpuset*, cfs_bandwidth01, run_cpuctl\*\*.sh | ◐ 2026-03-09 (`cpuacct.sh 1 1`, `cgroup_fj_function.sh cpuacct`, `cgroup_fj_function.sh cpuset`, `cgroup_fj_function.sh cpu` pass on musl+glibc) |
| Memory     | memcg\_\*, memcontrol[01-04], memctl_test01, min_free_kbytes | ✅ 2026-03-09 (`run_memctl_test.sh 1-2`, `memcontrol01`, `memcontrol03-04` pass on musl+glibc; `memcontrol02` env TCONF/PASS) |
| PIDs       | pids.sh, pids_task[1-2]                                      | ◐ 2026-03-09 (`pids.sh 1-3,5-9` pass on musl+glibc) |
| Regression | cgroup*regression*\*.sh                                      | ✅ 2026-03-09 (`cgroup_regression_test.sh` pass on musl+glibc; env TCONF on `sched_debug`/`lockdep`) |

✅ 2026-03-09 cgroup core subset passed on musl+glibc
(`cgroup_core01-03`).

✅ 2026-03-09 cgroup cpu/cpuset subset passed on musl+glibc
(`cpuacct.sh 1 1`, `cgroup_fj_function.sh cpuacct`,
`cgroup_fj_function.sh cpuset`, `cgroup_fj_function.sh cpu`).

✅ 2026-03-09 cgroup legacy v1 generic-controller subset passed on musl+glibc
(`cgroup_fj_function.sh freezer`, `cgroup_fj_function.sh devices`,
`cgroup_fj_function.sh blkio`, `cgroup_fj_function.sh net_cls`,
`cgroup_fj_function.sh perf_event`, `cgroup_fj_function.sh net_prio`,
`cgroup_fj_function.sh hugetlb`).

✅ 2026-03-09 cgroup regression script passed on musl+glibc
(`cgroup_regression_test.sh`; env TCONF on `sched_debug`/`lockdep`).

✅ 2026-03-09 cgroup pids subset passed on musl+glibc
(`pids.sh 1-3,5-9`).

✅ 2026-03-09 cgroup memcontrol v2 subset passed on musl+glibc
(`memcontrol01`, `memcontrol03-04`; `memcontrol02` env TCONF/PASS).

✅ 2026-03-09 cgroup legacy memcontrol migration tests passed on musl+glibc
(`run_memctl_test.sh 1-2`).

---

## 15. Security (~60)

| Group           | Tests                                                               |
| --------------- | ------------------------------------------------------------------- |
| Linux Keys      | keyctl[01-09], add_key[01-05], request_key[01-05]                   |
| IMA             | ima\_\*.sh, ima_mmap, ima_boot_aggregate                            |
| SMACK           | smack*\*.sh, smack_notroot, smack_set*\*                            |
| Crypto          | af_alg[01-07], crypto_user[01-02], pcrypt_aead01                    |
| BPF             | bpf_map01, bpf_prog[01-07] ✅ 2026-03-08 phase4                     |
| Capabilities    | (see section 2)                                                     |
| CVE regressions | cve-2014-0196, cve-2015-3290, cve-2016-_, cve-2017-_, cve-2022-4378 ◐ 2026-03-08 phase4 (`cve-2014-0196`, `cve-2015-3290`, `cve-2016-7117`, `cve-2017-17052`, `cve-2017-17053` PASS; `cve-2016-7042`, `cve-2016-10044`, `cve-2017-16939` env TCONF; `cve-2022-4378` pending) |

---

## 16. CPU Hotplug & Topology (~20)

| Group    | Tests                                                                                                        |
| -------- | ------------------------------------------------------------------------------------------------------------ |
| Hotplug  | cpuhotplug[01-07].sh, cpuhotplug*do*\*.sh                                                                    |
| Affinity | sched_getaffinity01, sched_setaffinity01, ht_affinity, ht_enabled                                            |
| NUMA     | numa01.sh, mbind[01-04], get_mempolicy[01-02], set_mempolicy[01-05], migrate_pages[01-03], move_pages[01-12] |

---

## 17. Kernel Modules & Devices (~15)

| Group        | Tests                                                         |
| ------------ | ------------------------------------------------------------- |
| Modules      | init_module[01-02], finit_module[01-02], delete_module[01-03] ✅ 2026-03-08 phase4 |
| IOCTL        | ioctl[01-09], ioctl_loop[01-07], ioctl_ns[01-07], ioctl_sg01 ✅ 2026-03-06 phase5 (`ioctl01-09`, `ioctl_loop01-07`, `ioctl_sg01`; `ioctl03/08/09/loop*/sg` 为环境 TCONF) + 2026-03-08 0305 (`ioctl02` with `/dev/tty`) |
| I/O priority | ioprio_get01, ioprio_set[01-03] ✅ 2026-03-08 phase5; ioperm[01-02], iopl[01-02] 为 riscv 架构 TCONF |

---

## 18. Performance / perf_event (~10)

| Group      | Tests                                                             |
| ---------- | ----------------------------------------------------------------- |
| perf_event | perf_event_open[01-03]                                            |
| ftrace     | ftrace_regression[01-02].sh, ftrace_stress, ftrace_stress_test.sh |

---

## 19. Miscellaneous Kernel Features (~40)

| Group                          | Tests                                                             |
| ------------------------------ | ----------------------------------------------------------------- |
| `uname / utsname`              | uname[01-02,04] (✅ 04), utsname[01-04], newuname01; set/get hostname/domainname (✅ sethostname[01-03], setdomainname[01-03], gethostname01, getdomainname01) |
| `sysinfo`                      | sysinfo[01-03] ✅ 2026-03-08 phase5 (`sysinfo01-03` PASS in current run) |
| `getrandom`                    | getrandom[01-05] ✅                                               |
| `sysctl / sysfs`               | sysctl[01-04], sysfs[01-05]                                       |
| `syslog / kmsg`                | syslog[11-12], kmsg01                                             |
| `prctl`                        | prctl[01-10] ✅ 2026-03-05 phase4                                 |
| `arch_prctl`                   | arch_prctl01                                                      |
| `personality`                  | personality[01-02] ✅ 2026-03-08 phase5                           |
| `syscall`                      | syscall01                                                         |
| `sysconf / confstr / pathconf` | confstr01, pathconf[01-02], fpathconf01 ✅ 2026-03-08 phase5 (`pathconf01`/`fpathconf01` dual-libc PASS; `pathconf02` glibc PASS, musl blocked; pending: `sysconf01`, `getpagesize01`) |
| `getcpu`                       | getcpu01 ✅ 2026-03-08 phase4                                      |
| `ptrace`                       | ptrace[01-11]                                                     |
| `proc`                         | proc01 ✅ 2026-03-19 dual-libc focused rerun                      |
| `reboot`                       | reboot[01-02]                                                     |
| `sgetmask / ssetmask`          | sgetmask01, ssetmask01                                            |
| `uaccess`                      | uaccess                                                           |
| `membarrier`                   | membarrier01 ✅ 2026-03-08 phase4                                  |
| `stack / ASLR`                 | stack_clash, stack_space, aslr01                                  |
| `meltdown / dirtyc0w`          | meltdown, dirtyc0w, dirtypipe                                     |
| `crash`                        | crash[01-02]                                                      |

---

✅ 2026-03-19 proc/sysctl follow-up completed on musl+glibc:
`proc01` passed on both libcs; `sysctl01`/`sysctl03`/`sysctl04` remain
expected riscv64 arch `TCONF` because LTP reports `__NR__sysctl` absent on
this arch; `commands/sysctl/sysctl01.sh` and `commands/sysctl/sysctl02.sh`
kept procfs-backed `/proc/sys` coverage stable, with remaining skips tied to
unsupported `sched_time_avg` and missing `CONFIG_KALLSYMS*` / `CONFIG_KASAN`,
not new kernel regressions.

## 20. Math / Floating Point (~40)

| Group     | Tests                                                                                                                                                                                                                                                                               |
| --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| float\_\* | float_bessel, float_exp_log, float_iperb, float_power, float_trigo                                                                                                                                                                                                                  |
| gen\*     | gensin, gencos, gentan, genasin, genacos, genatan, genatan2, gencosh, gensinh, gentanh, genexp, gensqrt, genlog, genlog10, genpow, genpower, genfabs, genceil, genfloor, genfmod, genfrexp, genhypot, genj0, genj1, geny0, geny1, genlgamma, genmodf, genldexp, geniperb, genbessel |

---

## 21. Shell-script / Admin Tests (~120)

Most entries ending in `.sh` are network-configuration scripts, distro
utilities, or LTP harness helpers. They generally require a full Linux
userspace and cannot run bare-metal; they are listed here for completeness.

Examples: `ar01.sh`, `df01.sh`, `du01.sh`, `gzip_tests.sh`, `tar_tests.sh`,
`insmod01.sh`, `ldd01.sh`, `lsmod01.sh`, `nm01.sh`, all NFS tests, all
`iptables*.sh`, all TPM scripts, all TCP/UDP/SCTP `.sh` wrappers, etc.

---

## 22. Stress / Benchmark (~25)

| Tests                                                                                                                                                                                   |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| hackbench, ebizzy, kernbench, stress, mmstress, fsstress, growfiles, starvation, tbio, rwtest, mallocstress, ebizzy, netstress, locktests, pipeio, rwtest, writetest, doio, fs_racer.sh |

---

## Coverage Status

Run `python3 tools/find_unimplement_ltp.py` for an up-to-date breakdown.

| Metric                                                                 | Count |
| ---------------------------------------------------------------------- | ----- |
| Total tests in ltp_all.md                                              | 2822  |
| Covered in `ltp_dependence/mod.rs`                                     | 1450  |
| Locally mostly passing (non-`NEXT*` groups in `ltp_dependence/mod.rs`) | ~918  |
| Missing (raw)                                                          | 1373  |
| Missing (compressed ranges)                                            | 861   |

### Priority Order for OS Kernel Implementation

1. **Sections 1–10** — core syscall correctness (process, file, memory, IPC, time, signals, credentials)
2. **Section 11** — basic socket/epoll/select (ignore TCP/UDP stress variants initially)
3. **Section 12** — inotify, fanotify, extended attributes, mount
4. **Section 13** — namespaces
5. **Sections 14–19** — cgroups, security, advanced kernel features
6. **Sections 20–22** — low priority (math tests, shell scripts, network stress)

### Current Recommended Next Lane

Current priority is not “pure optimization first”. The preferred lane is:

1. `procfs / filesystem / files` semantic refactor
2. LTP-driven follow-up batches around that refactor
3. Scheduler/ext4 hotspot optimization after the semantic boundary is cleaner

Recommended next batches after the 2026-03-19 proc/sysctl + mountns reruns:

1. A small bind-mount follow-up set after `mountns01-04`
2. Scheduler/ext4 hotspot optimization after the proc/files semantic boundary stays stable

### How to mark progress

When a group of tests is confirmed passing, add a ✅ to the table row in the
relevant section header or inline table, e.g.:

```
| `semop` | semop[01-05] | ✅ |
```

Then update the **Metric** table above with the new covered count so the
snapshot stays accurate across sessions.

### Section completion tracker

| Section | Category                       | Status              |
| ------- | ------------------------------ | ------------------- |
| 1       | Process Lifecycle              | ✅ core subset + 2026-03-09 core100 process follow-up (`clone05-07`, `wait403`, `execve05-06`, `execveat03`) + 2026-03-19 files-lifecycle follow-up (`vfork`, `vfork01-02`, `execl01`, `execle01`, `execv01`, `execve01-04`, `execveat01-02`) |
| 2       | Process Identity & Credentials | ✅ core subset + credential set/query extended batches + 2026-03-09 getgroups/setgroups follow-up |
| 3       | Scheduling & Priority          | ✅ core subset + NEXT100 sched_tc + get/setpriority |
| 4       | Signals                        | ✅ core subset + NEXT100 signal + 2026-03-09 core100 signal batch |
| 5       | File System — Basic Syscalls   | ✅ core subset + NEXT100 fs-io + fs-io-v2 + FS100 batch1-4 ✅ (`fcntl01-39` + `_64`, `llseek01-03`, `getdents01-02`) + metadata batch ✅ (`chmod[01,03,05-07]`, `chown[01-05]`, `access[01-04]`, `faccessat[01-02]`) + 2026-03-08 0305 metadata follow-up ✅ (`chown[01-05]_16`, `fchown[01-05]_16`, `lchown[01-03]`, `lchown[01-03]_16`) + 2026-03-08 0305 open follow-up ✅ (`open12`, `open14`, `openat02`, `openat04`, `openat2[01-03]`) + 2026-03-08 special fs follow-up ✅ (`creat07`, `creat09`) + directory batch ✅ (`getcwd[01-04]`, `chdir[01,04]`, `fchdir[01-03]`, `mkdir[02-05,09]`, `mkdirat[01-02]`, `rmdir[01-03]`) + link/symlink/readlink batch ✅ (`link[02,04-05,08]`, `linkat[01-02]`, `symlink[01-04]`, `symlinkat01`, `readlink[01,03]`, `readlinkat[01-02]`) + rename/unlink batch ✅ (`rename01,03-14`, `renameat01`, `renameat2[01-02]`, `unlink05,07-09`, `unlinkat01`) + A-metadata/sync batch ✅ (`fallocate01-06`, `sync01`, `fsync01-04`, `fdatasync01-03`, `syncfs01`, `sync_file_range01-02`, `readdir01`, `readdir21`, `mknodat02`, `statx04-12`) + 2026-03-09 core100 fs basics (`dup01-07`, `dup201-207`, `dup3_01-02`, `umask01`, `chroot01-04`, `copy_file_range01-03`, `fchmod01-06`, `fchmodat01-02`, `stat01-03`, `lstat01-02`, `fstat02-03`, `fstatat01`, `statfs02-03`) |
| 6       | Advanced / Async I/O           | ◐ B-phase2 + 2026-03-05 0302 IO subset ✅ (`aio*`, `io_*`, `readahead01-02`, `dio_append/read/sparse/truncate`, `diotest1-6`, `doio`); pending: `dma_thread_diotest` |
| 7       | Memory Management              | ✅ core subset + 2026-03-05 0302 MM full subset (musl+glibc) + 2026-03-08 OOM/overcommit batch + 2026-03-08 mmap/mremap/msync/mincore follow-up + 2026-03-08 special fs/mm follow-up (`userfaultfd01`) |
| 8       | IPC                            | ✅ core subset + NEXT100 ipc + 2026-03-01 SysV msg/sem subset (`msgctl12`, `msgget05`, `msgrcv05-08`, `msgsnd06`, `semctl09`, `semget02`) + 2026-03-02 POSIX MQ batch (`mq_open01`, `mq_notify01-03`, `mq_timedreceive01`, `mq_timedsend01`, `mq_unlink01`) + 2026-03-02 SysV SHM batch (`shmat02-04`, `shmctl01-08`, `shmdt01-02`, `shmget02-06`, `shmt02-10`) + 2026-03-02 IPC NS follow-up (`mqns_01-04`, `mesgq_nstest`, `msg_comm`, `sem_comm`, `shm_comm`, `shmem_2nstest`, `shmnstest`, `sem_nstest`, `semtest_2ns`) + 2026-03-09 core100 SysV msg/sem follow-up (`msgctl03-05`, `msgsnd02,05`, `msgrcv02-03`, `semctl03-05,07-08`, `semop03`) + 2026-03-09 `msgstress01` |
| 9       | Threading                      | ✅ robust/TID subset + 2026-03-05 0302 threading subset (`set_thread_area01`, `nptl01`, `ptrace01-11`, `pth_str01-03`, `run_sched_cliserv.sh`) |
| 10      | Time & Clocks                  | ✅ core subset + NEXT100 time + 2026-03-09 core100 time batch + 2026-03-10 POSIX timers (`timer_settime01-03`, `timer_gettime01`, `timer_delete01-02`, `timer_getoverrun01`) |
| 11      | Networking — Sockets           | ◐ Batch1+Batch2+Batch3+Batch4 core ✅ + 2026-03-10 full readiness follow-up (`select01-04`, `poll01-02`, `ppoll01`, `pselect01-03` + `_64`, `epoll_create01-02`, `epoll_create1_01-02`, `epoll_ctl01-05`, `epoll_wait01-07`, `epoll_pwait01-05`, `epoll-ltp`) |
| 12      | Filesystem Advanced            | ◐ Batch5 xattr core ✅ (except getxattr04/xfs) + 2026-03-08 0305 xattr follow-up ✅ (`fgetxattr01-03`, `fsetxattr01-02`, `flistxattr01-03`, `fremovexattr01-02`, `llistxattr01-03`, `lremovexattr01`) + 2026-03-08 0305 inotify follow-up ✅ (`inotify01-04`, `inotify10-12`, `inotify_init1_01-02`; `inotify05-09` TCONF accepted) + 2026-03-06 phase2 mount/umount ✅ (`mount01-07`, `umount01-03`, `umount2_01-02`, `mount_setattr01`) + 2026-03-06 phase3 new mount API ✅ (`fsconfig01-03`, `fsopen01-02`, `fsmount01-02`, `fspick01-02`, `open_tree01-02`, `move_mount01-02`) + 2026-03-08 0306 fanotify ✅ (`fanotify01-23`; env TCONF accepted where applicable) |
| 13      | Namespaces                     | ◐ IPC NS subset ✅ (`mqns_01-04`, `mesgq_nstest`, `shmem_2nstest`, `sem_nstest`, `shmnstest`) + 2026-03-05 (`timens01`, `setns01-02`, `unshare01-02`) + 2026-03-08 0306 full batch ✅ (`userns01-08`, `pidns01-06`, `pidns10`, `pidns12-13`, `pidns16-17`, `pidns20`, `pidns30-32`; env TCONF accepted where applicable) |
| 14      | cgroups                        | ◐ 2026-03-09 core ✅ (`cgroup_core01-03`) + legacy v1 generic ✅ (`cgroup_fj_function.sh freezer`, `cgroup_fj_function.sh devices`, `cgroup_fj_function.sh blkio`, `cgroup_fj_function.sh net_cls`, `cgroup_fj_function.sh perf_event`, `cgroup_fj_function.sh net_prio`, `cgroup_fj_function.sh hugetlb`) + freezer ✅ (`write_freezing.sh`, `freeze_write_freezing.sh`, `freeze_thaw.sh`, `freeze_self_thaw.sh`, `freeze_sleep_thaw.sh`, `freeze_move_thaw.sh`) + cpu/cpuset ✅ subset (`cpuacct.sh 1 1`, `cgroup_fj_function.sh cpuacct`, `cgroup_fj_function.sh cpuset`, `cgroup_fj_function.sh cpu`; remaining `run_cpuctl*.sh`/`cpuctl_def_task*` still pending scheduler work) + regression ✅ (`cgroup_regression_test.sh`, env TCONF on `sched_debug`/`lockdep`) + `pids.sh 1-3,5-9` ✅ + memory ✅ (`run_memctl_test.sh 1-2`, `memcontrol01`, `memcontrol03-04`; `memcontrol02` env TCONF/PASS) |
| 15      | Security                       | ◐ 2026-03-08 phase4 BPF ✅ + CVE subset ✅ (`cve-2022-4378` pending) |
| 16      | CPU Hotplug & Topology         |                     |
| 17      | Kernel Modules & Devices       | ◐ 2026-03-06 phase5 IOCTL core ✅ (`ioctl01-09`, `ioctl_loop01-07`, `ioctl_sg01`; 保留环境 TCONF) + 2026-03-08 0305 (`ioctl02` ✅ with `/dev/tty`) + 2026-03-08 phase4 modules ✅ (`init_module01-02`, `finit_module01-02`, `delete_module01-03`) + 2026-03-08 phase5 ioprio ✅ (`ioprio_get01`, `ioprio_set01-03`; `ioperm/iopl` riscv TCONF) |
| 18      | Performance / perf_event       |                     |
| 19      | Miscellaneous Kernel Features  | ◐ uts/getrandom subset ✅ + `prctl01-10` ✅ (2026-03-05 phase4) + 2026-03-08 phase4 (`getcpu01`, `membarrier01`) + 2026-03-08 phase5 (`sysinfo01-03`, `personality01-02`, `confstr01`, `pathconf01-02`, `fpathconf01`; `pathconf02` glibc PASS, musl blocked) + 2026-03-19 proc/sysctl follow-up (`proc01` PASS on musl+glibc; `sysctl01/03/04` riscv64 arch `TCONF`; `commands/sysctl/sysctl01-02.sh` stable with expected env/config `TCONF`) |
| 20      | Math / Floating Point          |                     |
| 21      | Shell-script / Admin Tests     |                     |
| 22      | Stress / Benchmark             |                     |
