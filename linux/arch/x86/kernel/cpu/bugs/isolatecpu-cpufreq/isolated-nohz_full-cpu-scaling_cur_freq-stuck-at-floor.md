# 隔离核心 NOHZ_FULL 下 scaling_cur_freq 卡死在最低频（假性降频）

## 环境信息

- 复现主机：`test-host-01`
- 内核版本：`6.6.0-0001.ctl4.x86_64`（CTyunOS）
- CPU：`GenuineIntel family:model:stepping 6:173:1 (0x6:0xad:1)`，`base_mhz 2100`，`max_mhz 3900`，`TSC 2100MHz`，`intel_pstate` 驱动，HWP 使能（`APERF, TURBO, DTS, PTM, HWP, HWPwindow, HWPepp, HWPpkg, EPB`）
- 关键配置：`isolcpus=` 与 `nohz_full=` 使用**完全相同的 CPU 列表**——这是排查最终定位根因的关键线索
- 触发操作：在隔离核心上 `taskset -c <cpu> stress --cpu 1` 满载压测后读取该核心的 `scaling_cur_freq`
- 工单来源：内部工单，2025年-公有云-京津冀海光百G环境-测试，CTyunOS4-6.6-006，2026年7月20日反馈

```bash
cat /proc/cmdline
```

```
BOOT_IMAGE=/vmlinuz-6.6.0-0001.ctl4.x86_64 root=/dev/mapper/ctyunos-root ro rd.lvm.lv=ctyunos/root cgroup_disable=files apparmor=0 crashkernel=512M selinux=0 iommu=pt intel_iommu=on default_hugepagesz=1024M hugepagesz=1024M hugepages=5676 isolcpus=115-119,235-239,355-359,475-479,111-114,231-234,351-354,471-474 irqaffinity=0,120,240,360,1-110,121-230,241-350,361-470 nohz_full=115-119,235-239,355-359,475-479,111-114,231-234,351-354,471-474
```

## 问题现象

### 现象一：隔离核心压测时 scaling_cur_freq 恒为最低频，与实际吞吐不符

工单原始描述：

> pmd的使用的预留核的cpu运行频率一直在800，与服务器运行频率不一致
>
> 实际问题：普通核心查看 `cat /sys/devices/system/cpu/cpu248/cpufreq/cpuinfo_cur_freq` 和 `/proc/cpuinfo`，CPU的频率动态变化，测试时能达到3200MHZ，但是隔离核心 `cat /sys/devices/system/cpu/cpu248/cpufreq/cpuinfo_cur_freq` 和 `/proc/cpuinfo` 一直是800MHZ，需要这内核暂时不支持，需要换个版本重新做下测试。
> 其余问题：线上环境显示 `cat /sys/devices/system/cpu/cpu248/cpufreq/cpuinfo_cur_freq` 为0，需要分析驱动代码。需要提需求单

测试机复现命令：

```bash
taskset -c 479 stress --cpu 1
cat /sys/devices/system/cpu/cpu479/cpufreq/scaling_cur_freq
```

结果：`scaling_cur_freq` 恒定为 `800000`，与普通核心压测时可跑到 3200MHz 明显不一致；线上部分环境 `cpuinfo_cur_freq` 直接读到 0。

## 根因分析

### 是否是 smart_grid / governor 软件调频策略差异（结果：均未生效，排除）

```bash
cat /proc/cmdline | grep -o smart_grid
cat /sys/devices/system/cpu/cpufreq/smart_grid_governor_enable 2>/dev/null
cat /sys/devices/system/cpu/cpu479/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpu479/cpufreq/scaling_available_governors
```

```
[secure@test-host-01 ~]$ cat /proc/cmdline | grep -o smart_grid
[secure@test-host-01 ~]$ cat /sys/devices/system/cpu/cpufreq/smart_grid_governor_enable 2>/dev/null
[secure@test-host-01 ~]$ cat /sys/devices/system/cpu/cpu479/cpufreq/scaling_governor
performance
[secure@test-host-01 ~]$ cat /sys/devices/system/cpu/cpu479/cpufreq/scaling_available_governors
performance powersave
[secure@test-host-01 ~]$ cat /proc/cmdline | grep -o 'isolcpus=[^ ]*'
isolcpus=115-119,235-239,355-359,475-479,111-114,231-234,351-354,471-474
[secure@test-host-01 ~]$
```

`smart_grid` cmdline 匹配为空，`smart_grid_governor_enable` 这个 sysfs 文件本身都不存在（说明 `CONFIG_QOS_SCHED_SMART_GRID` 特性没有在运行时启用），governor 已经是 `performance`。CTKernel 特有的 QoS Smart Grid（会按调度域拓扑周期性覆盖隔离核心之外的 CPU 的 governor，见 `drivers/cpufreq/cpufreq.c` 的 `smart_grid_work_handler()` 和提交 `c9926cd174ca9 "smart_grid: cpufreq: clear offline and isolated CPU in warm CPUs"`）被排除。

再对比 tuned 层面：

```bash
tuned-adm active
cat /etc/tuned/active_profile
grep -rn "isolated_cores\|no_balance_cores" /etc/tuned/cpu-partitioning-variables.conf 2>/dev/null
```

```
[secure@test-host-01 ~]$ tuned-adm active
Current active profile: throughput-performance
[secure@test-host-01 ~]$ cat /etc/tuned/active_profile
throughput-performance
[secure@test-host-01 ~]$ grep -rn "isolated_cores\|no_balance_cores" /etc/tuned/cpu-partitioning-variables.conf 2>/dev/null
2:# isolated_cores=2,4-7
3:# isolated_cores=2-23
6:isolated_cores=${f:calc_isolated_cores:1}
9:# no_balance_cores=5-10
[secure@test-host-01 ~]$
```

活跃的 tuned profile 是 `throughput-performance`，不是 NFV 常用的 `cpu-partitioning`——`cpu-partitioning-variables.conf` 文件存在但里面的 `isolated_cores` 只是默认模板注释，没有生效，排除 tuned 层面的电源策略差异。

### 是否是硬件层面确实降频了（cpupower / HWP_REQUEST MSR 交叉验证，结果：硬件配置无差异）

