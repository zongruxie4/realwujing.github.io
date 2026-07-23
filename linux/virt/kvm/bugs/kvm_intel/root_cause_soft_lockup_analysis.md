# kvm_intel 宿主机 soft lockup 根因分析

## 工单信息

| 字段 | 内容 |
|------|------|
| 产品线 | 弹性计算 |
| 优先级 | P1 |
| 类型 | bug |
| 方案状态 | 无方案 |
| 故障场景 | 上海 7 宿主机 HA 故障 |
| 资源 ID / IP | `sh07-compute-s6-10e35e36e203`、`sh07-compute-s6-10e35e36e210` |
| 问题描述 | 宿主机故障，带外无明显告警 |
| 待回答问题 | 宿主机 HA 故障原因；agent 为什么会出现内核 hang |

> `e203`/`e210` 现场暂无 dmesg，本文分析的是同批次机型上已捕获到的一次 soft lockup 样本 `dmesg-2606`（主机 `sh07-compute-s6-10e35e36e202`，H3C UniServer R4900 G3，2×Intel Xeon Gold 6248R，96 逻辑核，`4.19.90-2102.2.0.0060.ctl2.x86_64`），用来还原"内核 hang"时内核态到底在做什么，供比对 `e203`/`e210` 现场时参考。

---

## 结论先行

- **触发点**：`CPU 1/KVM`（vCPU1 线程）在 `vmx_vcpu_load()` 里做 VMCLEAR 跨核 IPI（`smp_call_function_single`）时忙等 22 秒，被 watchdog 判定 soft lockup。这不是死锁，也不是 KVM 单独的 bug。
- **同一时刻还有 3 组独立的跨核广播/同步在跑**：RCU expedited GP、KVM 自身的 NX 大页回收 TLB flush、以及一次全机 `stop_machine` 集合点。**这几路已用 `release-0060.y` 源码核实过调用路径**，但把 `stop_machine` 归因于 `livepatch_memory60` 周期性触发的猜测经代码核查后**已被推翻**（该补丁的 `stop_machine` 只在加载/卸载那一刻各跑一次，早在这台机器开机 21 秒时就已完成，距离本次故障已过去约 4.2 年 uptime）——真正触发这次 `stop_machine` 的调用方仍未确认。
- **对全部活跃核心做了 IF（中断使能）位核查**：37 个非 idle 核心里有 14 个当时确实是关中断状态，但逐一核对 RIP 后发现全部落在 `vmx_complete_atomic_exit`（VM-exit 收尾的固有极短关中断窗口）或调度 tick 的 `__update_load_avg_cfs_rq`，这些都是纳秒/微秒级的正常关中断区间，**没有任何一个核心的现场能直接解释 22 秒量级的阻塞**。
- **因此修正后的判断是**：软件层面观测到的这几路跨核同步更像是"22 秒全局冻结期间大家都在排队"的**伴随症状**，而不一定是导致 22 秒这个量级延迟的**根本原因**；结合同一窗口内两块不同厂商网卡（i40e、mlx5）都报了 TX 队列超时——这种跨越网络/RCU/KVM 多个互不相关子系统同时失能的现象，更符合一次**硬件/固件级别的停顿**（如 SMI storm、ECC 巡检、PCIe AER 恢复等，对 OS 完全不可见，dmesg 里自然也不会有任何记录）的特征，这与工单里"带外无明显告警"并不矛盾——SMI 本身就不会被 BMC 当作告警上报。
- **对 HA 的影响**：不管根因是软件 IPI 拥塞还是硬件级停顿，故障窗口内两块网卡先后报 TX 队列超时，才是宿主机对 HA 心跳"失联"的直接原因。
- **agent 内核 hang 的本质**：不是 agent 自身的问题，是这几秒内整机（不只是某一个核）对中断/调度失去响应，agent 心跳包和它依赖的网络栈一样被"冻"住了。
- **产线日志印证（2026-07-23 群内同步）**：最新一次产线日志明确记录了"KVM vCPU 关抢占发 IPI 中断后发生 soft lockup"，与本文档的分析方向一致——说明这不是 e202 这一份样本的偶然巧合，而是同一条故障路径在反复复现。

---

## 当前进展与协同状态（截至 2026-07-23，群内讨论纪要）

