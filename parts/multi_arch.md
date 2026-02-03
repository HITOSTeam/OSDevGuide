# 关于 多种架构支持

## 如何处理多种结构

此部分 参考 往年 rocketOs的设计,一个核心的arch文件夹里 使用不同cfg来pub use公开对外接口.
详见 `arch/`

```rs
#[cfg(target_arch = "riscv64")]
mod riscv64;
#[cfg(target_arch = "loongarch64")]
mod loongarch64;

#[cfg(target_arch = "riscv64")]
pub use riscv64::*;
#[cfg(target_arch = "loongarch64")]
pub use loongarch64::*;


```

注意: 部分接口 如ipi 需要妥善解决.
修改的部分 可以见 arch 下 对应 的 mod.rs 中的 pub 接口.
外部Makefile添加 上 arch参数来选择对应的架构

## 对于重要改动的说明

对于全新架构需要 将对应的汇编代码 进行全新的编写
以 looongarch架构为例 见下
基本上 对应 riscv64

```asm
.section .text.entry
.global _start

.equ CSR_CPUID, 0x20
.equ CSR_DMW0, 0x180
.equ CSR_DMW1, 0x181

_start:
    csrrd $a0, CSR_CPUID
    获取执行CPU

    li.d $t0, 4096 * 16
    mul.d $t1, $a0, $t0
    la.global $sp, boot_stack_top
    sub.d $sp, $sp, $t1
    为当前CPU 在指定位置 分配地址

    pcaddi      $t0,    0x0
    srli.d      $t0,    $t0,    0x30
    slli.d      $t0,    $t0,    0x30
    addi.d      $t0,    $t0,    0x11
    csrwr       $t0,    CSR_DMW1
    sub.d       $t0,    $t0,    $t0
    addi.d      $t1,    $t0,    0x11
    csrwr       $t1,    CSR_DMW0
    pcaddi      $t0,    0x0
    slli.d      $t0,    $t0,    0x10
    srli.d      $t0,    $t0,    0x10
    jirl        $t0,    $t0,    0x10
    sub.d       $t2,    $t1,    $t1
    addi.d      $t2,    $t2,    0x11
    csrwr       $t2,    CSR_DMW1
    csrwr       $t1,    CSR_DMW0

    bl rust_main

    .section .bss.stack
    .globl boot_stack
boot_stack:
    .space 4096 * 16 * 4
    .globl boot_stack_top
boot_stack_top:

```

### DMW 机制

上述代码 中的第二部分 涉及 一个特殊的叫做DMW机制
pcaddi $t0,    0x0       # PC相对寻址，$t0 = 当前PC地址
srli.d $t0, $t0, 0x30 # 右移48位，提取地址最高16位
slli.d $t0, $t0, 0x30 # 左移48位，恢复到高位，低位清零
addi.d $t0, $t0, 0x11 # 加0x11配置属性 # bit[0]=1: 有效 bit[4]=1: 可写 等
csrwr $t0, CSR_DMW1 # 写入DMW1 (用于高地址映射)