```bash
cat /sys/devices/system/cpu/cpu479/cpufreq/scaling_max_freq
cat /sys/devices/system/cpu/cpu479/cpufreq/scaling_min_freq
cat /sys/devices/system/cpu/cpu479/cpufreq/cpuinfo_max_freq
cat /sys/devices/system/cpu/cpu479/cpufreq/cpuinfo_min_freq
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq
cat /sys/devices/system/cpu/cpu479/cpufreq/scaling_driver
cat /sys/devices/system/cpu/cpufreq/boost 2>/dev/null
cpupower -c 479 frequency-info 2>/dev/null
```

```
[secure@test-host-01 ~]$ cat /sys/devices/system/cpu/cpu479/cpufreq/scaling_max_freq
3900000
[secure@test-host-01 ~]$ cat /sys/devices/system/cpu/cpu479/cpufreq/scaling_min_freq
800000
[secure@test-host-01 ~]$ cat /sys/devices/system/cpu/cpu479/cpufreq/cpuinfo_max_freq
3900000
[secure@test-host-01 ~]$ cat /sys/devices/system/cpu/cpu479/cpufreq/cpuinfo_min_freq
800000
[secure@test-host-01 ~]$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq
3900000
[secure@test-host-01 ~]$ cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq
3900000
[secure@test-host-01 ~]$ cat /sys/devices/system/cpu/cpu479/cpufreq/scaling_driver
intel_pstate
[secure@test-host-01 ~]$ cat /sys/devices/system/cpu/cpufreq/boost 2>/dev/null
[secure@test-host-01 ~]$ cpupower -c 479 frequency-info 2>/dev/null
analyzing CPU 479:
  driver: intel_pstate
  CPUs which run at the same hardware frequency: 479
  CPUs which need to have their frequency coordinated by software: 479
  maximum transition latency:  Cannot determine or is not supported.
  hardware limits: 800 MHz - 3.90 GHz
  available cpufreq governors: performance powersave
  current policy: frequency should be within 800 MHz and 3.90 GHz.
                  The governor "performance" may decide which speed to use
                  within this range.
  current CPU frequency: Unable to call hardware
  current CPU frequency: 800 MHz (asserted by call to kernel)
  boost state support:
    Supported: yes
    Active: yes
[secure@test-host-01 ~]$
```

关键发现：真实驱动是 **`intel_pstate`**，不是 AMD/海光的 `acpi-cpufreq`（尽管工单来自"海光百G环境"这个测试环境目录，但这台机器的物理 CPU 是 Intel 的）。`scaling_max_freq` 与 `cpuinfo_max_freq` 一致（3900000），并未被软件层限速；`cpu0` 同样的读数完全一致，说明这不是隔离核心特有的软件限速。`cpupower` 自己也提示 `current CPU frequency: Unable to call hardware`，说明它读到的 800MHz 也只是"kernel asserted"（即 sysfs 缓存值），不是真正的硬件直读。

再用 `HWP_REQUEST` MSR（0x774）对比隔离核心与普通核心：

```bash
cat /sys/devices/system/cpu/cpu479/cpufreq/energy_performance_preference
cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference
cat /sys/devices/system/cpu/cpu479/cpufreq/energy_performance_available_preferences
rdmsr -p 479 0x774
sudo rdmsr -p 479 0x774
sudo rdmsr -p 0 0x774
cpupower -c 0 frequency-info 2>/dev/null
```

```
[secure@test-host-01 ~]$ cat /sys/devices/system/cpu/cpu479/cpufreq/energy_performance_preference
performance
[secure@test-host-01 ~]$ cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference
performance
[secure@test-host-01 ~]$ cat /sys/devices/system/cpu/cpu479/cpufreq/energy_performance_available_preferences
default performance balance_performance balance_power power
[secure@test-host-01 ~]$ rdmsr -p 479 0x774
rdmsr: open: Permission denied
[secure@test-host-01 ~]$ sudo rdmsr -p 479 0x774
2727
[secure@test-host-01 ~]$ sudo rdmsr -p 0 0x774
2727
[secure@test-host-01 ~]$ cpupower -c 0 frequency-info 2>/dev/null
analyzing CPU 0:
  driver: intel_pstate
  CPUs which run at the same hardware frequency: 0
  CPUs which need to have their frequency coordinated by software: 0
  maximum transition latency:  Cannot determine or is not supported.
  hardware limits: 800 MHz - 3.90 GHz
  available cpufreq governors: performance powersave
  current policy: frequency should be within 800 MHz and 3.90 GHz.
                  The governor "performance" may decide which speed to use
                  within this range.
  current CPU frequency: Unable to call hardware
  current CPU frequency: 3.21 GHz (asserted by call to kernel)
  boost state support:
    Supported: yes
    Active: yes
[secure@test-host-01 ~]$
```

`energy_performance_preference` 在 cpu479 和 cpu0 上完全相同（`performance`）。`HWP_REQUEST` MSR 在两个 CPU 上读数也完全相同：`0x2727`。解码：bit[7:0] `Min_Perf=0x27=39`，bit[15:8] `Max_Perf=0x27=39`，bit[31:24] `EPP=0`。结合 `cpuinfo_max_freq=3900000`（约 100MHz/单位），`39 → 3.9GHz`——**隔离核心与普通核心的 HWP 硬件请求完全一致**，都被要求钉死在 3.9GHz 档，软件/硬件配置层面找不到任何差异化设置。

同时也注意到 `cpu0` 用 `cpupower` 读到的"3.21 GHz"同样标注为 `asserted by call to kernel`，即同一套软件缓存机制——这为后续定位"上报缓存"问题埋下了线索。

### 压测是否确实钉在隔离核心上运行，且频率读数在压测全程一直冻结（结果：确认钉住，排除采样时机偶然性）

```bash
taskset -c 479 stress --cpu 1 &
for i in $(seq 1 10); do
  cat /sys/devices/system/cpu/cpu479/cpufreq/scaling_cur_freq
  sleep 0.5
done
ps -eLo pid,psr,comm | grep stress
```

