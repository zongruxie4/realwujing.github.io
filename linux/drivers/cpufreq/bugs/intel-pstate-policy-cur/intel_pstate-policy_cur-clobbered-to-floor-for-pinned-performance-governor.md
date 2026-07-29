# intel_pstate: policy->cur 被清回 floor，导致隔离核心 scaling_cur_freq 假性降频

## 背景

这是 [x86/aperfmperf isolcpu-cpufreq 那一轮排查](../../../../arch/x86/kernel/cpu/bugs/isolatecpu-cpufreq/isolated-nohz_full-cpu-scaling_cur_freq-stuck-at-floor.md) 过程中，通过审查 `drivers/cpufreq/intel_pstate.c` 源码发现的一个独立 bug。原始症状、复现命令、客户工单都跟那份文档一致（隔离 + `nohz_full` 核心压测时 `scaling_cur_freq`/`cpuinfo` 的 "cpu MHz" 恒为最低频，`turbostat` 却能证明硬件其实跑在满血涡轮频率）。

## 根因

`drivers/cpufreq/intel_pstate.c` 的 `intel_pstate_set_policy()`：

```c
if (cpu->policy == CPUFREQ_POLICY_PERFORMANCE) {
	int pstate = max(cpu->pstate.min_pstate, cpu->max_perf_ratio);

	/*
	 * NOHZ_FULL CPUs need this as the governor callback may not
	 * be invoked on them.
	 */
	intel_pstate_clear_update_util_hook(policy->cpu);
	intel_pstate_set_pstate(cpu, pstate);
} else {
	intel_pstate_set_update_util_hook(policy->cpu);
}
...
/*
 * policy->cur is never updated with the intel_pstate driver, but it
 * is used as a stale frequency value. So, keep it within limits.
 */
policy->cur = policy->min;
```

当 `cpu->policy == CPUFREQ_POLICY_PERFORMANCE` 时（也就是 `scaling_governor=performance`，这也是这台机器唯一可选的两个 governor 之一，见前一份文档的 `scaling_available_governors: performance powersave`），驱动会把这个核心钉死在一个确定的 `pstate` 上，代码自己甚至专门写了注释说明"这是特意为 NOHZ_FULL CPU 准备的，因为 governor 的回调可能压根不会在它们身上被调用"——但**函数末尾无条件地把 `policy->cur` 清回 `policy->min`**，把刚刚为这个分支专门算出来、专门生效的钉死值直接扔掉了。

这段代码本身也留了一句耐人寻味的注释：`policy->cur is never updated with the intel_pstate driver, but it is used as a stale frequency value. So, keep it within limits.` ——上游作者自己也知道 `policy->cur` 在这个驱动下"从来不是准的"，干脆钉在下限，图省事，但没意识到 `CPUFREQ_POLICY_PERFORMANCE` 这个分支其实是可以精确知道的。

### 原本代码会造成什么问题

任何读 `/sys/devices/system/cpu/cpuN/cpufreq/scaling_cur_freq` 或 `/proc/cpuinfo` 里 "cpu MHz" 字段的人或程序，只要撞上 `arch_freq_get_on_cpu()` 的采样过期回退路径（见下一节），看到的都会是这个硬编码的 `policy->min`，而不是这个核心实际被钉住运行的频率——即便它正在满载运行也一样。

这里要强调一点：即使按下面的修复方式改正，`policy->cur` 报出来的也只是驱动**要求**这个核心钉住运行的目标频率（一个估算上限），不是 `turbostat` 那种直接读 APERF/MPERF 实测到的瞬时精确值——这两者不是一回事，后面"精度的取舍"一节会展开说明。原本代码的问题是连这个"估算上限"都没能正确报出来，直接报成了下限。

### 为什么普通核心几乎看不出这个问题

`arch_freq_get_on_cpu()`（`arch/x86/kernel/cpu/aperfmperf.c`）只有在 APERF/MPERF 采样过期（`MAX_SAMPLE_AGE`，20ms）时才会走到 `cpufreq_quick_get()` → `policy->cur` 这条回退路径。普通、持续打 tick 的核心几乎不会撞上这个过期窗口，所以这个 bug 长期潜伏、没人注意到。而**被隔离且覆盖了 `nohz_full`、只剩一个可运行任务的核心，会永久停止打 tick**，于是永久走这条回退路径，把这个从未被正确赋值过的 `policy->min` 当成"当前频率"报出来——尽管硬件明明就钉在上面算出来的那个高频点上运行。

## 修复