- **历史规律**：2024 年问题集中爆发时，日志里还留得住 KVM 相关 call trace；2025、2026 年的 HA 重启则基本没能留下内核日志——`e203`/`e210` 拿不到 dmesg，不是这两台机器独有的偶然情况，而是同一个"日志留不住"问题的延续。
- **核心阻塞：vmcore 生不出来**。HA 触发的是**带外强制重启**，系统来不及把日志 write back 到磁盘，因此既没有 vmcore，dmesg 也保不住。已有同事多次提议改成不走带外强制重启（争取软重启/kdump 留出时间），但产线为了提高机器利用率不接受，需要推动 COC 介入。**这是当前推进根因定位的最大障碍**——没有 vmcore，前面『内核源码核查』4) 里"目标 pCPU 是谁、当时在干什么"这类问题永远没法用 `e203`/`e210` 自己的证据来回答，只能拿 `e202` 这份旧样本做类比参考。
- **责任归属尚未明确**：一种意见认为，虚拟化是 KVM 的使用方，根因排查应继续留在内核侧跟踪；也有意见认为，如果内核侧没有明确承诺过接手排查，理论上应转给虚拟化侧排查。目前没有明确的下一步责任人。
- **候选缓解方向：给 KVM MSR 调试日志限流**。日志里存在大量 KVM MSR debug 日志刷屏（因为受影响的 guest 是 Windows，部分 KVM 寄存器不受支持导致 host 侧高频打印），有同事怀疑这些高频打印本身可能干扰或掩盖了故障现场，提议先出一个热补丁给这些日志做 rate limit，观察资源池是否还会复现。但也有意见认为不能确定这就是根因，暂未有团队接手落地。
- **其他背景**：出问题的机器都是 H3C 和华为（超聚变）的 Intel 服务器；已找过服务器厂商核实，对方判断不是外设问题——但这与本文档"硬件/固件级停顿"假说不完全冲突，外设排查范围通常不覆盖 SMI/固件层面的隐性问题，不能算已经排除。
- **小结**：现象已经指向 KVM 相关 soft lockup 这个方向，但 vmcore 因带外 HA 机制无法生成，导致关键证据始终缺位；责任归属、下一步行动方案（尤其是 rate-limit 热修方案是否落地）都还没有定论。

---

## 根因链路

各节点是否 **关中断**（`local_irq_disable`/硬件 IF=0，期间该核连 IPI 都无法被打断执行）、是否 **关抢占**（`preempt_disable`，期间该核不会主动调度但仍可响应中断），标法不完全一样：CPU1、CPU0/CPU48、CPU10、两个等 `mmu_lock` 的核，是根据对应内核路径的**源码语义**直接判定的；CPU90 这一条是**实测 EFLAGS 解出的 IF 位**，比源码推断更确切；"目标 pCPU"那一条则**没有能确认**——它是谁、当时状态如何，这份 dmesg 里没有直接证据，只能列出两个未证实的假说（见图后表格与说明，详细论证见后文「内核源码核查」与「现场排查建议」两节）。

