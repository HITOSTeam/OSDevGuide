#  双镜像架构

  QEMU 启动时挂载两块 VirtIO 块设备：

  ┌────────────┬─────────────┬───────────────────────────────┬────────────────────────────────────┐
  │   驱动器   │  MMIO 地址  │             来源              │                内容                │
  ├────────────┼─────────────┼───────────────────────────────┼────────────────────────────────────┤
  │ x0 (bus.0) │ 0x1000_1000 │ ext4-fs-packer/target/fs.ext4 │ 系统库 + 用户应用 + 脚本           │
  ├────────────┼─────────────┼───────────────────────────────┼────────────────────────────────────┤
  │ x1 (bus.1) │ 0x1000_2000 │ sdcard-rv.img                 │ 测试二进制（LTP/busybox/iozone等） │
  └────────────┴─────────────┴───────────────────────────────┴────────────────────────────────────┘

  各自提供什么

  x0 (fs.ext4) — 由 ext4-fs-packer 构建，三部分组成：
  - base: img/disk.img → 提供 /lib/（glibc + musl 运行时）、/bin/（LTP 工具）、/etc/
  - user apps: /user/（你的内核用户态程序）
  - extra: /extra/（part_ltp_glibc.sh、part_ltp_musl.sh 等脚本）

  x1 (sdcard-rv.img) — 预构建的测试镜像：
  /
  ├── musl/
  │   ├── ltp/testcases/bin/    ← 1942 个 musl LTP 二进制
  │   ├── lib/libc.so           ← musl libc（也是 soft-float!）
  │   └── busybox, netperf...
  ├── glibc/
  │   ├── ltp/testcases/bin/    ← glibc LTP 二进制
  │   ├── lib/                  ← glibc 运行时
  │   └── busybox...

  内核如何选择 root

  内核检查哪个盘有 /user 目录：
  - x0 有 /user（ext4-fs-packer 放进去的）→ x0 做 PRIMARY
  - x1 没有 /user → x1 做 SECONDARY
  - 路径查找时：先查 PRIMARY，找不到再 fallback 到 SECONDARY

  关键问题：musl 运行时从哪里加载？

  LTP musl 二进制请求的 interpreter 是 /lib/ld-musl-riscv64.so.1（绝对路径）。

  路径查找顺序：先 x0（PRIMARY）→ 找到 img/disk.img 里的 /lib/ld-musl-riscv64.so.1（soft-float）→ 就用这个了。

  所以虽然 sdcard 的 /musl/lib/libc.so 也是 soft-float，但实际加载的是 disk.img 里的那份。

  这意味着：需要替换的是 img/disk.img 里的那个

  而且 sdcard 里的 /musl/lib/libc.so 也是 soft-float（我之前查过），但它不在 /lib/ 路径下，LTP 二进制不会直接用它（除非有 LD_LIBRARY_PATH
  之类的设置）。

  所以替换 img/disk.img 中的 /lib/ld-musl-riscv64.so.1 就能解决问题。 要继续执行替换吗？你倾向从哪里获取 lp64d 的 musl（Alpine 下载 /
  源码编译）？