```diff
--- a/drivers/cpufreq/intel_pstate.c
+++ b/drivers/cpufreq/intel_pstate.c
@@ -2908,8 +2908,23 @@ static int intel_pstate_set_policy(struct cpufreq_policy *policy)
 		 */
 		intel_pstate_clear_update_util_hook(policy->cpu);
 		intel_pstate_set_pstate(cpu, pstate);
+
+		/*
+		 * Report the exact pinned frequency instead of the floor:
+		 * the CPU is pinned to pstate here and nothing else changes
+		 * it, unlike the general case below.
+		 */
+		policy->cur = pstate * cpu->pstate.scaling;
 	} else {
 		intel_pstate_set_update_util_hook(policy->cpu);
+
+		/*
+		 * Keep policy->cur within limits here: outside of the pinned
+		 * CPUFREQ_POLICY_PERFORMANCE case above, it is never updated
+		 * by the intel_pstate driver, but it is used as a stale
+		 * frequency value.
+		 */
+		policy->cur = policy->min;
 	}
 
 	if (hwp_active) {
@@ -2922,11 +2937,6 @@ static int intel_pstate_set_policy(struct cpufreq_policy *policy)
 			intel_pstate_clear_update_util_hook(policy->cpu);
 		intel_pstate_hwp_set(policy->cpu);
 	}
-	/*
-	 * policy->cur is never updated with the intel_pstate driver, but it
-	 * is used as a stale frequency value. So, keep it within limits.
-	 */
-	policy->cur = policy->min;
 
 	mutex_unlock(&intel_pstate_limits_lock);
```

只在 `CPUFREQ_POLICY_PERFORMANCE` 分支把 `policy->cur` 设成刚刚计算并应用的真实钉死频率（`pstate * cpu->pstate.scaling`，这是这份文件里到处在用的 pstate→kHz 换算方式），其余情况维持原样落到 `policy->min`——因为那种情况下频率本来就没法在不重新采样的前提下确定。

这个修复完全不需要 IPI 或任何跨核心通信：`policy->cur` 是在 policy 配置阶段（改 governor、上线核心时）本地算好、本地赋值的，不是每次 sysfs 读取时才现场触发。

`Fixes:` 标签指向真正引入这处 clobber 的提交，已核对完整 40 位 SHA1 一致：

```
Fixes: d51847acb018 ("cpufreq: intel_pstate: set stale CPU frequency to minimum")
```

## 精度的取舍：能不能拿到硬件实测的精确值

这个修复报的是**请求的上限**（`pstate * scaling`，比如 3.9GHz），不是 `turbostat` 实测到的**真正稳定跑到的频率**（比如受 RAPL/涡轮功耗预算限制后的 3.2GHz）。

排查过有没有办法拿到精确值而不触碰目标核心：

- `turbostat` 本身也拿不到"免费"的精确值——它读 MSR 走的是 `/dev/cpu/N/msr`，内核实现（`arch/x86/kernel/msr.c` 的 `msr_read()` → `rdmsr_safe_regs_on_cpu()`）内部是 `smp_call_function_single()`，也就是要现场在目标核心上执行一次 rdmsr，只是这次是人手动跑一次诊断工具触发的，不是挂在一个会被监控系统自动周期性轮询的 sysfs 文件上。
- 部分 Xeon 平台的 uncore 有 PCU mailbox，理论上能不碰目标核心就查到"某核心当前 P-state"；HFI 也有一张所有核心都能读的共享表——但前者是芯片世代/uncore 强绑定，不是通用机制，后者存的是"性能/能效等级"分类信息，不是"实际跑到多少赫兹"。都不适合作为一个通用、可移植的小修复。

**结论：在完全不打扰目标核心的前提下，没有通用机制能拿到瞬时精确值。** 这次修复报的"钉死上限"是能做到的最好近似——比修复前"卡在 800MHz 下限、方向完全错误"已经是质的改善（低估 4 倍 → 高估约 20%，而且高估的方向不会被误判为"降频异常"）。

这个补丁目前只做过编译验证，**没有做过装机实测**。

## 上游提交状态

已作为独立 bug 提交：

- **收件人**：Srinivas Pandruvada、Len Brown（`INTEL PSTATE DRIVER` 维护者）、Rafael J. Wysocki、Viresh Kumar（`CPU FREQUENCY SCALING FRAMEWORK` 维护者）、Doug Smythies；抄送 `linux-pm@vger.kernel.org`、`linux-kernel@vger.kernel.org`
- Tags：`Fixes:` + `Co-developed-by`/`Signed-off-by: Qiliang Yuan` + `Signed-off-by: Jing Wu`
- `checkpatch.pl` 干净，针对当前 upstream 树编译验证通过
- lore 链接：<https://lore.kernel.org/r/20260729-bug-intel-pstate-policy-cur-v1-1-51f61e5cbd74@gmail.com>

## 参考

- `drivers/cpufreq/intel_pstate.c`（`intel_pstate_set_policy()`）
- `arch/x86/kernel/cpu/aperfmperf.c`（`arch_freq_get_on_cpu()`，回退路径 `cpufreq_quick_get()` → `policy->cur`）
- `arch/x86/kernel/msr.c` / `arch/x86/lib/msr-smp.c`（`turbostat` 走 `/dev/cpu/N/msr` 时同样依赖 `smp_call_function_single()`）
- 前置文档：[isolated-nohz_full-cpu-scaling_cur_freq-stuck-at-floor.md](../../../../arch/x86/kernel/cpu/bugs/isolatecpu-cpufreq/isolated-nohz_full-cpu-scaling_cur_freq-stuck-at-floor.md)