```
[背景，触发方未确认]                        [—]
某次 stop_machine() 调用（cpu_online_mask 全量），
调用方未定（livepatch 已排除，候选见后文『内核源码核查』3)）
      │
      ▼
[04:47:~22 / uptime≈132573296.8]            CPU1 关抢占，不关中断
CPU1 (vCPU1) 发生迁核，vmx_vcpu_load() 对旧 pCPU 发起
同步 VMCLEAR IPI (smp_call_function_single)，开始忙等
（get_cpu() 关抢占；csd_lock_wait 自旋中断保持开启。判定为 soft 而非
 hard lockup 的直接原因见后文『内核源码核查』2) 补充——CPU1 自己踢
 不动看门狗的"踢狗线程"，不需要借助任何外部假设）
（watchdog 22s 后报警，倒推出的起始时刻）
      │
      ▼
[04:47:26.9 / uptime 132573300.9]           [—]（NAPI/softirq 得不到调度，非本核自己关中断）
NETDEV WATCHDOG: ens9f1 (i40e) 65 号发送队列超时
      │
      ▼
[04:47:39.3 / uptime 132573314.3]           [—]
mlx5_core ens3f1 TX timeout detected（另一厂商网卡也超时）
      │
      ▼
[04:47:44.795 / uptime 132573318.795]       CPU1 关抢占持续中（同上）
watchdog: BUG: soft lockup - CPU#1 stuck for 22s!
→ 触发 Sending NMI from CPU1 to CPUs 0,2-95 (318.839471)
      │
      ▼
[04:47:44.840 / uptime 132573318.840，同一次 NMI 快照内，间隔 <1ms]
      ┌───────────────────────────────────────────────────────────────────┐
      │ CPU90 migration/90: multi_cpu_stop 屏障 (839989)                   │
      │   → 实测 IF=1，仍在 PREPARE 陪等，未进入 DISABLE_IRQ（见下文 4)） │
      │                                                                   │
      │ CPU0  kworker rcu_par_gp: sync_rcu_exp_select_node_cpus (839966)   │
      │ CPU48 kworker rcu_par_gp: 同上，另一 NUMA 节点 (839978)             │
      │   → 均为关抢占（get_cpu()），不关中断（csd_lock_wait 自旋，中断开）  │
      │                                                                   │
      │ CPU10 kvm-nx-lpage-re: kvm_flush_remote_tlbs 广播 (840080)          │
      │   → 关抢占（get_cpu()），不关中断（同上自旋模式）                    │
      │                                                                   │
      │ 两个核：tdp_page_fault 里 spin_lock(&mmu_lock) (840144 / 840217)    │
      │   → 关抢占（spin_lock 隐含），不关中断（用的是 _raw_spin_lock，       │
      │     不是 _irqsave/_irq 变体）                                      │
      └───────────────────────────────────────────────────────────────────┘
                    │
                    ▼
[目标 pCPU：上次跑 vCPU1 的那个物理核，快照中未露面，未确认]
      假说 A：关中断（stop_machine 场景，证据不足，详见后文『内核源码核查』4)）
      假说 B：硬件/固件级停顿（SMI 等，详见文末排查建议）
                    │
                    ▼
CPU1 在 smp_call_function_single 里忙等满 22 秒 → watchdog 判定 soft lockup
                    │
                    └── 同一窗口内 NIC softirq/NAPI 同样得不到调度
                        → i40e(04:47:26) / mlx5(04:47:39) 先后 TX queue timeout
                        → 宿主机对外网络（含 HA 心跳）在这几秒内实质性失联
                        → 上层 HA 系统判定该宿主机故障
```

**关中断 vs 关抢占速查**

| 节点 | 关中断 | 关抢占 | 依据 |
|------|:---:|:---:|------|
| CPU1：`smp_call_function_single` 忙等（22s） | 否 | **是** | `get_cpu()`=`preempt_disable`+读 cpu id；`csd_lock_wait()` 用 `cpu_relax()` 自旋，未 `local_irq_disable`，故仍可被 NMI/普通中断打断 |
| CPU90：`migration/90` 的 `multi_cpu_stop` 屏障 | 否（实测 IF=1） | 否（仍在 PREPARE） | 详见后文『内核源码核查』4) |
| CPU0 / CPU48：`sync_rcu_exp_select_node_cpus` 内 `smp_call_function_single` | 否 | **是** | 同 CPU1，标准 `smp_call_function_single` 语义 |
| CPU10：`kvm-nx-lpage-re` 的 `kvm_flush_remote_tlbs`（`smp_call_function_many/single`） | 否 | **是** | 同上，`smp_call_function_many` 同样 `get_cpu()` 包裹全程 |
| 两个核：`tdp_page_fault` 里等 `mmu_lock` | 否 | **是** | 调用的是 `_raw_spin_lock`（非 `_irq`/`_irqsave` 变体），只关抢占不关中断 |
| 目标 pCPU（CPU1 的 VMCLEAR IPI 接收方，具体哪个核未直接观测到） | 未确认 | 未确认 | 两个未证实假说，详见后文『内核源码核查』4) 与文末排查建议 |

> 这里要分清两件事：**"为什么判定成 soft 而不是 hard lockup"** 和 **"为什么这次等待恰好是 22 秒这么长"** 是两个独立的问题。前者已经 100% 确认——CPU1 自己"关抢占不关中断"，直接、独立地解释了 soft/hard 的判定（机制见后文『内核源码核查』2) 补充，不依赖任何外部假设）；后者（为什么目标 pCPU 迟迟没响应）才是仍未确认的部分——其余几路当时都只是在自旋等别人，没有现场能证明自己"关中断关了很久"，这也是为什么后文『内核源码核查』4) 把这部分判定为待定，而非直接归因于某一路软件机制。

> 时间说明：NMI 快照里各 CPU 的现场是在同一次 `Sending NMI` 广播（04:47:44.839471 起）中被逐核打印出来的，彼此打印时间相差不到 1 毫秒，代表的是**同一时刻**的系统状态，不是先后发生的独立事件；而 CPU1 忙等的起点（04:47:~22）与两次网卡 TX 超时（04:47:26、04:47:39）则是这 22 秒窗口内先后发生、层层印证同一次全局迟滞的独立证据。

