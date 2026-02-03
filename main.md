# CongOs 结构说明

最后更新时间 2025 01 21 13:23
目前最新分支 basictest_dev

## 项目开发大致过程

1. 前期: 参考 RCore 完成基本的系统框架.从内存管理 到 多线程.(单核)

2. 中期: 做完 rCore 的 基本要求之后,开始研究比赛要求,发现是多核的,就想着先把多核给做了,以免以后大部分代码都要修改.

**这个阶段较为困难,耗时最长.存在 大量 race condtion**

3. 补充 syscall 以及 各种系统特性

4. 文档撰写: 由于开发时间仓促,文档不够详尽,谅解.

## todoList

1. 添加 long arch 支持,多核,文档 //
   2026.1.31 完成arch licbench
2. 完成ltp
3. bug
4. 代码清理
5. 项目开发流程
6. 调度策略

## 各部分说明

该部分介绍系统的大致构成

- [多核设计](./parts/multi_core.md)
- [文件系统](./parts/filesystem.md)
- [内存空间设计](./parts/memory_space.md)
- [如何支持多架构?](./parts/multi_arch.md)
- [优化](./parts/optimize.md)

一些开发文档,记录一些开发细节

- [Shell]
- [ext4-fs-packer 说明]
- [misc 各种开发记录]
- [为通过 BasicTest 而做的努力](./parts/basictest_record.md)
- [为通过 Busybox 而做的努力](./parts/busybox_record.md)
- [为做 cyclictest_testcode 而作的努力]