```
[secure@test-host-01 ~]$ taskset -c 479 stress --cpu 1 &
[1] 1050106
[secure@test-host-01 ~]$ stress: info: [1050106] dispatching hogs: 1 cpu, 0 io, 0 vm, 0 hdd

[secure@test-host-01 ~]$ for i in $(seq 1 10); do
  cat /sys/devices/system/cpu/cpu479/cpufreq/scaling_cur_freq
  sleep 0.5
done
800000
800000
800000
800000
800000
800000
800000
800000
800000
800000
[secure@test-host-01 ~]$ ps -eLo pid,psr,comm | grep stress
1050106 479 stress
1050107 479 stress
[secure@test-host-01 ~]$
```

`ps` 的 `psr` 列确认 `stress` 主线程和 worker 线程都实实在在跑在 CPU 479 上；`scaling_cur_freq` 在持续 5 秒的满载采样窗口内始终是 `800000`，排除"读取时机凑巧落在空闲间隙"的可能。

### turbostat + 手工 APERF/MPERF 直读验证：硬件频率其实完全正常

```bash
sudo turbostat --cpu 479 --interval 1 --num_iterations 5
```

turbostat 每次运行只在开头打印一次系统/MSR 信息，然后每秒打印一次采样表。为方便阅读拆成两块，数值均为原始输出，未做任何修改。

**(1) 一次性系统信息**（进程/内核信息 + 平台 MSR 能力）：

```
[secure@test-host-01 ~]$ sudo turbostat --cpu 479 --interval 1 --num_iterations 5
turbostat version 2023.03.17 - Len Brown <lenb@kernel.org>
Kernel command line: BOOT_IMAGE=/vmlinuz-6.6.0-0001.ctl4.x86_64 root=/dev/mapper/ctyunos-root ro rd.lvm.lv=ctyunos/root cgroup_disable=files apparmor=0 crashkernel=512M selinux=0 iommu=pt intel_iommu=on default_hugepagesz=1024M hugepagesz=1024M hugepages=5676 isolcpus=115-119,235-239,355-359,475-479,111-114,231-234,351-354,471-474 irqaffinity=0,120,240,360,1-110,121-230,241-350,361-470 nohz_full=115-119,235-239,355-359,475-479,111-114,231-234,351-354,471-474
CPUID(0): GenuineIntel 0x24 CPUID levels
CPUID(1): family:model:stepping 0x6:ad:1 (6:173:1) microcode 0x1000380
CPUID(0x80000000): max_extended_levels: 0x80000008
CPUID(1): SSE3 - SMX EIST TM2 TSC MSR ACPI-TM HT TM
CPUID(6): APERF, TURBO, DTS, PTM, HWP, No-HWPnotify, HWPwindow, HWPepp, HWPpkg, EPB
cpu11: MSR_IA32_MISC_ENABLE: 0x00810089 (TCC EIST No-MWAIT PREFETCH TURBO)
CPUID(7): SGX No-Hybrid
cpu11: MSR_IA32_FEATURE_CONTROL: 0x00100005 (Locked )
CPUID(0x15): eax_crystal: 2 ebx_tsc: 168 ecx_crystal_hz: 25000000
TSC: 2100 MHz (25000000 Hz * 168 / 2 / 1000000)
CPUID(0x16): base_mhz: 2100 max_mhz: 3900 bus_mhz: 100
Uncore Frequency pkg0 die0: 800 - 2500 MHz (800 - 2500 MHz)
Uncore Frequency pkg1 die0: 800 - 2500 MHz (800 - 2500 MHz)
/dev/cpu_dma_latency: 2000000000 usec (default)
current_driver: acpi_idle
current_governor: menu
current_governor_ro: menu
cpu11: POLL: CPUIDLE CORE POLL IDLE
cpu11: C1: ACPI HLT
cpu11: cpufreq driver: intel_pstate
cpu11: cpufreq governor: performance
cpufreq intel_pstate no_turbo: 0
cpu0: MSR_PM_ENABLE: 0x00000001 (HWP)
cpu0: MSR_HWP_CAPABILITIES: 0x05081527 (high 39 guar 21 eff 8 low 5)
cpu0: MSR_HWP_REQUEST: 0x00002727 (min 39 max 39 des 0 epp 0x0 window 0x0 pkg 0x0)
cpu0: MSR_HWP_REQUEST_PKG: 0x8000ff00 (min 0 max 255 des 0 epp 0x80 window 0x0)
cpu0: MSR_HWP_STATUS: 0x00000004 (No-Guaranteed_Perf_Change, Excursion_Min)
cpu120: MSR_PM_ENABLE: 0x00000001 (HWP)
cpu120: MSR_HWP_CAPABILITIES: 0x05081527 (high 39 guar 21 eff 8 low 5)
cpu120: MSR_HWP_REQUEST: 0x00002727 (min 39 max 39 des 0 epp 0x0 window 0x0 pkg 0x0)
cpu120: MSR_HWP_REQUEST_PKG: 0x8000ff00 (min 0 max 255 des 0 epp 0x80 window 0x0)
cpu120: MSR_HWP_STATUS: 0x00000004 (No-Guaranteed_Perf_Change, Excursion_Min)
cpu0: EPB: 0 (performance)
cpu120: EPB: 0 (performance)
cpu0: Guessing tjMax 100 C, Please use -T to specify
cpu120: Guessing tjMax 100 C, Please use -T to specify
cpu0: MSR_IA32_PACKAGE_THERM_STATUS: 0x88220800 (66 C)
cpu0: MSR_IA32_PACKAGE_THERM_INTERRUPT: 0x00000003 (100 C, 100 C)
cpu120: MSR_IA32_PACKAGE_THERM_STATUS: 0x88210800 (67 C)
cpu120: MSR_IA32_PACKAGE_THERM_INTERRUPT: 0x00000003 (100 C, 100 C)
```

这里的 `MSR_HWP_REQUEST: 0x00002727 (min 39 max 39 ...)` 与上一步 `rdmsr -p 479/0 0x774` 读到的 `0x2727` 完全一致，算是同一事实的第二次独立交叉验证（不同工具，同一结论）。

**(2) 每秒采样表**（5 次，表头相同故只保留一次；上面一行是整机聚合行，下面一行是 cpu479 单核行）：