这不是"哪一行代码错了"能描述的 bug。`stop_machine`/RCU expedited GP/KVM VMCLEAR IPI/KVM NX 大页回收 TLB flush 这四路合法但相互独立的跨核同步机制，确实在同一时刻叠加到了同一批物理核上，这一点源码和快照都能证实；但经过对 `release-0060.y` 源码的核查（详见后文『内核源码核查』），**它们是否就是导致 22 秒这个量级延迟的根本原因，证据并不充分**（没有任何核心表现出与 22 秒匹配的关中断时长）。更完整的判断是：这几路软件机制描述了故障窗口内系统"很堵"的状态，而"为什么堵了整整 22 秒、堵到两块不同网卡都超时"这个量级问题，还需要结合硬件/固件层面的证据（见文末『排查建议』）才能定论。

---

## CPU1 调用栈解读

```
watchdog: BUG: soft lockup - CPU#1 stuck for 22s! [CPU 1/KVM:1035079]
RIP: smp_call_function_single+0xdf/0x100

Call Trace:
 vmx_vcpu_load+0x2be/0x410 [kvm_intel]      // 把 VMCS 加载到本 pCPU
 kvm_arch_vcpu_load+0x7b/0x2b0 [kvm]
 finish_task_switch+0x184/0x2b0
 __schedule+0x29e/0x900
 schedule+0x28/0x80
 kvm_vcpu_block+0x87/0x2f0 [kvm]            // guest HLT，vCPU 线程被调度出去又调回来
 kvm_arch_vcpu_ioctl_run+0x168/0x5b0 [kvm]
 kvm_vcpu_ioctl+0x32b/0x5f0 [kvm]
 do_vfs_ioctl / ksys_ioctl / __x64_sys_ioctl / do_syscall_64
```

对应 4.19 内核 `arch/x86/kvm/vmx.c` 的逻辑：

```c
static void vmx_vcpu_load(struct kvm_vcpu *vcpu, int cpu)
{
    if (vmx->loaded_vmcs->cpu != cpu)
        loaded_vmcs_clear(vmx->loaded_vmcs);   // vCPU 换到新 pCPU 时才会走到这里
}

static void loaded_vmcs_clear(struct loaded_vmcs *loaded_vmcs)
{
    int cpu = loaded_vmcs->cpu;
    if (cpu != -1)
        smp_call_function_single(cpu, __loaded_vmcs_clear, loaded_vmcs, 1); // 同步等待
}
```

vCPU1 线程被调度到了新的物理核上运行，而它的 VMCS 还"挂"在上一次运行它的那个 pCPU 上；VMX 要求先在**那个旧 pCPU** 上做 `VMCLEAR`，因此发一次同步 IPI 并**忙等**（`get_cpu()` 关抢占，不关中断）。这段忙等为什么最终被判定为 "soft" 而非 "hard" lockup，机制比"中断没关"更具体一层，见后文『内核源码核查』2) 补充。

这段代码本身没问题（KVM 标准逻辑），真正的问题是：**目标 pCPU 为什么迟迟没空响应这个 IPI？**——答案在下一节。

---

## 同一瞬间的全核 NMI 快照

`Sending NMI from CPU 1 to CPUs 0,2-95` 打印了全部 95 个核心当时的状态：56 个核心 `idling`，37 个核心在 `vmx_vcpu_run`（guest 运行中被 NMI 打断的正常快照）。剩下几个核心的现场，指向另外 3 组独立的跨核广播/同步操作**同时在进行**：

**① RCU expedited grace period 的 IPI（两个 worker 同时在跑）**

| CPU | 进程 | 现场 |
|-----|------|------|
| CPU0 | `kworker/0:1`，Workqueue `rcu_par_gp` | `smp_call_function_single` ← `sync_rcu_exp_select_node_cpus` |
| CPU48 | `kworker/48:2`，Workqueue `rcu_par_gp` | 同上，另一 NUMA 节点的 CPU 掩码 |

`synchronize_rcu_expedited()` 在 96 核系统上按 NUMA 节点并行下发强制静止状态检查 IPI（dmesg 行 268-320）。

**② KVM 自身的 NX 大页回收线程在广播 TLB flush IPI**

| CPU | 进程 | 现场 |
|-----|------|------|
| CPU10 | `kvm-nx-lpage-re`（KVM NX 大页回收 kthread） | `smp_call_function_single` ← `smp_call_function_many` ← `kvm_make_vcpus_request_mask` ← `kvm_flush_remote_tlbs` ← `kvm_mmu_commit_zap_page` |

`kvm-nx-lpage-re` 是 L1TF/iTLB-multihit 缓解措施引入的后台线程，周期性把拆分过的大页重新 zap，每次都要对**该 VM 所有 vCPU 所在 pCPU** 发 `smp_call_function_many`（dmesg 行 13674-13699）。

**③ 一次全机 `stop_machine` 集合点正在进行**

| CPU | 进程 | 现场 |
|-----|------|------|
| CPU90 | `migration/90`（PID 486，stopper 线程） | `multi_cpu_stop` ← `cpu_stopper_thread` ← `smpboot_thread_fn` |

`multi_cpu_stop` 是 `stop_machine()` 的核心函数：所有目标 CPU 先各自 `local_irq_disable()`，在屏障上互相等待，等最慢的核到齐才统一执行、放行、重新开中断。**屏障等待期间中断是关闭的**，无法处理任何挂起的 IPI（包括 CPU1 发出的 VMCLEAR IPI）。

> **这里最初怀疑是 `livepatch_memory60` 周期性触发的 `stop_machine()`，但对照 `release-0060.y` 源码后已排除**，见下一节「内核源码核查」。

**④ 另外两个核在抢 KVM MMU 的 `mmu_lock`**

`_raw_spin_lock` ← `tdp_page_fault`（dmesg 行 13821、13941）：同一 VM 的多个 vCPU 并发缺页争抢 per-VM `mmu_lock`，进一步拉长 MMU 相关操作（含①②）的完成时间。

---

## 内核源码核查（release-0060.y）

代码已切到 `release-0060.y`（与 dmesg 里的 `4.19.90-2102.2.0.0060.ctl2` 对应），逐一核实上面几个假设。

### 1) `vmx_vcpu_load` / `loaded_vmcs_clear`：与推测一致

`arch/x86/kvm/vmx.c:2222-2229`：

```c
static void loaded_vmcs_clear(struct loaded_vmcs *loaded_vmcs)
{
	int cpu = loaded_vmcs->cpu;

	if (cpu != -1)
		smp_call_function_single(cpu,
			 __loaded_vmcs_clear, loaded_vmcs, 1);
}
```

调用点在 `vmx_vcpu_load`（约 3062-3075 行）：`vmx->loaded_vmcs->cpu != cpu` 时才触发——即 vCPU 换到新 pCPU 时。这一段和 dmesg 里的调用栈完全对得上，没有问题。

### 2) `multi_cpu_stop`：确认"全员必须到齐才能翻页"+"翻页前中断已关"

`kernel/stop_machine.c:187-245`（节选关键部分）：

```c
static int multi_cpu_stop(void *data)
{
	...
	do {
		cpu_relax_yield();
		if (msdata->state != curstate) {
			curstate = msdata->state;
			switch (curstate) {
			case MULTI_STOP_DISABLE_IRQ:
				local_irq_disable();
				hard_irq_disable();
				break;
			case MULTI_STOP_RUN:
				if (is_active)
					err = msdata->fn(msdata->data);
				break;
			}
			ack_state(msdata);          // 本核对当前状态打卡
		} else if (curstate > MULTI_STOP_PREPARE) {
			touch_nmi_watchdog();       // 状态没变就一直自旋等别人
		}
	} while (curstate != MULTI_STOP_EXIT);
	local_irq_restore(flags);           // 直到 EXIT 才恢复中断
	...
}
```

`ack_state()`（170-184 行）用 `atomic_dec_and_test(&msdata->thread_ack)` 判断：**必须所有参与的 CPU 都对当前状态打卡完，才会推进到下一个状态**。也就是说只要有 1 个核迟迟没跑到自己的 `migration/N` 线程去打卡，其余所有参与的核都会卡在同一个状态原地打转——而且一旦进入 `MULTI_STOP_DISABLE_IRQ` 之后的阶段，这些"陪等"的核也是关中断的。

`migration/N`（stopper 线程）的调度类：`kernel/sched/core.c:1587-1603` 确认它被设成 `stop_sched_class`——**内核里优先级最高的调度类**，正常情况下一旦可运行就会立刻抢占任何其它任务。