```
Package Core CPU  Avg_MHz Busy%  Bzy_MHz TSC_MHz IPC   IRQ   POLL C1    POLL% C1%   CoreTmp CoreThr PkgTmp
-      -    -    151     4.62   3262    2141    1.64  25484 54   23410 0.00  97.24 67      0       67
1      167  479  3192    99.76  3200    2100    1.92  277   0    0     0.00  0.00
-      -    -    138     4.31   3203    2102    1.78  54046 57   42043 0.00  95.80 67      0       67
1      167  479  3192    99.76  3200    2100    1.90  522   0    0     0.00  0.00
-      -    -    136     4.24   3199    2100    1.80  29549 34   26935 0.00  95.75 67      0       67
1      167  479  3192    99.76  3200    2100    1.91  360   0    0     0.00  0.00
-      -    -    142     4.44   3202    2101    1.72  29855 53   24666 0.00  95.62 66      0       67
1      167  479  3192    99.76  3200    2100    1.91  364   0    0     0.00  0.00
-      -    -    150     4.70   3197    2098    1.64  43123 110  34245 0.00  95.22 67      0       67
1      167  479  3192    99.76  3200    2100    1.91  370   0    0     0.00  0.00
[secure@test-host-01 ~]$
```

`turbostat` 直读 APERF/MPERF/TSC MSR，绕开 `intel_pstate` 自己的上报缓存，五次采样一致显示：**cpu479 的 `Bzy_MHz=3200`，`Busy%=99.76`**——满载状态下持续运行在 3.2GHz。

再手工验证一次，直接读 APERF（0xE8）/MPERF（0xE7）算增量：

```bash
sudo rdmsr -p 479 0xE8; sudo rdmsr -p 479 0xE7   # APERF, MPERF
sleep 1
sudo rdmsr -p 479 0xE8; sudo rdmsr -p 479 0xE7
```

```
[secure@test-host-01 ~]$ sudo rdmsr -p 479 0xE8; sudo rdmsr -p 479 0xE7   # APERF, MPERF
sleep 1
sudo rdmsr -p 479 0xE8; sudo rdmsr -p 479 0xE7
f2dd94e07175
9f6474eeb72c
f2de5afb3155
9f64f6ece298
[secure@test-host-01 ~]$
```

对应 `(APERF_t0, MPERF_t0) = (0xf2dd94e07175, 0x9f6474eeb72c)`，`(APERF_t1, MPERF_t1) = (0xf2de5afb3155, 0x9f64f6ece298)`。计算：

```
delta_APERF = 0xf2de5afb3155 - 0xf2dd94e07175 = 3323641824 (0xc61abfe0)
delta_MPERF = 0x9f64f6ece298 - 0x9f6474eeb72c = 2180918124 (0x81fe2b6c)
ratio       = delta_APERF / delta_MPERF ≈ 1.5240
freq        ≈ 2100MHz(base) * ratio ≈ 3200.3 MHz
```

与 `turbostat` 的 `Bzy_MHz=3200` 完全吻合，双重独立验证。

**结论：硬件频率完全正常，`scaling_cur_freq` 显示的 800MHz 是一个纯软件上报问题，不是真实降频。**

### 代码路径定位：intel_pstate 无 .get() 回调 + arch_scale_freq_tick 缓存过期

`intel_pstate` 在 HWP 模式下没有实现 `.get()` 回调（`drivers/cpufreq/intel_pstate.c:2776-2788`），因此 `scaling_cur_freq` 的读取路径是：

```
show_scaling_cur_freq()                         drivers/cpufreq/cpufreq.c:747
  -> arch_freq_get_on_cpu(policy->cpu)           drivers/cpufreq/cpufreq.c:752
```

`arch_freq_get_on_cpu()`（`arch/x86/kernel/cpu/aperfmperf.c:414`）读取的是一份按 `jiffies` 打时间戳的 APERF/MPERF 增量缓存（`struct aperfmperf` per-cpu），这份缓存**只由 `arch_scale_freq_tick()` 更新**：

```c
void arch_scale_freq_tick(void)
{
    ...
    rdmsrl(MSR_IA32_APERF, aperf);
    rdmsrl(MSR_IA32_MPERF, mperf);
    ...
    s->last_update = jiffies;
    ...
}
```

而 `arch_scale_freq_tick()` **只在 `scheduler_tick()` 里被调用**（`kernel/sched/core.c:5708`），没有任何独立的 hrtimer/kthread 兜底刷新它。

缓存代码本身写了注释说明这是已知取舍：

```c
/*
 * Discard samples older than the define maximum sample age of 20ms. There
 * is no point in sending IPIs in such a case. If the scheduler tick was
 * not running then the CPU is either idle or isolated.
 */
#define MAX_SAMPLE_AGE	((unsigned long)HZ / 50)
```

超过 20ms 没有更新就认为缓存"过期"，直接 fallback 到 `cpufreq_quick_get()`，本质是返回 `policy->cur` 这个静态旧值（intel_pstate 场景下即 P-state 下限）。

**串联到隔离核心的场景**：`nohz_full` 与 `isolcpus` 用的是同一份 CPU 列表。当隔离核心上只剩一个 CPU-bound 任务（`stress --cpu 1`）时，`sched_can_stop_tick()` / `can_stop_full_tick()`（`kernel/time/tick-sched.c:303-323`）只检查 `tick_dep_mask` 里的位（`POSIX_TIMER`、`PERF_EVENTS`、`SCHED`、`RCU` 等，见 `include/linux/tick.h:109-114`），而 `TICK_DEP_BIT_SCHED` 本身也只由 `nr_running > 1` / RT/DL 计数 / CFS bandwidth 决定（`kernel/sched/core.c:1213-1257`）——**完全没有考虑 cpufreq 采样的需求**。单任务隔离核心满足所有停 tick 的条件，`scheduler_tick()` 从此再也不会在这个 CPU 上触发，`arch_scale_freq_tick()` 随之永久停摆，缓存冻结在隔离生效前的状态（通常是刚上电、接近空闲时的低频快照）。

而 HWP 硬件本身是**完全自主**运行的（`HWP_REQUEST` 的 Min=Max 已经钉死），不依赖软件 tick 驱动，所以实际频率一直正确——这也是为什么 `turbostat`（绕开这层软件缓存直读 MSR）能看到真实的 3.2GHz。

## 修复步骤（已验证：编译通过 + 装机重测通过）