**但是**：`smp_call_function_single()`（`kernel/smp.c`）在等待跨核 IPI 完成期间用的是 `get_cpu()`（即 `preempt_disable()`）包住整个忙等（`csd_lock_wait()` 用 `cpu_relax()` 自旋，不调用 `schedule()`/`cond_resched()`）。**`preempt_disable()` 期间，就算 `migration/N` 是最高优先级也无法真正被换上 CPU**——调度器只能标记 `NEED_RESCHED`，真正切换要等 `preempt_enable()`（即 `put_cpu()`）之后才会发生。

**这是一个真实存在、源码可验证的结构性风险**：如果 `stop_machine()` 的目标掩码里包含 CPU1，而 CPU1 恰好在这个时间点正卡在 `smp_call_function_single()` 的非抢占忙等里，那么 CPU1 自己的 `migration/1` 线程就进不去、打不了卡，导致整个 `stop_machine()` 集合点**卡死在 `MULTI_STOP_PREPARE`，直到 CPU1 的 IPI 等待自行结束为止**。这与我们在 CPU90 的 EFLAGS 上验证到的 IF=1（还没进入 `DISABLE_IRQ` 阶段，仍在陪等）是吻合的。

#### 补充：为什么这次 100% 确定是 soft 而非 hard lockup —— 看门狗自己的"踢狗"机制，走的是同一条路

之前的说法（"CPU1 中断没关，所以能被 NMI 打断，因此是 soft 而非 hard lockup"）只说对了一半——soft/hard 的判定其实是两个独立的计数器分别决定的，`kernel/watchdog.c` 里能查到确切机制：

```c
static enum hrtimer_restart watchdog_timer_fn(struct hrtimer *hrtimer)
{
	...
	/* kick the hardlockup detector */
	watchdog_interrupt_count();               // 每次 hrtimer 触发就自增，只要 IF=1 就会走到这一步

	/* kick the softlockup detector */
	if (completion_done(this_cpu_ptr(&softlockup_completion))) {
		reinit_completion(this_cpu_ptr(&softlockup_completion));
		stop_one_cpu_nowait(smp_processor_id(),
				softlockup_fn, NULL,
				this_cpu_ptr(&softlockup_stop_work));   // ← 排到本核 stopper 线程队列里
	}
	...
	duration = is_softlockup(touch_ts);       // touch_ts 迟迟没被刷新就报警
```

`softlockup_fn()`（同文件）只干一件事：`__touch_watchdog()` 刷新 `watchdog_touch_ts`。关键是 `stop_one_cpu_nowait()` 把这个"踢狗"工作**排进了 CPU1 自己那个 `stop_sched_class`（最高优先级）的 stopper 线程**——和上面 `migration/N` 是**同一套排队机制**。于是：

- `is_hardlockup()`：检查 `hrtimer_interrupts` 计数器有没有在涨——CPU1 中断没关，hrtimer 一直在正常触发，这个计数器在稳定增长，所以 **hard lockup 检测器认为 CPU1 一切正常**。
- `is_softlockup()`：检查 `watchdog_touch_ts` 有没有被刷新——CPU1 的"踢狗"工作虽然排上了队，但和 `migration/1` 一样，**因为 CPU1 自己 `preempt_disable()` 着，这个最高优先级的排队工作也换不上 CPU**，`touch_ts` 迟迟不刷新，超过阈值后 `is_softlockup()` 返回真，报警打印。

**这就完整、独立地解释了为什么是 soft 而不是 hard lockup——不需要借助任何关于目标 pCPU 或 stop_machine 的假设，纯粹是 CPU1 自己 `preempt_disable()` 期间连自己的踢狗线程都调度不了。** 但这只回答了"为什么报的是 soft"，不回答"为什么这次等待恰好长达 22 秒"——后者仍然是本节前半部分和 4) 里悬而未决的问题。

### 3) 排除 livepatch 作为本次 `stop_machine()` 的触发源

搜索整个源码树里 `stop_machine(`/`stop_machine_cpuslocked(` 的调用点，`kernel/livepatch/core.c` 里确实有两处：

```c
// __klp_disable_patch()
ret = stop_machine(klp_try_disable_patch, &patch_data, cpu_online_mask);
// __klp_enable_patch()
ret = stop_machine(klp_try_enable_patch, &patch_data, cpu_online_mask);
```

但这两处只在**补丁"启用/卸载"这一个动作时各跑一次**，不是周期性的。真正周期性重试的是 `kernel/livepatch/transition.c` 里的 `klp_try_complete_transition()`——但它用的是 `read_lock(&tasklist_lock)` 逐个任务查栈 + `schedule_delayed_work(..., HZ)`（每秒重试一次），**全程没有调用 `stop_machine()`**。

而且从 `Kconfig`（`kernel/livepatch/Kconfig:37-58`）看，`stop_machine` 版本的一致性模型只在 `LIVEPATCH_WO_FTRACE`（不依赖 ftrace 的热补丁方式）下才会被选中，`LIVEPATCH_FTRACE` 走的是纯 per-task 一致性模型（同样不含 `stop_machine`）。

**关键的是时间线**：dmesg 里 `livepatch: enabling patch 'livepatch_memory60'` 发生在这台机器**开机后 21 秒**（`21.235182`），而本次故障发生在 uptime **132573318 秒**（约 **1534 天 / 4.2 年**之后）。补丁早在四年多前就已经"启用"完毕，`stop_machine()` 那一次性调用早就跑完了，不可能是本次快照里 `migration/90` 的来源。

**结论：之前文档里"疑似 livepatch 周期性触发 stop_machine"的推测是错的，予以撤回。** 这次 `stop_machine()` 的真正调用方还未确认，源码里其它 x86 通用的候选调用点包括：`ftrace_run_stop_machine`（`kernel/trace/ftrace.c:2633`，ftrace 动态开关探针时）、`mtrr_rendezvous_handler`（`arch/x86/kernel/cpu/mtrr/mtrr.c`，驱动增删 MTRR 区间时）、`change_clocksource`（`kernel/time/timekeeping.c:1404`）、`__reload_late`（`arch/x86/kernel/cpu/microcode/core.c:607`，微码热加载）——这几个都需要额外的现场证据（比如同一时间是否有人在跑 `perf`/`bpftrace`/`ftrace`，是否有微码热更新任务）才能坐实，仅凭这份 dmesg 无法确定。CPU 热插拔的 `take_cpu_down`（`kernel/cpu.c:874`）已可排除——dmesg 显示这台机器是 `0 hotplug CPUs`，且日志里也没有任何 CPU 上下线的记录。

### 4) 对全部活跃核心做 IF 位核查，并追查目标 pCPU 身份：均无直接证据

把这次 NMI 快照里全部 37 个非 idle 核心的 `EFLAGS` 逐一取出、解出 IF 位（bit 9），关中断（IF=0）的有 14 个，但把它们的 RIP 都比对了一遍：

| RIP | 出现次数 | 说明 |
|-----|---------|------|
| `vmx_complete_atomic_exit+0x6d/0xb0 [kvm_intel]` | 11 | VM-exit 收尾时"是否要重新注入 NMI/外部中断"的固有关中断窗口，纳秒级，每次 VM-exit 都会经过 |
| `__update_load_avg_cfs_rq` | 2 | 调度 tick 里更新 runqueue 负载，rq lock 相关的微秒级关中断区间 |
| `kvm_put_guest_xcr0` | 1 | `vcpu_put` 路径里的几条指令，同样是极短区间 |

这三类都是内核里众所周知的、**预期内**的短暂关中断片段（正常运行的 96 核系统上，随便一次 NMI 快照大概率都能捞到几个），**没有任何一个核心表现出能持续 22 秒量级的关中断行为**。也就是说，"某个核关中断太久，挡住了 CPU1 的 VMCLEAR IPI" 这个假说，**在这份 dmesg 里找不到直接证据支撑**。

**追问一步：能不能反过来，直接从现场里找出目标 pCPU 是谁？** 做了两个尝试，结果都是否定的：

- **从 CPU1 自己的寄存器反查**：CPU1 那条 `RIP: smp_call_function_single+0xdf/0x100` 附带的 `Code:` 字节，实际反汇编出来正是 `csd_lock_wait()` 的尾循环——`mov edx,[rbp-0x38]; and edx,1; je ...; pause; ...`（反复读 `csd->flags`，判断 `CSD_FLAG_LOCK` 位有没有清掉）。这段代码**不再引用 `cpu` 这个参数**——`cpu` 只在函数前半段用来选定发往哪个 per-CPU 队列，用完即被编译器判定为"死值"，不会继续占着寄存器。也就是说，哪怕逐字节反汇编，CPU1 自己的寄存器里也已经找不到目标 CPU 号了：这不是"日志没打全"，而是这个信息在执行到这一步时已经不存在。
- **从另外 95 个核里找"正在执行 IPI 处理函数"的现场**：搜了整份快照里的 `call_function_single_interrupt`/`__loaded_vmcs_clear`，唯一命中的一处在 CPU1 自己的调用栈里、且带 `?` 前缀（`? call_function_single_interrupt+0xa/0x20`）——`?` 意味着栈回溯器自己都不确定这是不是真实调用帧，更像是 CPU1 栈上的旧数据残留。**没有任何一个核心的现场显示自己正在执行这个中断处理函数**。