在 `arch_freq_get_on_cpu()` 判定缓存过期、准备 fallback 之前，如果目标 CPU 在线且不处于 idle 状态，先通过 IPI（`smp_call_function_single`）远程触发一次 `arch_scale_freq_tick()`，刷新之后再重新读取。

利用的关键性质：APERF/MPERF 在 idle（C1+）期间都不走字，因此即使缓存已经"过期"了很久，只要这段时间 CPU 一直在忙，delta 比值依然是正确的忙时平均频率——不需要凑一个"实时短窗口"，直接复用旧缓存基线重新采样即可。只在 CPU 非 idle 时才发 IPI，避免无意义地打扰真正空闲的隔离核心。

### 1. 补丁内容

```diff
--- a/arch/x86/kernel/cpu/aperfmperf.c
+++ b/arch/x86/kernel/cpu/aperfmperf.c
@@ -12,6 +12,7 @@
 #include <linux/math64.h>
 #include <linux/percpu.h>
 #include <linux/rcupdate.h>
+#include <linux/sched.h>
 #include <linux/sched/isolation.h>
 #include <linux/sched/topology.h>
 #include <linux/smp.h>
@@ -411,16 +412,41 @@ void arch_scale_freq_tick(void)
  */
 #define MAX_SAMPLE_AGE	((unsigned long)HZ / 50)

+static void aperfmperf_snapshot_cpu_ipi(void *info)
+{
+	arch_scale_freq_tick();
+}
+
+/*
+ * A NOHZ_FULL CPU with a single runnable task (e.g. an isolated CPU running
+ * a pinned PMD/busy-poll workload) stops its periodic tick, so nothing ever
+ * calls arch_scale_freq_tick() for it again and cpu_samples goes stale
+ * forever, not just for one MAX_SAMPLE_AGE window. Force one on-demand
+ * sample via IPI so a deliberate frequency read doesn't report the P-state
+ * floor from before isolation took effect. Skip idle CPUs: their frequency
+ * genuinely doesn't matter and there is no point poking them with an IPI.
+ */
+static bool aperfmperf_refresh_stale_sample(int cpu)
+{
+	if (!cpu_online(cpu) || idle_cpu(cpu))
+		return false;
+
+	smp_call_function_single(cpu, aperfmperf_snapshot_cpu_ipi, NULL, 1);
+	return true;
+}
+
 unsigned int arch_freq_get_on_cpu(int cpu)
 {
 	struct aperfmperf *s = per_cpu_ptr(&cpu_samples, cpu);
 	unsigned int seq, freq;
 	unsigned long last;
+	bool refreshed = false;
 	u64 acnt, mcnt;

 	if (!cpu_feature_enabled(X86_FEATURE_APERFMPERF))
 		goto fallback;

+again:
 	do {
 		seq = raw_read_seqcount_begin(&s->seq);
 		last = s->last_update;
@@ -432,8 +458,14 @@ unsigned int arch_freq_get_on_cpu(int cpu)
 	 * Bail on invalid count and when the last update was too long ago,
 	 * which covers idle and NOHZ full CPUs.
 	 */
-	if (!mcnt || (jiffies - last) > MAX_SAMPLE_AGE)
+	if (!mcnt || (jiffies - last) > MAX_SAMPLE_AGE) {
+		if (!refreshed) {
+			refreshed = true;
+			if (aperfmperf_refresh_stale_sample(cpu))
+				goto again;
+		}
 		goto fallback;
+	}

 	return div64_u64((cpu_khz * acnt), mcnt);

fallback:
```

### 2. 设计取舍

这个修复会给隔离/`nohz_full` 核心引入一次额外的 IPI，但只会发生在**有人主动读取该核心 `scaling_cur_freq`** 的时候（不是任何周期性/后台路径），代价与 `acpi-cpufreq` 的 `.get()` 今天对所有核心（包括隔离核心）无条件做的事情完全一样——所以这不是新增的一类抖动风险，而是让 `intel_pstate`/HWP 路径的行为和已有驱动保持一致。

### 3. 编译验证

```bash
scripts/checkpatch.pl --no-tree   # 无 warning/error
make ARCH=x86_64 arch/x86/kernel/cpu/aperfmperf.o   # 编译通过，无警告
```

### 重要提醒

1. **这个修复只解决 `scaling_cur_freq`/`cpuinfo_cur_freq` 的软件上报问题，不改变任何实际调频行为。** HWP 硬件本身一直是对的，客户业务吞吐从未受影响。
2. **IPI 只在非 idle CPU 上触发**，真正空闲的隔离核心不会被打扰，不会引入新的抖动风险。
3. 打包验证过程中遇到的模块签名报错、RPM spec 语法报错均为**环境/发行版配置问题，与本补丁无关**，见下节，不要误判为本次修复引入的问题。

## 生产环境打包与装机验证记录

修复提交后，在容器化构建环境里打包验证，过程中遇到两个与本次 cpufreq 修复无关的独立环境问题，一并记录以便后续排查参考。

```bash
cd /mnt/sdb/wujing/code/realwujing.github.io/linux/virt/container/docker/ctyunos/ctyunos3
make shell
cd ctkernel-lts-6.6-develop
make binrpm-pkg -j48 LOCALVERSION=-isolatecpu-cpufreq-1d399b68f62cd 2> make_error.log
```

### 一、模块签名报错（独立问题，与本 bug 无关）

`make_error.log`：

```
warning: line 34: It's not recommended to have unversioned Obsoletes: Obsoletes: kernel-headers
+ umask 022
+ cd /home/wujing/code/ctkernel-lts-6.6-develop
+ make ARCH=x86 KERNELRELEASE=6.6.0-isolatecpu-cpufreq-1d399b68f62cd KBUILD_BUILD_VERSION=1.ctl3
WARN: resolve_btfids: unresolved symbol cubictcp_state
WARN: resolve_btfids: unresolved symbol cubictcp_recalc_ssthresh
WARN: resolve_btfids: unresolved symbol cubictcp_init
WARN: resolve_btfids: unresolved symbol cubictcp_cwnd_event
WARN: resolve_btfids: unresolved symbol cubictcp_cong_avoid
WARN: resolve_btfids: unresolved symbol cubictcp_acked
+ RPM_EC=0
++ jobs -p
+ exit 0
...
++ make ARCH=x86 -s image_name
+ cp arch/x86/boot/bzImage /home/wujing/code/ctkernel-lts-6.6-develop/rpmbuild/BUILDROOT/kernel-6.6.0_isolatecpu_cpufreq_1d399b68f62cd-1.ctl3.x86_64/boot/vmlinuz-6.6.0-isolatecpu-cpufreq-1d399b68f62cd
+ make ARCH=x86 INSTALL_MOD_PATH=/home/wujing/code/ctkernel-lts-6.6-develop/rpmbuild/BUILDROOT/kernel-6.6.0_isolatecpu_cpufreq_1d399b68f62cd-1.ctl3.x86_64 modules_install
At main.c:320:
- SSL error:2E0AA06F:CMS routines:cms_sd_asn1_ctrl:ctrl failure: crypto/cms/cms_sd.c:235
sign-file: CMS_add1_signer
make[4]: *** [scripts/Makefile.modinst:121: .../lib/modules/6.6.0-isolatecpu-cpufreq-1d399b68f62cd/kernel/arch/x86/events/amd/power.ko] Error 1
make[4]: *** Deleting file '.../kernel/arch/x86/events/amd/power.ko'
...（对 intel-cstate.ko、intel-uncore.ko、rapl.ko、twofish-x86_64.ko、mce-inject.ko、twofish-x86_64-3way.ko、serpent-sse2-x86_64.ko、twofish-avx-x86_64.ko、serpent-avx-x86_64.ko、serpent-avx2.ko 等模块重复同样的报错）...
make[3]: *** [Makefile:1827: modules_install] Error 2
error: Bad exit status from /var/tmp/rpm-tmp.QX2S33 (%install)
make[2]: *** [scripts/Makefile.package:92: binrpm-pkg] Error 1
make[1]: *** [/home/wujing/code/ctkernel-lts-6.6-develop/Makefile:1544: binrpm-pkg] Error 2
make: *** [Makefile:234: __sub-make] Error 2
```

关键点：`cp arch/x86/boot/bzImage ...` 已经成功，说明内核本身（含本次 `aperfmperf.c` 补丁）编译完全正常；报错发生在**之后**的 `modules_install` 阶段的模块签名步骤，且对每一个 `.ko`（`power.ko`、`intel-cstate.ko`、`twofish-x86_64.ko`、`mce-inject.ko` 等互不相关的模块）都是同样的 `cms_sd_asn1_ctrl:ctrl failure` 报错——这个模式说明问题出在签名基础设施/配置本身，与具体模块内容无关，**与本次 cpufreq 修复无关**。

**根因：**

```bash
grep -n "CONFIG_MODULE_SIG" .config
```

```
CONFIG_MODULE_SIG_FORMAT=y
CONFIG_MODULE_SIG=y
# CONFIG_MODULE_SIG_FORCE is not set
CONFIG_MODULE_SIG_ALL=y
# CONFIG_MODULE_SIG_SHA1 is not set
# CONFIG_MODULE_SIG_SHA224 is not set
# CONFIG_MODULE_SIG_SHA256 is not set
# CONFIG_MODULE_SIG_SHA384 is not set
# CONFIG_MODULE_SIG_SHA512 is not set
CONFIG_MODULE_SIG_SM3=y
CONFIG_MODULE_SIG_HASH="sm3"
CONFIG_MODULE_SIG_KEY="certs/signing_key.pem"
CONFIG_MODULE_SIG_KEY_TYPE_RSA=y
# CONFIG_MODULE_SIG_KEY_TYPE_ECDSA is not set
```

`.config` 要求用 **SM3**（国密哈希，GM/T 0004）+ **RSA** 密钥对模块签名。但 SM3 在国密标准里是配合 **SM2** 密钥使用的（GM/T 0003），标准 PKCS#1/X.509/CMS 体系里并不存在"RSA + SM3"的组合签名算法 OID。`scripts/sign-file` 走 OpenSSL 的 `CMS_add1_signer()`，其内部需要把"RSA 密钥类型 + SM3 摘要 NID"解析成一个复合的签名算法标识写进 `SignerInfo`；如果链接的 libcrypto 里找不到这个组合对应的 OID，就会在 `cms_sd_asn1_ctrl` 这一步失败——正是本次报错的错误信息和位置。

进一步确认这是 Kconfig 层面的缺陷，不是环境问题：

```bash
grep -rn "MODULE_SIG_KEY_TYPE" --include="Kconfig*" .
```

```
certs/Kconfig:25:config MODULE_SIG_KEY_TYPE_RSA
certs/Kconfig:30:config MODULE_SIG_KEY_TYPE_ECDSA
```

`certs/Kconfig` 的签名密钥类型 `choice` 完全是上游原版，只有 `RSA`/`ECDSA` 两个选项，**整棵树里没有任何 SM2 密钥类型选项**。而 `kernel/module/Kconfig:259-273` 里的 `MODULE_SIG_SM3` 是一个完全独立的哈希算法 `choice` 分支：

```c
config MODULE_SIG_SM3
	bool "Sign modules with SM3"
	select CRYPTO_SM3
...
config MODULE_SIG_HASH
	string
	default "sm3" if MODULE_SIG_SM3
```

两个 `choice` 块之间没有任何 `depends on`/约束关联，Kconfig 允许同时选中 `MODULE_SIG_SM3` 和 `MODULE_SIG_KEY_TYPE_RSA`——这个组合在 Kconfig 层面"合法"，但在 OpenSSL/CMS 层面无法签名，必然导致 `make modules_install` 在签名阶段失败。`scripts/sign-file.c`、`Documentation/admin-guide/module-signing.rst` 里也都没有任何针对 SM3 的特殊处理或说明，说明这个 `MODULE_SIG_SM3` 选项很可能是为满足国密合规检查项而添加，但从未配合 SM2 密钥类型做过端到端验证。

**修复（本地打包解阻塞）：**

这是发行版级别的 `.config`/Kconfig 问题，不属于本次 cpufreq 修复的范畴，也不应该由本分支代为决定国密签名策略——按下面的方式仅解除本地验证构建的阻塞，是否要为 CTyunOS 的国密合规单独修 Kconfig（补 SM2 密钥类型，或让 `MODULE_SIG_SM3` 依赖对应密钥类型）应由发行版构建配置的负责人决定：