所以"目标 pCPU 是谁、当时在干什么"不是分析没做到位，而是这份 dmesg 本身没有留下能回答这个问题的信息：1 毫秒宽的快照窗口没有正好拍到目标核处理 IPI 的那个瞬间，调用方的寄存器里也没保留这个值。

综合以上四点，把"22 秒"这个量级的阻塞完全归因于纯软件层面的 IPI/`stop_machine` 拥塞，证据并不充分；两块不同厂商网卡在同一窗口报 TX 超时，更像是在提示一次**跨子系统、跨厂商设备同时失能的全局性冻结**，值得同时排查硬件/固件层面（详见下节建议）。

---

## `e203` / `e210` 现场排查建议

**优先级从高到低：**

0. **前提，且是当前最大障碍：先解决"证据留不下来"的问题**。HA 走带外强制重启，日志来不及 write back，`e203`/`e210` 复现时大概率既没有 vmcore 也没有 dmesg——下面第 1-5 条建议全都要先有日志才能做。需要推动把 HA 的重启方式改为能留出日志/kdump 时间的方案（涉及产线机器利用率的取舍，需要更高层面协调推动）。在此之前，每一次新的复现都会重蹈"只能事后靠 `e202` 这份旧样本类比"的覆辙。
1. **硬件/固件层面**：
   - 查 BMC SEL / IPMI 事件日志（`ipmitool sel elist`）在故障窗口前后是否有 ECC、PCIe AER、CATERR 等记录。
   - 查是否有 `mcelog`/`rasdaemon` 记录（即便 dmesg 里没有 MCE，专门的 RAS 日志也可能单独留存）。
   - 若条件允许，用 `turbostat` 观察 SMI 计数（`SMI` 列）是否在故障时间点前后异常增长——SMI 对 OS 完全不可见，dmesg 天然不会有记录，这也是"带外无明显告警"仍然可能是硬件级问题的原因（很多 BMC 不会把普通 SMI 当作告警上报）。已找服务器厂商核实过，对方判断不是外设问题，但外设排查通常不覆盖 SMI/固件层面的隐性问题，不能算已经排除。
2. **网卡/调度证据**：确认这两台机器的 `dmesg`/`sosreport` 是否也有 `soft lockup` + 同一时间窗口的 `NETDEV WATCHDOG`/`TX timeout`——两块不同厂商网卡同时超时，是判断"这是全局性冻结而不是单一子系统卡住"的最直接证据。
3. **`stop_machine()` 的真实触发源**：`livepatch` 已被源码核查排除（其 `stop_machine` 只在补丁启用/卸载时各跑一次，早已在开机 21 秒时完成）。若要坐实这次 `migration/N` 的来源，需要看当时是否有人在跑 `ftrace`/`kprobe`/`perf probe`（对应 `ftrace_run_stop_machine`）、是否有驱动在增删 MTRR 区间、是否有微码热加载任务、或时钟源切换记录。
4. 若机器上虚机较多、内存较大，`kvm-nx-lpage-re` 的周期性 zap 也会周期性发起全 vCPU TLB flush，核数越多一次广播的尾延迟越大，可评估是否需要调整 NX 大页回收周期（`kvm.nx_huge_pages_recovery_period_ms` 等参数）。
5. 本次 dmesg 只是一次孤立快照，且已确认无法从中直接找到"哪个核关中断关了 22 秒"的证据；`e203`/`e210` 复现时最好能拿到连续的 `dmesg`/`sosreport`/BMC 日志，而不只是单次 soft lockup 的打印，才能在软件和硬件两条线之间做出取舍。
6. **跟踪 KVM MSR debug 日志限流热补丁的落地情况**：受影响 guest 为 Windows 时，部分 KVM 寄存器不支持会导致 host 侧高频打印 MSR debug 日志，有排查思路认为这些高频打印可能干扰/掩盖故障现场，值得先上限流观察资源池复现率是否变化——但这只是缓解疑似干扰因素，不能替代第 0-3 条的根因证据收集。