```bash
cd ~/code/ctkernel-lts-6.6-develop   # 容器内，与宿主机是同一份挂载目录

./scripts/config --disable MODULE_SIG_SM3
./scripts/config --enable  MODULE_SIG_SHA512
make ARCH=x86 olddefconfig

grep MODULE_SIG_HASH .config
# CONFIG_MODULE_SIG_HASH="sha512"

# 删除旧的 RSA+SM3 签名证书，让它按新哈希重新生成
rm -f certs/signing_key.pem certs/signing_key.x509 certs/signing_key.csr certs/signing_key.key certs/signing_key.cfg

make binrpm-pkg -j48 LOCALVERSION=-isolatecpu-cpufreq-1d399b68f62cd 2> make_error.log
```

`MODULE_SIG_HASH` 本身没有独立的 `prompt`，只能通过其所属 `choice` 分支的默认值推导，因此用 `olddefconfig` 重新解析而不是直接改字符串值，才能保证 `.config` 内部一致。

### 二、走 ./build/build.sh 打包路径时，需要改的是 defconfig 而不是 .config

上面的 `scripts/config` + `olddefconfig` 只解决了直接 `make binrpm-pkg`（内核自带打包目标）这条路径。CTyunOS 还有一条独立的打包路径：

```bash
cd .../virt/container/docker/ctyunos/ctyunos3
make shell
cd ctkernel-lts-6.6-develop
./build/build.sh
```

`build/build.sh` 走的是 `build/spec/kernel.spec`（`rpmbuild -ba`），其 `%build` 阶段固定会重新跑一次：

```
build/spec/kernel.spec:431/434
    make ARCH=%{Arch} ctyunos_defconfig
```

也就是说每次都会用 `arch/x86/configs/ctyunos_defconfig` **重新生成一份 `.config`**，之前对 `.config` 的手工改动（不管是 `menuconfig` 还是 `scripts/config`）会被这一步整个覆盖掉。检查发现 defconfig 里是同一个根因：

```bash
grep -n "MODULE_SIG" arch/x86/configs/ctyunos_defconfig
# 171:CONFIG_MODULE_SIG_SM3=y
```

只有这一行，没有配套的密钥类型行，会落到 Kconfig `choice` 的默认值 `MODULE_SIG_KEY_TYPE_RSA`——和前面在 `.config` 里踩到的是同一个 RSA+SM3 组合。对应修复：

```diff
--- a/arch/x86/configs/ctyunos_defconfig
+++ b/arch/x86/configs/ctyunos_defconfig
@@ -168,7 +168,7 @@ CONFIG_MODULE_FORCE_LOAD=y
 CONFIG_MODULE_UNLOAD=y
 CONFIG_MODVERSIONS=y
 CONFIG_MODULE_SRCVERSION_ALL=y
-CONFIG_MODULE_SIG_SM3=y
+CONFIG_MODULE_SIG_SHA512=y
 CONFIG_BLK_DEV_ZONED=y
 CONFIG_BLK_DEV_THROTTLING=y
 CONFIG_BLK_DEV_SUPPORT_LEGACY_GLOBAL_LIMIT=y
```

同样要清掉旧的 RSA+SM3 签名证书残留（`certs/signing_key.*`），再跑 `./build/build.sh`。另外 `git status` 会发现 `build/spec/kernel.spec` 也被改动了一处：

```diff
--- a/build/spec/kernel.spec
+++ b/build/spec/kernel.spec
@@ -53,7 +53,7 @@ rm -f test_ctyunos_sign.ko test_ctyunos_sign.ko.sig
 %global upstream_sublevel   0
 %global devel_release       0
 %global maintenance_release 0
-%global pkg_release         01.2
+%global pkg_release         01.2.isolatecpu-cpufreq-1d399b68f62cd
 
 %global ctyunos_lts       1
 %global ctyunos_major     2506
```

这一处不是手工改的——是此前跑过 `make binrpm-pkg -j48 LOCALVERSION=-isolatecpu-cpufreq-1d399b68f62cd` 之后，构建工具链自动把 `pkg_release` 和当时用的 `LOCALVERSION` 对齐了。

### 三、build.sh 第二个报错：Release 字段不能含 `-`（独立问题，与本 bug 及国密签名均无关）

按上面的方式改完 defconfig、清掉旧证书后再跑 `./build/build.sh`，紧接着又报了一个新错，是纯 RPM spec 语法问题：

```
[wujing@67439b2eeb85 ctkernel-lts-6.6-develop]$ ./build/build.sh
~/code/ctkernel-lts-6.6-develop ~/code/ctkernel-lts-6.6-develop
~/code/ctkernel-lts-6.6-develop
error: line 103: Illegal char '-' (0x2d) in: Release: 0001.2.isolatecpu-cpufreq-1d399b68f62cd
```

`kernel.spec:103` 里 `Release: %{devel_release}%{?maintenance_release}%{?pkg_release}` 会把 `devel_release=0`、`maintenance_release=0`、`pkg_release=01.2.isolatecpu-cpufreq-1d399b68f62cd` 拼接成 `Release:` 字段的值——而 RPM 规范里 `Release:`（以及 `Version:`）字段**不允许出现 `-`**，因为 `-` 是最终包名 `name-version-release` 里的分隔符本身。构建工具链自动拿 `LOCALVERSION`（`-isolatecpu-cpufreq-1d399b68f62cd`）去拼 `pkg_release` 时，把分支名里的连字符也带进去了，直接踩了这条 RPM 规则。

修复：把 `pkg_release` 里的 `-` 换成 RPM 允许的 `.`：

```diff
--- a/build/spec/kernel.spec
+++ b/build/spec/kernel.spec
@@ -53,7 +53,7 @@ rm -f test_ctyunos_sign.ko test_ctyunos_sign.ko.sig
 %global upstream_sublevel   0
 %global devel_release       0
 %global maintenance_release 0
-%global pkg_release         01.2.isolatecpu-cpufreq-1d399b68f62cd
+%global pkg_release         01.2.isolatecpu.cpufreq.1d399b68f62cd
 
 %global ctyunos_lts       1
 %global ctyunos_major     2506
```

（中间短暂试过把 `-` 换成 `_`，RPM 语法本身能过，但为了跟 `upstream_version`/`upstream_sublevel` 等字段统一用 `.` 分隔，最终改成了 `.`。）

同样地，`arch/x86/configs/ctyunos_defconfig` 是 CTyunOS 平台共用的发行版配置，不是这个分支专属的东西；这里的改动只用来解除本地用 `./build/build.sh` 验证 cpufreq 修复时的构建阻塞，国密签名策略本身要不要改、怎么改，仍然应该由发行版构建配置的负责人决定，不建议把这一行改动带进 isolcpu/cpufreq 这个分支的提交里。

### 四、装机重测（最终验证）

用上面修好的 `./build/build.sh` 产出的 RPM，在最初复现问题的同一台机器（`test-host-01`）上装机重启，跑同样的压测流程：

```
[secure@test-host-01 ~]$ mpstat -P 474
Linux 6.6.0-0001.2.isolatecpu_cpufreq_1d399b68f62cd.ctl3.x86_64 (test-host-01) 	07/28/2026	_x86_64_	(480 CPU)

06:51:24 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
06:51:24 PM  474    4.89    0.00    0.52    0.00    0.01    0.00    0.00    0.00    0.00   94.58
[secure@test-host-01 ~]$ for i in $(seq 1 10); do   cat /sys/devices/system/cpu/cpu474/cpufreq/scaling_cur_freq;   sleep 0.5; done
800000
800000
800000
800000
800000
800000
800000
800000
800000
800000
[secure@test-host-01 ~]$ taskset -c 474 stress --cpu 1 &
[1] 32823
[secure@test-host-01 ~]$ stress: info: [32823] dispatching hogs: 1 cpu, 0 io, 0 vm, 0 hdd

[secure@test-host-01 ~]$ for i in $(seq 1 10); do   cat /sys/devices/system/cpu/cpu474/cpufreq/scaling_cur_freq;   sleep 0.5; done
3200000
3200000
3200000
3200000
3200000
3200000
3200000
3200000
3200000
3200000
[secure@test-host-01 ~]$ ps -eLo pid,psr,comm | grep stress
  32823 474 stress
  32824 474 stress
[secure@test-host-01 ~]$ kill 32823  32824
[secure@test-host-01 ~]$ for i in $(seq 1 10); do   cat /sys/devices/system/cpu/cpu474/cpufreq/scaling_cur_freq;   sleep 0.5; done
800000
[1]+  Terminated              taskset -c 474 stress --cpu 1
800000
800000
800000
800000
800000
800000
800000
800000
[secure@test-host-01 ~]$
```

`mpstat -P 474` 里的内核版本字符串 `6.6.0-0001.2.isolatecpu_cpufreq_1d399b68f62cd.ctl3.x86_64` 确认这台机器跑的就是带本次修复的内核（`pkg_release` 是当时还用 `_` 分隔的中间版本，后来才统一改成 `.`，不影响功能）。三段行为完全符合预期，且是双向验证：

- 压测前：空闲，`800000` —— 正常。
- 压测中（`ps` 确认 `stress` 主线程和 worker 线程都钉在 `psr=474` 上）：连续 5 秒 10 次采样全部是 `3200000` —— 修复前这里应该会一直卡在 `800000`。
- `kill` 掉压测进程后：马上回落到 `800000` —— 说明这不是"永远报高值"的简单粗暴修法，而是真实反映了 CPU 当前状态：变忙时命中过期缓存触发 IPI 现场刷新，变回真正空闲时 `idle_cpu()` 判断成立，直接跳过 IPI 用回退值，回退值本身在真空闲时刚好也是准的。

这是在最初报障的物理机上，用修复后的内核完整走了一遍"空闲 → 压测 → 停止"的真实场景验证，不只是编译/单元层面的检查。

## 结论

1. **不是真实的性能问题。** 客户的 PMD/DPDK 业务没有被降频，实测吞吐不受影响。
2. **是一个真实存在的内核报告 bug**：`arch_scale_freq_tick()`（依赖调度 tick）与 `nohz_full`（会为单任务核心彻底停掉 tick）之间存在一个从未被处理过的缝隙，导致隔离核心的 `scaling_cur_freq`/依赖它的监控工具永久性地读到错误的低频值，造成误报。`cpuinfo_cur_freq` 读到 0 是同一根因在 `show_cpuinfo_cur_freq()` → `__cpufreq_get()` 路径上的另一种表现（`intel_pstate` 无 `.get()` 回调）。
3. 修复已在最初报障的物理机上完成装机重测，压测前/中/后三段行为均符合预期，问题解决。

## 后续待办

1. 打包过程中暴露的 `MODULE_SIG_SM3` + `MODULE_SIG_KEY_TYPE_RSA` 组合在 OpenSSL/CMS 层面无法签名的 Kconfig 缺陷，与本次 cpufreq 修复无关，暂未推动修复，需要发行版构建配置的负责人决定是否补齐 SM2 密钥类型选项或让 `MODULE_SIG_SM3` 依赖对应密钥类型。
2. `build/spec/kernel.spec` 里 `pkg_release` 拼接 `LOCALVERSION` 时不做 `-` → `.` 转义的问题，目前只是手工绕过，未推动构建工具链层面的修复。

## 参考

- `arch/x86/kernel/cpu/aperfmperf.c`
- `drivers/cpufreq/cpufreq.c`（`show_scaling_cur_freq`、`show_cpuinfo_cur_freq`）
- `drivers/cpufreq/intel_pstate.c`（HWP 模式下无 `.get()` 回调）
- `kernel/time/tick-sched.c`（`can_stop_full_tick()`）
- `kernel/sched/core.c` / `kernel/sched/fair.c`（`TICK_DEP_BIT_SCHED` 的设置条件）
- `kernel/module/Kconfig`（`MODULE_SIG_SM3` 哈希选择）、`certs/Kconfig`（签名密钥类型 `choice`，无 SM2 选项）
- `arch/x86/configs/ctyunos_defconfig`（`./build/build.sh` 路径实际读取的 defconfig）
- `build/spec/kernel.spec`（`./build/build.sh` 对应的 rpmbuild spec，`%build` 阶段的 `make ctyunos_defconfig`）
