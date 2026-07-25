# xfs_inodegc_flush 导致 statfs 阻塞引发大面积 hung_task

## 环境信息

本文实际涉及**两台独立主机**的两次独立故障（命中同一处内核代码，但不是同一次事件），证据完整度不一样，先在这里列清楚，避免正文和附录混着看的时候搞混：

| | 主机 1（本文正文分析对象） | 主机 2（附录：长沙42-os） |
|---|---|---|
| 资源池 | 华东1 资源池 | 长沙42-os |
| 主机 ID | `bef9d276-d2af-7729-90e9-939b` | IP `43.65.3.106`，主机 uuid `ae25d315-e9e3-4800-3069-1b9aff4a5d04` |
| 故障时间 | 2026-07-21 15:11–15:13 | 2026-07-21 08:20–09:37 |
| **现有证据** | `dmesg.click.7.21.1.log` **+ vmcore**（已用 `crash` 复核，坐实 ABBA 死锁） | 仅 `messages-20260724` 日志，**没有 vmcore**，结论止步于 dmesg 级别推断 |

以下环境细节针对**主机 1**（正文的主要分析对象）：

- 磁盘故障处理，华东1 资源池
- 主机ID：bef9d276-d2af-7729-90e9-939b
- 内核版本：`5.10.0-136.12.0.92.ctl3.x86_64`
- Hypervisor：KVM（`GoStack Foundation OpenStack Nova`）
- 故障磁盘：`vdb`，virtio-blk，6291456000 个 512 字节逻辑块（3.22 TB / 2.93 TiB），XFS V5 文件系统
- 系统盘 `vda1` 同为 XFS，未受影响

主机 2 的环境信息见后面「附：同一处代码在另一台主机（长沙42-os）上的独立复现」一节的故障通报原文。

## 问题现象

`2026-07-21 15:11:00` 与 `15:13:03`（hung_task 检测器默认 120 秒巡检一次，两次告警是同一次卡死在相邻巡检周期分别被记录，阻塞时长从 122 秒累加到 245 秒），大量进程进入 D 状态：

```
INFO: task 19100_node_expo:2394 blocked for more than 122 seconds.
INFO: task BgSchPool:3519866 blocked for more than 122 seconds.
INFO: task AsyncMetrics:3520121 blocked for more than 122 seconds.
INFO: task kworker/u34:3:3702130 blocked for more than 122 seconds.
INFO: task kworker/2:0:3702423 blocked for more than 122 seconds.
INFO: task telegraf:2335 blocked for more than 122 seconds.
INFO: task 19100_node_expo:2394 blocked for more than 245 seconds.
INFO: task titanagent:2970602 blocked for more than 122 seconds.
INFO: task BgSchPool:3519866 blocked for more than 245 seconds.
INFO: task AsyncMetrics:3520121 blocked for more than 245 seconds.
```

被卡住的进程分两类：

1. **一堆用户态监控/agent 进程**（`19100_node_expo`、`BgSchPool`、`AsyncMetrics`、`telegraf`、`titanagent`），都卡在同一条调用链上；
2. **两个 XFS 内核工作线程**（`kworker/u34:3` 对应 `writeback`，`kworker/2:0` 对应 `xfs-inodegc/vdb`），卡在 buffer 锁上。

日志里反复出现的 `"echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.` 只是 hung_task 检测器每条告警都会打印的固定提示语，并不代表发生了多次独立故障；整个 dmesg 里更早的 Dec 25 cgroup OOM、Jul 16/17 的 `SLUB: Unable to allocate memory`（skbuff_head_cache 分配失败）是另外的、时间上不相关的历史事件。

## 日志分析

### 用户态进程：卡在 statfs -> xfs_inodegc_flush

```
task:19100_node_expo state:D stack:    0 pid: 2394 ppid:     1 flags:0x00000080
Call Trace:
 __schedule+0x2ea/0x640
 schedule+0x46/0xb0
 schedule_timeout+0x240/0x2b0
 ? ttwu_queue+0x41/0xc0
 ? __legitimize_path+0x27/0x60
 wait_for_common+0x97/0x140
 __flush_work.isra.0+0x60/0x80
 ? flush_workqueue_prep_pwqs+0x140/0x140
 ? is_vmalloc_or_module_addr+0x24/0x30
 ? __check_object_size.part.0+0x135/0x1c0
 xfs_inodegc_flush.part.0+0x3e/0x90 [xfs]
 xfs_fs_statfs+0x2d/0x1a0 [xfs]
 statfs_by_dentry+0x64/0x90
 user_statfs+0x57/0xc0
 __se_sys_statfs+0x25/0x60
 do_syscall_64+0x3d/0x80
 entry_SYSCALL_64_after_hwframe+0x61/0xc6
```

`BgSchPool`、`AsyncMetrics`、`telegraf`、`titanagent` 的调用栈与此完全一致，都是调用 `statfs(2)`/`statvfs(3)` 查询磁盘使用率（常见于监控采集、容量上报等定时任务），在内核里落到：

```
sys_statfs -> user_statfs -> statfs_by_dentry -> xfs_fs_statfs -> xfs_inodegc_flush -> __flush_work
```

`xfs_fs_statfs()` 为了拿到准确的空闲 inode/block 计数，会先调用 `xfs_inodegc_flush()` 把当前 CPU 上挂起的 inode 延迟回收（inodegc）工作刷完，也就是 `flush_work()` 等待 `xfs-inodegc` workqueue 上的 worker 跑完。如果这个 worker 本身卡住，所有调用 `statfs` 的进程都会被一起拖住——这正是本次告警里进程数量远多于"真正故障点"的原因。

### xfs-inodegc worker：卡在 AGF buffer 锁上

```
task:kworker/2:0     state:D stack:    0 pid:3702423 ppid:     2 flags:0x00004080
Workqueue: xfs-inodegc/vdb xfs_inodegc_worker [xfs]
Call Trace:
 __schedule+0x2ea/0x640
 schedule+0x46/0xb0
 schedule_timeout+0x240/0x2b0
 __down+0x83/0xe0
 down+0x3b/0x50
 xfs_buf_lock+0x2c/0xb0 [xfs]
 xfs_buf_find+0x345/0x640 [xfs]
 xfs_buf_get_map+0x44/0x230 [xfs]
 xfs_buf_read_map+0x54/0x280 [xfs]
 xfs_trans_read_buf_map+0x13c/0x2a0 [xfs]
 xfs_read_agf+0x87/0x110 [xfs]
 xfs_alloc_read_agf+0x33/0x190 [xfs]
 xfs_alloc_fix_freelist+0x326/0x390 [xfs]
 xfs_alloc_vextent+0x23b/0x460 [xfs]
 __xfs_inobt_alloc_block.isra.0+0xbf/0x1a0 [xfs]
 __xfs_btree_split+0xec/0x660 [xfs]
 xfs_btree_split+0x4b/0x100 [xfs]
 xfs_btree_make_block_unfull+0x197/0x1d0 [xfs]
 xfs_btree_insrec+0x4ad/0x590 [xfs]
 xfs_btree_insert+0xa8/0x1f0 [xfs]
 xfs_difree_finobt+0xae/0x250 [xfs]
 xfs_difree+0x125/0x1a0 [xfs]
 xfs_ifree+0x88/0x150 [xfs]
 xfs_inactive_ifree.isra.0+0xa2/0x1b0 [xfs]
 xfs_inactive+0xf5/0x170 [xfs]
 xfs_inodegc_inactivate+0x16/0x50 [xfs]
 xfs_inodegc_worker+0x76/0xc0 [xfs]
 process_one_work+0x1ad/0x350
 worker_thread+0x49/0x310
 kthread+0xfb/0x140
 ret_from_fork+0x1f/0x30
```

释放一个 inode（`xfs_inactive_ifree` -> `xfs_difree`）需要更新 inobt/finobt，触发了一次 btree 分裂（`__xfs_btree_split`），分裂又需要从 AG 空闲块里分配新块（`xfs_alloc_vextent` -> `xfs_alloc_fix_freelist` -> `xfs_alloc_read_agf`），于是要去读/锁 AGF（Allocation Group Free space）buffer。栈顶停在 `xfs_buf_lock -> down()`，即在等待这块 buffer 的信号量——说明该 buffer 当前被别的上下文持有且迟迟没有释放。

### writeback worker：卡在 inode cluster buffer 锁上

```
task:kworker/u34:3   state:D stack:    0 pid:3702130 ppid:     2 flags:0x00004080
Workqueue: writeback wb_workfn (flush-253:16)
Call Trace:
 __schedule+0x2ea/0x640
 schedule+0x46/0xb0
 schedule_timeout+0x240/0x2b0
 ? xfs_buf_read_map+0x54/0x280 [xfs]
 ? xfs_btree_read_buf_block.constprop.0+0x9d/0xe0 [xfs]
 ? xfs_trans_add_item+0x33/0xa0 [xfs]
 __down+0x83/0xe0
 down+0x3b/0x50
 xfs_buf_lock+0x2c/0xb0 [xfs]
 xfs_buf_find+0x345/0x640 [xfs]
 xfs_buf_get_map+0x44/0x230 [xfs]
 xfs_buf_read_map+0x54/0x280 [xfs]
 xfs_trans_read_buf_map+0x13c/0x2a0 [xfs]
 xfs_imap_to_bp+0x61/0xb0 [xfs]
 xfs_trans_log_inode+0x1b7/0x270 [xfs]
 xfs_bmap_btalloc+0x773/0x840 [xfs]
 xfs_bmapi_allocate+0xd0/0x2e0 [xfs]
 xfs_bmapi_convert_delalloc+0x293/0x480 [xfs]
 xfs_map_blocks+0x1b0/0x400 [xfs]
 iomap_writepage_map+0x15c/0x570
 write_cache_pages+0x169/0x4f0
 iomap_writepages+0x1c/0x40
 xfs_vm_writepages+0x64/0x90 [xfs]
 do_writepages+0x31/0xc0
 __writeback_single_inode+0x39/0x200
 writeback_sb_inodes+0x203/0x540
 __writeback_inodes_wb+0x4c/0xd0
 wb_writeback+0x1d8/0x2a0
 wb_check_old_data_flush+0xb6/0xc0
 wb_do_writeback+0xc1/0x180
 wb_workfn+0x5a/0x180
 process_one_work+0x1ad/0x350
 worker_thread+0x49/0x310
 kthread+0xfb/0x140
 ret_from_fork+0x1f/0x30
```

`writeback` 内核线程在做延迟分配落盘（`xfs_bmap_btalloc` 里已经完成一次 extent 分配），随后要把新分配结果写回 inode（`xfs_trans_log_inode` -> `xfs_imap_to_bp`），需要读取该 inode 所在的 inode cluster buffer，同样卡在 `xfs_buf_lock -> down()`。

## 根因分析

> **本节结论已被后面「vmcore 复核」一节的实测数据推翻，保留在此仅作为"仅凭 dmesg 能得出的初步判断"的记录，实际根因见下面的更新版本。**

两个内核工作线程最初看起来都不是"死锁"在 CPU 上打转，而是在等 **buffer 信号量**（`xfs_buf_lock`），dmesg 层面无法判断是等一次 buffer I/O（读盘）完成，还是等持锁者释放锁：

```
用户态 statfs 一堆进程
        │  等待
        ▼
xfs-inodegc/vdb worker（卡在 AGF buffer 锁 / down()）
                                  │
                                  ▼（同一块 vdb 上的另一 buffer）
writeback (flush-253:16) worker（卡在 inode cluster buffer 锁 / down()）
                                  │
                                  ▼
                          vdb 磁盘 I/O 长时间未完成 ← 这一步是错的，见下文
```

（**已订正**：拿到 vmcore 之后用 `crash` 核实，两个 worker 卡的确实是各自不同的 buffer——`xfs-inodegc/vdb` 卡在 **AGF**，`writeback` 卡在一块 **inode cluster buffer**——但两者并不是分别独立等磁盘 I/O，而是构成了一个**真实的 ABBA 循环死锁**：详见「vmcore 复核」一节，用 `b_transp -> t_ticket -> t_task` 这条内核自身维护的关联关系完整验证过。）

真正让故障"扩散"的是 XFS 的一个设计点：`xfs_fs_statfs()` 在返回 df/statfs 结果前会调用 `xfs_inodegc_flush()` 等待 inodegc 全部跑完，以保证空闲块/inode 计数准确。这本是为了拿到精确结果，但代价是：**只要 inodegc worker 因为任何原因（包括真死锁）被卡住，所有调用 statfs/statvfs 的进程都会被一起挂起**，包括各类监控 agent（`node_exporter`、`telegraf`、`titanagent` 等定时采集磁盘使用率的组件），造成"一次内核态死锁"被放大成"大面积业务进程 D 状态"。

## vmcore 复核：完整排查过程与最终结论

拿到了这台故障机（`bef9d276-d2af-7729-90e9-939b`）对应的 vmcore 之后，用 `crash 8.0.2` + 匹配的 `vmlinux`/`xfs.ko` debuginfo 做了进一步排查。这一节把每一步实际跑过的 `crash` 命令、拿到的结果、以及每一步的推理都完整记录下来（包括走过的弯路和中途的误判），最终坐实了一个此前基于 dmesg 分析不出来的结论：**这是一个真实的 ABBA 循环死锁，不是等磁盘慢 I/O**。

### 环境准备

vmcore 是在 host 上用 `virsh qemu-monitor-command --hmp <vm> "dump-guest-memory -z /path/to/xxx.img"` 导出的（`-z` = kdump 压缩格式，zlib 压缩，不是标准 ELF，`file` 认不出来、`makedumpfile` 也不能直接处理，但 `crash` 能原生识别）。

先加载全部模块符号：

```
mod -S
```

输出（节选，都是跟 XFS 无关的存储/控制台/显卡驱动加载失败，可以忽略）：

```
mod: cannot find or load object file for ata_piix module
mod: cannot find or load object file for failover module
mod: cannot find or load object file for virtio_blk module
mod: cannot find or load object file for serio_raw module
mod: cannot find or load object file for net_failover module
mod: cannot find or load object file for virtio_console module
mod: cannot find or load object file for ata_generic module
mod: cannot find or load object file for cirrus module
mod: cannot find or load object file for ghash_clmulni_intel module
mod: cannot find or load object file for crc32c_intel module
```

真正要用的是 `xfs.ko`，先确认机器上这个模块文件的状态：

```
ls -l /lib/modules/5.10.0-136.12.0.92.ctl3.x86_64/kernel/fs/xfs/xfs.ko.xz
```

输出：

```
-rw-r--r-- 1 root root 514780 Apr 29  2025 /lib/modules/5.10.0-136.12.0.92.ctl3.x86_64/kernel/fs/xfs/xfs.ko.xz
```

这个是压缩过的裸模块（`.xz`），没有调试符号。真正带 DWARF 调试信息的是单独的 debuginfo 包：

```
ls -l /lib/debug/lib/modules/5.10.0-136.12.0.92.ctl3.x86_64/kernel/fs/xfs/xfs.ko-5.10.0-136.12.0.92.ctl3.x86_64.debug
```

输出：

```
-r--r--r-- 1 root root 35960528 Apr 29  2025 /lib/debug/lib/modules/5.10.0-136.12.0.92.ctl3.x86_64/kernel/fs/xfs/xfs.ko-5.10.0-136.12.0.92.ctl3.x86_64.debug
```

（35MB，比裸 `.ko` 大得多，确认这个才是带符号的）。用到的 debuginfo 汇总：

- 内核本体：`/usr/lib/debug/lib/modules/5.10.0-136.12.0.92.ctl3.x86_64/vmlinux`
- `xfs.ko` 模块符号：用 `mod -s xfs /lib/debug/lib/modules/5.10.0-136.12.0.92.ctl3.x86_64/kernel/fs/xfs/xfs.ko-5.10.0-136.12.0.92.ctl3.x86_64.debug` 加载。

### 第 1 步：`ps | grep UN` —— D 状态任务的真实规模比 dmesg 大得多

结果：dump 时刻共有 **286 个**处于 D 状态的任务，远多于 dmesg 里能看到的 10 条告警——印证了前面「`hung_task_warnings` 到底是不是 10」那一节的判断：dmesg 只是打满额度后被静默截断的一角。除了 dmesg 里已知的 `kworker/2:0`（xfs-inodegc，PID 3702423）、`kworker/u34:3`（writeback，PID 3702130）、`telegraf`、`19100_node_expo`、`titanagent`，还包括：

- `[xfsaild/vdb]`（AIL 推送线程，PID 5062）——后续 `bt -f 5062` 核实发现它只是在正常的周期性 `schedule_timeout` 休眠里（XFS 用 `TASK_KILLABLE` 休眠，`ps` 会显示成 D 状态），栈里没有任何 `xfs_buf_lock`/`down`，并没有真的卡在 buffer 锁上，属于误报，排除。
- 大量该主机上某 Java 进程（父 PID 3519292）的 `HTTPHandler`/`BgSchPool`/`AsyncMetrics` 线程，以及一批 `(ostnamed)`、`df`、`ls`。

### 第 2 步：`bt -a -f` —— dump 时刻系统是完全静止的

结果：全部 16 个 CPU 都停在 `swapper/N` 的 `default_idle`，没有任何一个 CPU 正在处理 I/O 完成或跑软中断。说明持锁的上下文此刻也必然是睡眠状态，不在任何 CPU 上运行。

### 第 3 步：`foreach bt -f UN` —— 拿到全部 D 状态任务的调用栈 + 原始栈数据

一次性把 286 个 D 状态任务的调用栈和栈帧原始内存都导出来（后来发现 `foreach bt -f`不加过滤条件产出的文件跟 `foreach bt -f UN` 完全一致，说明这台机器上睡眠中的任务，含用户态 futex 等待，实际有近三千个，`UN` 过滤在这次操作里没生效，等于是全量数据）。

### 第 4 步：第一次尝试定位 `xfs_buf`（猜错了，记录下来避免同样的坑）

肉眼在 `kworker/2:0` 和 `kworker/u34:3` 各自的 `__down`/`down`/`xfs_buf_lock` 栈帧原始数据里，找到一对两边都出现的地址，凭直觉猜测 `bp = ff38011c00dd7000`：

```
struct xfs_buf ff38011c00dd7000
```

输出：

```
struct xfs_buf {
  b_rhash_head = {
    next = 0xff38011c02e36000
  },
  b_bn = 172846663860225,
  b_length = 16777473,
  b_hold = {
    counter = 514
  },
  b_lru_ref = {
    counter = 6
  },
  b_flags = 63802947,
  b_sema = {
    lock = {
      raw_lock = {
        {
          val = {
            counter = 27701546
          },
          {
            locked = 42 '*',
            pending = 177 '\261'
          },
          {
            locked_pending = 45354,
            tail = 422
          }
        }
      }
    },
    count = 52,
    wait_list = {
      next = 0x9f94000006498,
      prev = 0x0
    }
  },
  ...（b_lru/b_lock/b_state/b_io_error/b_waiters/b_list/b_pag/b_mount/b_target/
      b_addr/b_ioend_work/b_iowait/b_log_item/b_maps/b_page_count/b_offset/
      b_error/b_last_error/b_ops 等字段——数值同样全部异常，此处从略）
}
```

结果：`b_hold.counter = 514`、`b_flags = 63802947`、`b_sema.wait_list.next = 0x9f94000006498` 等字段全是明显不合理的爆炸值。

```
kmem ff38011c00dd7000
```

输出：

```
CACHE            OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ff380114c0034b00     1024    1500019   1633888  51059   32k  kmalloc-1k
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffe7f9c821037400  ff38011c00dd0000     1     32         32     0
  FREE / [ALLOCATED]
  [ff38011c00dd7000]

      PAGE            PHYSICAL      MAPPING            INDEX CNT FLAGS
ffe7f9c8210375c0     840dd7000     dead000000000400       0   0  57fffffc0000000
```

结果：这个地址属于通用 `kmalloc-1k` 缓存，不是 XFS 专用的 buffer slab——说明地址本身是个合法对象，但**类型猜错了**（不是 `xfs_buf`）。这个地址后来在第 8 步查明其实是 `xfs_perag`（AGI/AGF 所在 AG 的元数据结构），只是这一步误当成 `xfs_buf` 去解析，字段全部错位导致看起来像垃圾值。

### 第 5 步：改用反汇编精确计算，不再肉眼猜

先拿真实的字段偏移：

```
struct -o xfs_buf
```

输出：

```
struct xfs_buf {
    [0] struct rhash_head b_rhash_head;
    [8] xfs_daddr_t b_bn;
   [16] int b_length;
   [20] atomic_t b_hold;
   [24] atomic_t b_lru_ref;
   [28] xfs_buf_flags_t b_flags;
   [32] struct semaphore b_sema;
   [56] struct list_head b_lru;
   [72] spinlock_t b_lock;
   [76] unsigned int b_state;
   [80] int b_io_error;
   [88] wait_queue_head_t b_waiters;
  [112] struct list_head b_list;
  [128] struct xfs_perag *b_pag;
  [136] struct xfs_mount *b_mount;
  [144] xfs_buftarg_t *b_target;
  [152] void *b_addr;
  [160] struct work_struct b_ioend_work;
  [224] struct completion b_iowait;
  [256] struct xfs_buf_log_item *b_log_item;
  [264] struct list_head b_li_list;
  [280] struct xfs_trans *b_transp;
  [288] struct page **b_pages;
  [296] struct page *b_page_array[2];
  [312] struct xfs_buf_map *b_maps;
  [320] struct xfs_buf_map __b_map;
  [336] int b_map_count;
  [340] atomic_t b_pin_count;
  [344] atomic_t b_io_remaining;
  [348] unsigned int b_page_count;
  [352] unsigned int b_offset;
  [356] int b_error;
  [360] int b_retries;
  [368] unsigned long b_first_retry_time;
  [376] int b_last_error;
  [384] const struct xfs_buf_ops *b_ops;
}
SIZE: 392
```

结果：`b_sema` 在偏移 **0x20**，不是第 4 步瞎猜时以为的 `0x98`（`0x98` 其实是 `b_addr` 字段的偏移）。

再反汇编 `down()`/`__down()`，精确搞清楚 `sem` 参数在栈上/寄存器里的传递路径：

```
dis -l down 30
```

输出：

```
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 54
0xffffffff90d53ee0 <down>:        nopl   0x0(%rax,%rax,1) [FTRACE NOP]
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 57
0xffffffff90d53ee5 <down+5>:       push   %rbp
0xffffffff90d53ee6 <down+6>:       mov    %rdi,%rbp
0xffffffff90d53ee9 <down+9>:       sub    $0x8,%rsp
0xffffffff90d53eed <down+13>:      call   0xffffffff916fb560 <_raw_spin_lock_irqsave>
0xffffffff90d53ef2 <down+18>:      mov    %rax,%rsi
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 58
0xffffffff90d53ef5 <down+21>:      mov    0x4(%rbp),%eax
0xffffffff90d53ef8 <down+24>:      test   %eax,%eax
0xffffffff90d53efa <down+26>:      je     0xffffffff90d53f0f <down+47>
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 59
0xffffffff90d53efc <down+28>:      sub    $0x1,%eax
0xffffffff90d53eff <down+31>:      mov    %rbp,%rdi
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 59
0xffffffff90d53f02 <down+34>:      mov    %eax,0x4(%rbp)
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 62
0xffffffff90d53f05 <down+37>:      add    $0x8,%rsp
0xffffffff90d53f09 <down+41>:      pop    %rbp
0xffffffff90d53f0a <down+42>:      jmp    0xffffffff916fb4e0 <__cpuidle_text_end>
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 61
0xffffffff90d53f0f <down+47>:      mov    %rbp,%rdi
0xffffffff90d53f12 <down+50>:      mov    %rsi,(%rsp)
0xffffffff90d53f16 <down+54>:      call   0xffffffff916f8cc0 <__down>
0xffffffff90d53f1b <down+59>:      mov    (%rsp),%rsi
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 62
0xffffffff90d53f1f <down+63>:      mov    %rbp,%rdi
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 63
0xffffffff90d53f22 <down+66>:      add    $0x8,%rsp
0xffffffff90d53f26 <down+70>:      pop    %rbp
0xffffffff90d53f27 <down+71>:      jmp    0xffffffff916fb4e0 <__cpuidle_text_end>
0xffffffff90d53f2c <down+76>:      nopl   0x0(%rax)
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 76
0xffffffff90d53f30 <down_interruptible>: nopl 0x0(%rax,%rax,1) [FTRACE NOP]
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 80
0xffffffff90d53f35 <down_interruptible+5>: push %r12
0xffffffff90d53f37 <down_interruptible+7>: push %rbp
```

```
dis -l __down 40
```

输出：

```
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 236
0xffffffff916f8cc0 <__down>:       nopl   0x0(%rax,%rax,1) [FTRACE NOP]
0xffffffff916f8cc5 <__down+5>:     push   %r12
0xffffffff916f8cc7 <__down+7>:     mov    %rdi,%r12
0xffffffff916f8cca <__down+10>:    push   %rbp
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 209
0xffffffff916f8ccb <__down+11>:    lea    0x8(%rdi),%rbp
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 236
0xffffffff916f8ccf <__down+15>:    push   %rbx
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../include/linux/list.h: 67
0xffffffff916f8cd0 <__down+16>:    mov    %rbp,%rdx
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 236
0xffffffff916f8cd3 <__down+19>:    sub    $0x30,%rsp
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../include/linux/list.h: 100
0xffffffff916f8cd7 <__down+23>:    mov    0x10(%rdi),%rbx
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 236
0xffffffff916f8cdb <__down+27>:    mov    %gs:0x28,%rax
0xffffffff916f8ce4 <__down+36>:    mov    %rax,0x28(%rsp)
0xffffffff916f8ce9 <__down+41>:    xor    %eax,%eax
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../include/linux/list.h: 67
0xffffffff916f8ceb <__down+43>:    mov    %rsp,%rdi
0xffffffff916f8cee <__down+46>:    mov    %rbx,%rsi
0xffffffff916f8cf1 <__down+49>:    call   0xffffffff91170960 <__list_add_valid>
0xffffffff916f8cf6 <__down+54>:    test   %al,%al
0xffffffff916f8cf8 <__down+56>:    je     0xffffffff916f8d0b <__down+75>
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../include/linux/list.h: 70
0xffffffff916f8cfa <__down+58>:    mov    %rsp,0x10(%r12)
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../include/linux/list.h: 71
0xffffffff916f8cff <__down+63>:    mov    %rbp,(%rsp)
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../include/linux/list.h: 72
0xffffffff916f8d03 <__down+67>:    mov    %rbx,0x8(%rsp)
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../include/linux/list.h: 73
0xffffffff916f8d08 <__down+72>:    mov    %rsp,(%rbx)
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../arch/x86/include/asm/current.h: 15
0xffffffff916f8d0b <__down+75>:    mov    %gs:0x1f880,%rbx
0xffffffff916f8d14 <__down+84>:    movabs $0x7fffffffffffffff,%rbp
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 210
0xffffffff916f8d1e <__down+94>:    mov    %rbx,0x10(%rsp)
/usr/src/debug/kernel-5.10.0-136.12.0.92.ctl3.x86_64/.../kernel/locking/semaphore.c: 211
0xffffffff916f8d23 <__down+99>:    movb   $0x0,0x18(%rsp)
```

关键发现：`down()` 入口 `mov %rdi,%rbp` 把 `%rbp` 当成普通的 callee-saved 寄存器复用来保存 `sem`（不是常规的帧指针语义），一直保持到调用 `__down` 之后；`__down()` 入口 `push %r12; mov %rdi,%r12; push %rbp; lea 0x8(%rdi),%rbp; push %rbx; sub $0x30,%rsp`——这条 `push %rbp` 压的是**调用者（down）的 `%rbp`**，也就是 `sem` 本身，发生在它被自己改写成 `sem+8`（`&sem->wait_list`）之前。

用 `#4 down at ffffffff90d53f1b` 这个已知的返回地址反推 `__down` 帧的 `final_RSP = 返回地址位置 - 0x48`（3 次 push + `sub $0x30`），再从 `final_RSP + 0x38` 读出压栈保存的 `sem`。对 `kworker/2:0` 算出：

```
sem = ff380114e72ecc60
bp（sem - 0x20）= ff380114e72ecc40
```

交叉验证：`down()` 自己那一帧（调用者 `xfs_buf_lock` 的 `bp` 局部变量，作为"调用者的旧 rbp"被压在 `down` 帧里）里原样出现了 `ff380114e72ecc40`，跟反推结果完全对上。

### 第 6 步：验证新地址是真的

```
kmem ff380114e72ecc40
```

输出：

```
CACHE            OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ff38011c06022600      392     169198    215532   5987   16k  xfs_buf
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffe7f9c8049cbb00  ff380114e72ec000     0     36         36     0
  FREE / [ALLOCATED]
  [ff380114e72ecc40]

      PAGE            PHYSICAL      MAPPING            INDEX CNT FLAGS
ffe7f9c8049cbb00     1272ec000     ff38011c06022600       0   1  17ffffc0010200 slab,head
```

结果：`CACHE ff38011c06022600  OBJSIZE 392  NAME xfs_buf`——对象大小跟 `struct -o xfs_buf` 算出的 `SIZE: 392` 完全一致，处于 `[ALLOCATED]`。这次是真的。

```
struct xfs_buf.b_bn,b_length,b_flags,b_hold ff380114e72ecc40
```

输出：

```
b_bn = 1572864001,
b_length = 1,
b_flags = 2097200,
b_hold = {
  counter = 4
},
```

结果：`b_bn = 1572864001`、`b_length = 1`、`b_flags = 2097200`、`b_hold.counter = 4`——数值都合理。`b_flags = 2097200` 解码：

```
2097200 = _XBF_KMEM(1<<21) | XBF_DONE(1<<5) | XBF_ASYNC(1<<4)
```

**`XBF_DONE` 已置位——这块 buffer 的 I/O 早就完成了**，不是还在等磁盘返回；问题变成"I/O 已经做完，但锁一直没被释放，持锁的上下文卡在哪"。

### 第 7 步：`search -t` + `b_sema` —— 纠正一处中途误判

```
search -t ff380114e72ecc40
```

输出：

```
PID: 3519984  TASK: ff380115fe490000  CPU: 2    COMMAND: "Common"
ff7ad4f7c9dbd8d8: ff380114e72ecc40

PID: 3519989  TASK: ff380117fc763180  CPU: 5    COMMAND: "Common"
ff7ad4f7c9c038d8: ff380114e72ecc40

PID: 3520026  TASK: ff3801158200b180  CPU: 1    COMMAND: "SystemLogFlush"
ff7ad4f7c9d27440: ff380114e72ecc40

PID: 3520043  TASK: ff380115204e0000  CPU: 13   COMMAND: "ThreadPool"
ff7ad4f7c9f3f2e8: ff380114e72ecc40

PID: 3520048  TASK: ff3801166ef7b180  CPU: 1    COMMAND: "SystemLogFlush"
ff7ad4f7c9f67440: ff380114e72ecc40

PID: 3520050  TASK: ff380114f607b180  CPU: 3    COMMAND: "ThreadPool"
ff7ad4f7c9f77690: ff380114e72ecc40

PID: 3520084  TASK: ff38011c01670000  CPU: 1    COMMAND: "ThreadPool"
ff7ad4f7ccb132e8: ff380114e72ecc40

PID: 3520086  TASK: ff38011ce905b180  CPU: 7    COMMAND: "SystemLogFlush"
ff7ad4f7ccb23440: ff380114e72ecc40

PID: 3520117  TASK: ff38011e24978000  CPU: 12   COMMAND: "ThreadPool"
ff7ad4f7ccf3b690: ff380114e72ecc40

PID: 3520130  TASK: ff38011cf3ffb180  CPU: 2    COMMAND: "ThreadPool"
ff7ad4f7ccfa32e8: ff380114e72ecc40

PID: 3702130  TASK: ff38011cd93eb180  CPU: 13   COMMAND: "kworker/u34:3"
ff7ad4f7cdcb36b0: ff380114e72ecc40

PID: 3702423  TASK: ff380115e4fe0000  CPU: 2    COMMAND: "kworker/2:0"
ff7ad4f7cc92f4a0: ff380114e72ecc40
ff7ad4f7cc92f568: ff380114e72ecc40
ff7ad4f7cc92f5d8: ff380114e72ecc40
ff7ad4f7cc92f868: ff380114e72ecc40
```

结果：命中了十几个任务，包括 `kworker/u34:3` 和一堆 `Common`/`ThreadPool`/`SystemLogFlush`。逐个用 `bt` 核实这些 `ThreadPool`/`Common`/`SystemLogFlush` 命中的完整调用栈，发现清一色是纯用户态 `futex_wait`（8 层：`entry_SYSCALL_64 -> do_syscall_64 -> __se_sys_futex -> do_futex -> futex_wait -> futex_wait_queue_me -> schedule -> __schedule`），跟 XFS 毫无关系，确认是这些线程历史上残留在内核栈里、尚未被覆盖的噪音数据。

```
struct xfs_buf.b_sema ff380114e72ecc40
```

输出：

```
b_sema = {
  lock = {
    raw_lock = {
      {
        val = {
          counter = 0
        },
        {
          locked = 0 '\000',
          pending = 0 '\000'
        },
        {
          locked_pending = 0,
          tail = 0
        }
      }
    }
  },
  count = 0,
  wait_list = {
    next = 0xff7ad4f7cc92f580,
    prev = 0xff7ad4f7cc92f580
  }
},
```

结果：`count = 0`（确实锁着），`wait_list.next = wait_list.prev = 0xff7ad4f7cc92f580`——这个地址正好就是第 5 步算出的 `kworker/2:0` 自己 `__down` 帧的 `final_RSP`，即 **wait_list 里只有 `kworker/2:0` 自己一个真实等待者**。这也纠正了此前的一个猜测：`kworker/u34:3` 栈里出现的同一个地址，其实是它**更早、已经用完并释放**的一次 AGF 访问留下的残留数据（对应 dmesg 里那几个带 `?` 号的旧帧），不是它当前真正等待的对象。

### 第 8 步：`b_pag` + `struct -o xfs_perag` —— 排除四个假设

```
struct xfs_buf.b_pag,b_bn ff380114e72ecc40
```

输出：

```
b_pag = 0xff38011c00dd7000,
b_bn = 1572864001,
```

结果：`b_pag = 0xff38011c00dd7000`（这正是第 4 步误判的那个地址，真相大白：它是一个 `xfs_perag`，不是 `xfs_buf`）。`pag_agno = 1`，确认是 **AG #1**。

```
struct -o xfs_perag
```

输出：

```
struct xfs_perag {
    [0] struct xfs_mount *pag_mount;
    [8] xfs_agnumber_t pag_agno;
   [12] atomic_t pag_ref;
   [16] char pagf_init;
   [17] char pagi_init;
   [18] char pagf_metadata;
   [19] char pagi_inodeok;
   [20] uint8_t pagf_levels[3];
   [23] bool pagf_agflreset;
   [24] uint32_t pagf_flcount;
   [28] xfs_extlen_t pagf_freeblks;
   [32] xfs_extlen_t pagf_longest;
   [36] uint32_t pagf_btreeblks;
   [40] xfs_agino_t pagi_freecount;
   [44] xfs_agino_t pagi_count;
   [48] xfs_agino_t pagl_pagino;
   [52] xfs_agino_t pagl_leftrec;
   [56] xfs_agino_t pagl_rightrec;
   [60] uint16_t pag_checked;
   [62] uint16_t pag_sick;
   [64] spinlock_t pag_state_lock;
   [68] spinlock_t pagb_lock;
   [72] struct rb_root pagb_tree;
   [80] unsigned int pagb_gen;
   [88] wait_queue_head_t pagb_wait;
  [112] atomic_t pagf_fstrms;
  [116] spinlock_t pag_ici_lock;
  [120] struct xarray pag_ici_root;
  [136] int pag_ici_reclaimable;
  [144] unsigned long pag_ici_reclaim_cursor;
  [152] spinlock_t pag_buf_lock;
  [160] struct rhashtable pag_buf_hash;
  [328] struct callback_head callback_head;
  [344] int pagb_count;
  [348] struct xfs_ag_resv pag_meta_resv;
  [360] struct xfs_ag_resv pag_rmapbt_resv;
  [376] struct delayed_work pag_blockgc_work;
  [568] uint8_t pagf_refcount_level;
  [576] struct rhashtable pagi_unlinked_hash;
}
```

```
struct xfs_perag.pag_agno,pagf_freeblks,pagf_longest,pagf_flcount,pagf_btreeblks,pagi_freecount,pagi_count,pagb_lock,pagb_tree,pagb_count ff38011c00dd7000
```

输出：

```
pag_agno = 1,
pagf_freeblks = 63802947,
pagf_longest = 27701546,
pagf_flcount = 6,
pagf_btreeblks = 52,
pagi_freecount = 25752,
pagi_count = 653632,
pagb_lock = {
  {
    rlock = {
      raw_lock = {
        {
          val = {
            counter = 0
          },
          {
            locked = 0 '\000',
            pending = 0 '\000'
          },
          {
            locked_pending = 0,
            tail = 0
          }
        }
      }
    }
  }
},
pagb_tree = {
  rb_node = 0x0
},
pagb_count = 0,
```

结果逐一核对四个常见假设：

1. **AG 空间不足**：`pagf_freeblks`/`pagf_longest` 换算下来空闲空间是几百 GB 量级，排除。
2. **忙碌区间（busy extent）等日志刷盘**：`pagb_tree = { rb_node = 0x0 }`，`pagb_count = 0`，`pagb_lock` 未上锁，排除。
3. **日志子系统本身卡住**：在 `foreach_bt_f.txt` 全量数据里搜索 `xlog_*`/`xfs_log_*`，零命中，排除。
4. **文件系统已经 shutdown/损坏**：

   ```
   log -T | grep -iE "xfs|corrupt|shutdown|Internal error|Metadata I/O error"
   ```

   输出：

   ```
   [Tue Sep  9 14:21:06 CST 2025] SGI XFS with ACLs, security attributes, quota, no debug enabled
   [Tue Sep  9 14:21:06 CST 2025] XFS (vda1): EXPERIMENTAL big timestamp feature in use. Use at your own risk!
   [Tue Sep  9 14:21:06 CST 2025] XFS (vda1): EXPERIMENTAL inode btree counters feature in use. Use at your own risk!
   [Tue Sep  9 14:21:06 CST 2025] XFS (vda1): Mounting V5 Filesystem
   [Tue Sep  9 14:21:06 CST 2025] XFS (vda1): Ending clean mount
   [Tue Sep  9 14:26:41 CST 2025] XFS (vdb): EXPERIMENTAL big timestamp feature in use. Use at your own risk!
   [Tue Sep  9 14:26:41 CST 2025] XFS (vdb): EXPERIMENTAL inode btree counters feature in use. Use at your own risk!
   [Tue Sep  9 14:26:41 CST 2025] XFS (vdb): Mounting V5 Filesystem
   [Tue Sep  9 14:26:41 CST 2025] XFS (vdb): Ending clean mount
   [Thu Dec 25 17:58:24 CST 2025]   __xfs_filemap_fault+0x144/0x240 [xfs]
   [Tue Jul 21 15:11:01 CST 2026]  xfs_inodegc_flush.part.0+0x3e/0x90 [xfs]
   [Tue Jul 21 15:11:01 CST 2026]  xfs_fs_statfs+0x2d/0x1a0 [xfs]
   [Tue Jul 21 15:11:01 CST 2026]  xfs_inodegc_flush.part.0+0x3e/0x90 [xfs]
   [Tue Jul 21 15:11:01 CST 2026]  xfs_fs_statfs+0x2d/0x1a0 [xfs]
   [Tue Jul 21 15:11:01 CST 2026]  xfs_inodegc_flush.part.0+0x3e/0x90 [xfs]
   [Tue Jul 21 15:11:01 CST 2026]  xfs_fs_statfs+0x2d/0x1a0 [xfs]
   [Tue Jul 21 15:11:01 CST 2026]  ? xfs_buf_read_map+0x54/0x280 [xfs]
   [Tue Jul 21 15:11:01 CST 2026]  ? xfs_btree_read_buf_block.constprop.0+0x9d/0xe0 [xfs]
   ```

   没有任何 `Internal error`/`Corruption`/`shutdown` 记录，只有正常挂载信息和已经分析过的 hung_task 调用栈摘要，排除。

### 第 9 步：不甘心止步于"未解之谜"，换角度扩大搜索——挖到 AGI 也卡住了

不再依赖地址搜索，直接在 `foreach bt -f` 全量数据里搜"当前调用栈里还挂着 `xfs_alloc`/`xfs_trans`/`xfs_buf`/`xfs_bmap`/`xfs_difree`/`xfs_btree` 相关函数"的任务：

```
awk '/^PID:/{...} 函数名匹配 xfs_alloc|xfs_trans|xfs_buf|xfs_bmap|xfs_ialloc|xfs_difree|xfs_btree' foreach_bt_f.txt
```

挖到一个漏网的：**`HTTPHandler`（PID 3686288）**，调用链是 `openat()` 建新文件 -> `xfs_create -> xfs_dir_ialloc -> xfs_ialloc -> xfs_dialloc -> xfs_ialloc_read_agi -> xfs_read_agi -> xfs_trans_read_buf_map -> xfs_buf_read_map -> xfs_buf_get_map -> xfs_buf_find -> xfs_buf_lock -> down()`，同样卡死。

用第 5 步同一套反汇编方法，对 `HTTPHandler` 算出：

```
sem = ff380114e72ecaa0
bp（sem - 0x20）= ff380114e72eca80
```

```
kmem ff380114e72eca80
```

输出：

```
CACHE            OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE  NAME
ff38011c06022600      392     169198    215532   5987   16k  xfs_buf
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffe7f9c8049cbb00  ff380114e72ec000     0     36         36     0
  FREE / [ALLOCATED]
  [ff380114e72eca80]

      PAGE            PHYSICAL      MAPPING            INDEX CNT FLAGS
ffe7f9c8049cbb00     1272ec000     ff38011c06022600       0   1  17ffffc0010200 slab,head
```

```
struct xfs_buf.b_sema ff380114e72eca80
```

输出：

```
b_sema = {
  lock = {
    raw_lock = {
      {
        val = {
          counter = 0
        },
        {
          locked = 0 '\000',
          pending = 0 '\000'
        },
        {
          locked_pending = 0,
          tail = 0
        }
      }
    }
  },
  count = 0,
  wait_list = {
    next = 0xff7ad4f7cd9237c0,
    prev = 0xff7ad4f7cd9237c0
  }
},
```

```
struct xfs_buf.b_pag,b_bn,b_flags,b_hold ff380114e72eca80
```

输出：

```
b_pag = 0xff38011c00dd7000,
b_bn = 1572864002,
b_flags = 2097200,
b_hold = {
  counter = 4
},
```

结果：跟 AGF buffer 同一个 slab 页分配出来的；`b_pag` 完全相同（同一个 AG#1）；`b_bn = 1572864002`，跟 AGF 的 `b_bn = 1572864001` **正好相邻**——对应 XFS 一个 AG 最前面几个块的标准布局（块0=superblock，块1=AGF，块2=AGI），确认是 **AG#1 的 AGI buffer**；`b_flags`/`b_hold` 跟 AGF buffer 完全一样（同样 `XBF_DONE` 已完成但锁未释放）；`b_sema.wait_list` 同样只有 `HTTPHandler` 自己一个真实等待者。

### 第 10 步：换用 `b_transp` 直接找持锁事务，不再靠猜——真相大白

裸信号量不记录持有者，但 XFS 的 buffer 一旦被读进一个事务会一直标记着，`struct -o xfs_buf` 里 `[280] struct xfs_trans *b_transp` 这个字段能直接带我们找到持有它的事务：

```
struct xfs_buf.b_transp,b_log_item,b_pin_count ff380114e72ecc40
```

输出：

```
b_transp = 0xff38011cb6802bc8,
b_log_item = 0xff3801162b48c840,
b_pin_count = {
  counter = 0
},
```

AGF buffer 结果：`b_transp = 0xff38011cb6802bc8`（非空，确实挂在一个活跃事务上）。顺着事务往下挖：

```
struct -o xfs_trans
```

输出：

```
struct xfs_trans {
    [0] unsigned int t_magic;
    [4] unsigned int t_log_res;
    [8] unsigned int t_log_count;
   [12] unsigned int t_blk_res;
   [16] unsigned int t_blk_res_used;
   [20] unsigned int t_rtx_res;
   [24] unsigned int t_rtx_res_used;
   [28] unsigned int t_flags;
   [32] xfs_fsblock_t t_firstblock;
   [40] struct xlog_ticket *t_ticket;
   [48] struct xfs_mount *t_mountp;
   [56] struct xfs_dquot_acct *t_dqinfo;
   [64] int64_t t_icount_delta;
   [72] int64_t t_ifree_delta;
   [80] int64_t t_fdblocks_delta;
   [88] int64_t t_res_fdblocks_delta;
   [96] int64_t t_frextents_delta;
  [104] int64_t t_res_frextents_delta;
  [112] int64_t t_dblocks_delta;
  [120] int64_t t_agcount_delta;
  [128] int64_t t_imaxpct_delta;
  [136] int64_t t_rextsize_delta;
  [144] int64_t t_rbmblocks_delta;
  [152] int64_t t_rblocks_delta;
  [160] int64_t t_rextents_delta;
  [168] int64_t t_rextslog_delta;
  [176] struct list_head t_items;
  [192] struct list_head t_busy;
  [208] struct list_head t_dfops;
  [224] unsigned long t_pflags;
}
SIZE: 232
```

```
struct xfs_trans.t_ticket,t_mountp,t_flags,t_log_res,t_log_count ff38011cb6802bc8
```

输出：

```
t_ticket = 0xff38011e5c275b50,
t_mountp = 0xff38011c02e36000,
t_flags = 37,
t_log_res = 201976,
t_log_count = 8,
```

拿到 `t_ticket = 0xff38011e5c275b50`。再往下：

```
struct -o xlog_ticket
```

输出：

```
struct xlog_ticket {
    [0] struct list_head t_queue;
   [16] struct task_struct *t_task;
   [24] xlog_tid_t t_tid;
   [28] atomic_t t_ref;
   [32] int t_curr_res;
   [36] int t_unit_res;
   [40] char t_ocnt;
   [41] char t_cnt;
   [42] char t_clientid;
   [43] char t_flags;
   [44] uint t_res_num;
   [48] uint t_res_num_ophdrs;
   [52] uint t_res_arr_sum;
   [56] uint t_res_o_flow;
   [60] xlog_res_t t_res_arr[15];
}
SIZE: 184
```

```
struct xlog_ticket.t_task,t_tid,t_curr_res,t_unit_res ff38011e5c275b50
```

输出：

```
t_task = 0xff38011cd93eb180,
t_tid = 1468600922,
t_curr_res = 207220,
t_unit_res = 207220,
```

**结果：`t_task = 0xff38011cd93eb180`——这正是 `kworker/u34:3`（writeback）自己的 `task_struct` 地址！**

也就是说：**AGF buffer 的事务持有者是 `kworker/u34:3`，不是此前猜测的 `kworker/2:0`**——`kworker/u34:3` 在自己的 `xfs_bmap_btalloc` 里先用 AGF 成功分配过一次 extent（dmesg 里那几个带 `?` 号的旧帧 `xfs_alloc_vextent`/`xfs_alloc_ag_vextent` 其实是真实发生过、仍在生效的操作，不是纯垃圾残留），然后往下走到 `xfs_imap_to_bp` 找 inode cluster buffer 时自己也卡住了，事务不提交，AGF 就一直锁着。

反过来查 `kworker/u34:3` 自己当前卡住的 buffer（用同一套反汇编方法算出 `bp = ff380117743ac540`）：

```
struct xfs_buf.b_transp,b_bn,b_pag ff380117743ac540
```

输出：

```
b_transp = 0xff38011e7fa9e9f8,
b_bn = 1577999968,
b_pag = 0xff38011c00dd7000,
```

```
struct xfs_trans.t_ticket ff38011e7fa9e9f8
```

输出：

```
t_ticket = 0xff38011c8265a5c0,
```

```
struct xlog_ticket.t_task ff38011c8265a5c0
```

输出：

```
t_task = 0xff380115e4fe0000,
```

**结果：`t_task = 0xff380115e4fe0000`——这正是 `kworker/2:0` 自己的 `task_struct`！**

最后交叉验证：AGI buffer 的 `b_transp` 是不是也指向 `kworker/2:0` 的同一个事务：

```
struct xfs_buf.b_transp ff380114e72eca80
```

输出：

```
b_transp = 0xff38011e7fa9e9f8,
```

**结果：`0xff38011e7fa9e9f8`——跟 `kworker/u34:3` 卡住的那块 inode cluster buffer 的事务地址完全相同**，证实 AGI 和这块 inode cluster buffer确实是被 `kworker/2:0` 的**同一个事务**同时锁着。

### 最终结论：坐实的 ABBA 循环死锁

```
kworker/2:0（释放某个 inode）的事务：
    持有 AGI + 持有该 inode 所在的 inode cluster buffer
    -> 还需要 AGF（finobt 分裂）-> 卡住（AGF 被 kworker/u34:3 的事务攥着）

kworker/u34:3（回写另一个 inode，可能与上面共享同一个 cluster）的事务：
    持有 AGF（延迟分配转换时用它分配过 extent，事务未提交）
    -> 还需要 inode cluster buffer（写回该 inode）-> 卡住（被 kworker/2:0 的事务攥着）
```

`kworker/2:0` 持有 A（AGI + inode cluster）要 B（AGF），`kworker/u34:3` 持有 B（AGF）要 A（inode cluster）——**这是一个可以用 `b_transp -> t_ticket -> t_task` 内核自身维护的关联关系完整验证的 ABBA 循环死锁，不是等磁盘慢 I/O**。这直接推翻了正文「根因分析」「结论」里"未见 ABBA 死锁证据，更符合等 I/O 完成"的判断，那两节需要连同后面"这批补丁能不能彻底解决 D 状态问题"的评估一起重写。

## 上游 `82842fee6e597` 是怎么修的

看一下这个修复的具体代码改动，理解它为什么能打破 AGI/AGF/inode cluster buffer 之间的锁序倒挂。

### 修复前：`xfs_trans_log_inode()` 会在事务执行过程中随时锁 inode cluster buffer

`xfs_trans_log_inode()` 是内核里"标记这个 inode 在当前事务里被改过、需要落盘"的函数，调用点散落在整个 XFS 代码里非常多的地方——只要事务里改了 inode 的任何字段（大小、extent、时间戳……），代码里随手就会调一次。在修复之前，这个函数**做的事情很重**，其中最关键的一段：

```c
// fs/xfs/libxfs/xfs_trans_inode.c（修复前）
if (!iip->ili_item.li_buf) {
        struct xfs_buf  *bp;
        ...
        error = xfs_imap_to_bp(ip->i_mount, tp, &ip->i_imap, &bp);
        ...
        xfs_buf_hold(bp);
        iip->ili_item.li_buf = bp;
        bp->b_flags |= _XBF_INODES;
        list_add_tail(&iip->ili_item.li_bio_list, &bp->b_li_list);
        xfs_trans_brelse(tp, bp);
}
```

也就是说：**只要这是这个事务里第一次 log 这个 inode，就会立刻去读并"钉住"（pin）这个 inode 所在的 inode cluster buffer**，好让它在 inode 变脏期间不会被提前写回。问题是这段代码**在哪个时间点执行、当时已经持有了哪些别的锁，完全取决于调用它的代码路径**，没有统一安排：

- `unlink`/`ifree` 这条路径（本文 `kworker/2:0` 卡住的那条）：先锁 **AGI**（处理 unlinked inode 链表），链表操作本身要锁 **inode cluster buffer**，然后如果还要做 finobt/inobt 的 btree 维护，才会去锁 **AGF**——顺序是 `AGI -> inode cluster buffer -> AGF`。
- 延迟分配转换这条路径（本文 `kworker/u34:3` 卡住的那条）：先分配 extent 要锁 **AGF**，分配完之后调 `xfs_trans_log_inode()` 把这次分配结果记到 inode 上，这一步才第一次去锁 **inode cluster buffer**——顺序是 `AGF -> inode cluster buffer`。

两条路径对 AGF 和 inode cluster buffer 的加锁顺序**正好相反**，而 inode cluster buffer 是好几个相邻 inode 共用的（一个 cluster 通常装 32 个 inode），只要这两条路径分别操作的 inode 恰好在同一个 cluster 里，就会撞上 ABBA 死锁——这正是本文 `kworker/2:0` 与 `kworker/u34:3` 之间实际发生的情况。

### 修复思路：把"锁 inode cluster buffer"这件事，挪到事务提交前的固定时机去做

直接去改遍布全代码的 `xfs_trans_log_inode()` 调用点、逐个理清楚加锁顺序，工作量太大也太容易漏。Dave Chinner 的做法是利用 XFS 事务提交流程里已有的一个钩子：`->iop_precommit`——这个回调**保证**在事务的所有修改都做完、包括 AGI、AGF 这些"大件"资源都已经锁好之后，提交前最后统一调用一次。

具体改动（`fs/xfs/xfs_inode_item.c` 新增 `xfs_inode_item_precommit()`，`fs/xfs/libxfs/xfs_trans_inode.c` 精简 `xfs_trans_log_inode()`）：

```c
// 修复后，xfs_trans_log_inode() 瘦身成只做轻量级记账，不碰任何锁
void
xfs_trans_log_inode(...)
{
        ...
        tp->t_flags |= XFS_TRANS_DIRTY;
        ...
        iip->ili_dirty_flags |= flags;   // 只记录"哪些字段脏了"，仅此而已
}
```

```c
// 修复后，真正读锁 inode cluster buffer 的逻辑挪到这里，
// 由事务提交流程在最后统一调用
static int
xfs_inode_item_precommit(struct xfs_trans *tp, struct xfs_log_item *lip)
{
        ...
        if (!iip->ili_item.li_buf) {
                error = xfs_imap_to_bp(ip->i_mount, tp, &ip->i_imap, &bp);
                ...
                iip->ili_item.li_buf = bp;
                ...
        }
        ...
}

static const struct xfs_item_ops xfs_inode_item_ops = {
        .iop_sort       = xfs_inode_item_sort,       // 新增：按 inode number 排序
        .iop_precommit  = xfs_inode_item_precommit,  // 新增：提交前统一处理
        ...
};
```

这样一来，不管 `xfs_trans_log_inode()` 在事务执行过程中被调用了多少次、在代码里的什么位置被调用，**真正去锁 inode cluster buffer 的动作，永远被推迟到事务提交前的最后一刻**——那时候 AGI、AGF 这些锁早就已经按 `AGI -> AGF` 的既定顺序锁好了，不会再出现"先锁了 AGF，事务中途才第一次去抢 inode cluster buffer 锁"这种情况。锁序被强制统一成 `AGI -> AGF -> inode cluster buffer`。

另外新增的 `xfs_inode_item_sort()`（按 `i_ino` 排序）解决的是一个相关的次生问题：如果同一个事务里改了不止一个 inode（因而涉及不止一个 cluster buffer），`->iop_precommit` 会对事务里所有脏的 inode log item 挨个调用——排序保证了这种"一个事务修改多个 inode"的场景下，所有事务锁 cluster buffer 的顺序也是一致的（按 inode 号从小到大），避免多 inode 之间再出现另一种 ABBA。

### 为什么这个修复对本文这次故障是对症下药的

本文 vmcore 复核确认的死锁，正是 `kworker/2:0`（`AGI -> inode cluster buffer -> AGF`）与 `kworker/u34:3`（`AGF -> inode cluster buffer`）之间的锁序倒挂，跟这个 commit message 描述的场景一字不差。合入这个修复之后，`kworker/u34:3` 那条延迟分配转换路径不会再在"已经锁了 AGF"的情况下现场去锁 inode cluster buffer——它对 inode 的修改只会被记到 `ili_dirty_flags` 里，真正锁 cluster buffer 的动作要等到事务提交前，此时锁序已经统一，不会再跟 `kworker/2:0` 的 `AGI -> inode cluster buffer -> AGF` 顺序产生倒挂，死锁窗口从代码层面被消除。

## 结论（已按 vmcore 复核结果更新）

1. 触发点**不是**磁盘慢 I/O，而是 XFS 内核代码里一个真实的 **ABBA 循环死锁**（详见「vmcore 复核」一节的完整验证过程）：

   ```
   kworker/2:0（xfs-inodegc，释放某个 inode）的事务：
       持有 AGI + 持有该 inode 所在的 inode cluster buffer
       -> 还需要 AGF（finobt 分裂）-> 卡住

   kworker/u34:3（writeback，回写另一个 inode）的事务：
       持有 AGF（延迟分配转换时分配过 extent，事务未提交）
       -> 还需要 inode cluster buffer（写回该 inode）-> 卡住
   ```

   两个内核线程互相持有对方需要的资源，谁都无法继续，形成循环等待——这是 XFS 代码层面的问题，跟这台主机的 `vdb` 磁盘本身是否健康**无关**（虽然不能完全排除某次慢 I/O 是触发这次死锁的诱因，但死锁一旦形成就会一直卡死，不会随磁盘恢复而自愈）。
2. `xfs-inodegc` 与 `writeback` 两个 worker 卡在各自的 `xfs_buf_lock` 上，看似是在等对应 buffer 的 I/O 完成（两块 buffer 的 `XBF_DONE` 标志确实都已置位，I/O 早就做完了），实际是在等**对方持有的锁**，是循环等待，不是等 I/O。
3. 监控/agent 进程大量 D 状态是 `xfs_fs_statfs -> xfs_inodegc_flush -> flush_work` 这条路径的放大效应，但这次**不会自愈**——因为根因是死锁而不是慢 I/O，只要死锁不解除，inodegc worker 就永远不会恢复，所有调用 statfs 的进程也会一直卡住，必须靠重启（或杀掉/绕过其中一个持锁事务）才能解除。
4. `HTTPHandler` 等建新文件的用户态进程，是被 `kworker/2:0` 持有的 AGI 连带拖下水的**次生受害者**，不是独立的第三个根因。

**局限性说明**：最初分析只拿到了全量 dmesg，没有 vmcore，dmesg 里只能看到"卡在 `xfs_buf_lock` 的 `down()` 上"，看不到这块 buffer 当时的持锁者是谁、在做什么，因此最初误判为"更像是等 I/O，未见死锁证据"。后续在 host 上补拍了一份该 VM 的 vmcore，用 `crash 8.0.2` 做了进一步核实，详见前面「vmcore 复核」一节——**这次确认了是真实的 ABBA 死锁**，纠正了最初的判断。

## inodegc 特性背景：它是干什么的

`inodegc`（inode garbage collection）是 XFS 在 **v5.15-rc1** 引入的优化（`ab23a7768739`，"xfs: per-cpu deferred inode inactivation queues"），本次故障里 `xfs-inodegc/vdb` worker 就是这个特性的产物。

### 要解决的问题

一个文件被 `unlink`、引用计数归零后，内核会走 VFS 的 `->evict_inode` -> XFS 的 `xfs_inactive()` 去做"清理"：

- 如果文件有很多 extent（比如被打散得很碎的大文件），要把这些 extent 全部释放回 AG 的空闲块，可能触发多轮事务
- 释放 inode 本身（`xfs_difree`），可能触发 inobt/finobt 的 btree 分裂或合并（就是本次日志里 `kworker/2:0` 卡住的那条路径 `xfs_inactive_ifree -> xfs_difree -> __xfs_btree_split`）

在没有 inodegc 之前，这些工作是**同步**跑在触发 `unlink()`、或者最后一次 `iput()` 释放引用的那个进程/CPU 上下文里的。文件 extent 数特别多、或者盘比较慢时，一次 `rm 大文件` 可能会在原地卡住很久。

### inodegc 的做法

commit 原文：

> Move inode inactivation to background work contexts so that it no longer runs in the context that releases the final reference to an inode.

具体实现：

- 每个 CPU 一个无锁链表（`llist`）队列 + 一个 workqueue work item：inode 变成 inactive 后不再原地处理，而是丢进当前 CPU 的队列
- 典型场景：`unlink` 一个目录下一堆大文件时，进程可以立刻处理下一个 inode，不用等上一个 inode 的 extent 全部释放完；真正的释放工作交给后台的 `xfs-inodegc/<dev>` 内核线程异步做
- 队列深度 32（对应一个 inode cluster buffer 里的 inode 数），攒够一批或超过阈值（256 个）就触发 worker 处理，以保持 CPU cache 局部性

一句话：**inodegc 把"删文件时清理 inode 元数据"从"谁删的谁等着"改成了"扔给后台线程异步做"，本意是避免大量 unlink/reflink 场景下用户态被阻塞。**

### 和本次故障的关系

正因为 inactivation 被挪到了后台的 `xfs-inodegc/vdb` worker 上，这个 worker 才会单独出现在 hung_task 列表里——它卡在 `xfs_buf_lock` 上本身是 inodegc 正常工作流程的一部分，不是 bug。

但特性上线后暴露出一个副作用：既然 inactivation 变成异步了，`xfs_fs_statfs()` 想拿到"准确"的空闲块/inode 数，就得先等这个后台队列跑完（`xfs_inodegc_flush`）——这一等，如果后台 worker 因为慢盘卡住，所有调用 `statfs` 的进程就全被拖下水了。这正是本次故障放大成一大片 D 状态进程的根本机制，也是下面 `xfs_inodegc_push()` 补丁要修的问题。

## 内核层面的修复：xfs_inodegc_push()

`statfs -> xfs_inodegc_flush -> flush_work` 这条阻塞路径本身，上游已经在 **v5.19-rc5** 修掉了：

```
commit 5e672cd69f0a534a445df4372141fd0d1d00901d
Author: Dave Chinner <dchinner@redhat.com>
Date:   Thu Jun 16 07:44:32 2022 -0700

    xfs: introduce xfs_inodegc_push()

    The current blocking mechanism for pushing the inodegc queue out to
    disk can result in systems becoming unusable when there is a long
    running inodegc operation. This is because the statfs()
    implementation currently issues a blocking flush of the inodegc
    queue and a significant number of common system utilities will call
    statfs() to discover something about the underlying filesystem.

    This can result in userspace operations getting stuck on inodegc
    progress, and when trying to remove a heavily reflinked file on slow
    storage with a full journal, this can result in delays measuring in
    hours.

    Avoid this problem by adding "push" function that expedites the
    flushing of the inodegc queue, but doesn't wait for it to complete.

    Convert xfs_fs_statfs() and xfs_qm_scall_getquota() to use this
    mechanism so they don't block but still ensure that queued
    operations are expedited.

    Fixes: ab23a7768739 ("xfs: per-cpu deferred inode inactivation queues")
    Reported-by: Chris Dunlop <chris@onthe.net.au>
    Signed-off-by: Dave Chinner <dchinner@redhat.com>
    [djwong: fix _getquota_next to use _inodegc_push too]
    Reviewed-by: Darrick J. Wong <djwong@kernel.org>
    Signed-off-by: Darrick J. Wong <djwong@kernel.org>
```

Dave Chinner 描述的场景与本次故障几乎一致：大量调用 `statfs()` 的用户态工具因为 `xfs_fs_statfs` 阻塞式等待 inodegc 而被一起拖死。补丁把 `xfs_fs_statfs()`（以及 `xfs_qm_scall_getquota()`）改成调用不阻塞的 `xfs_inodegc_push(mp)`——只是把 inodegc 队列"催一下"尽快跑，但 `statfs` 本身不再 `flush_work` 等它跑完：

```c
// fs/xfs/xfs_super.c（当前 upstream 主线）
STATIC int
xfs_fs_statfs(
	struct dentry		*dentry,
	struct kstatfs		*st)
{
	struct xfs_mount	*mp = XFS_M(dentry->d_sb);
	struct xfs_inode	*ip = XFS_I(d_inode(dentry));

	/*
	 * Expedite background inodegc but don't wait. We do not want to block
	 * here waiting hours for a billion extent file to be truncated.
	 */
	xfs_inodegc_push(mp);
	...
```

代价是返回的空闲块/inode 计数可能有极短暂的不精确（因为不再等最新一批 inactivation 落盘），换来的是不会被无关的慢 I/O 拖死一大片进程。

`ab23a7768739`（per-cpu deferred inode inactivation queues，即 inodegc 特性本身）是 **v5.15-rc1** 合入主线的特性，`5e672cd69f0a534a445df4372141fd0d1d00901d` 这个修复则是 **v5.19-rc5** 才合入。故障主机跑的是 `5.10.0-136.12.0.92.ctl3` —— 说明厂商把 inodegc 特性从 5.15 背移植（backport）到了 5.10，但没有一并带上晚于 inodegc 四个大版本才合入的这个修复，属于"只背新特性、漏背配套修复"的典型问题。

在 `ctkernel-lts-5.10-develop` 仓库里核实了这条 inodegc 特性引入与修复的现状：

- inodegc 特性由 **`705fccfbcf0b8` "xfs: per-cpu deferred inode inactivation queues"**（2022-01-07，走的是 openEuler 的 `mainline-inclusion from mainline-v5.14-rc4` 流程，Signed-off-by 到 Huawei 的 Lihong Kou/Zhang Yi/Zheng Zengkai）引入，是 `heads/ctkernel-lts-5.10/develop` 分支的祖先提交，确认已经在线上。
- 对应的修复提交在**当前 develop 分支上还没有合入**：`git merge-base --is-ancestor` 对 upstream 原始 hash `5e672cd69f0a534a` 以及各个厂商 cherry-pick 变体（`99e65f075e6cf`/`cc10c7d993f28`/`0c38d6918523a`/`bef4b481178cb`/`a435f02700ae6`）逐一核对，全部返回 `NOT ancestor`，说明这台故障主机（乃至当前 develop 分支上所有环境）目前都还暴露在这个问题下。
- 好消息是**修复已经有人准备好了**：仓库里已经存在一个未合并的远程分支 `remotes/ctkernel-lts-5.10/lchen/xfs-statfs`，比对 `heads/ctkernel-lts-5.10/develop` 领先两个提交：

  ```
  26b5a22d7a83f xfs: flush inodegc workqueue tasks before cancel
  a435f02700ae6 xfs: introduce xfs_inodegc_push()
  ```

  其中 `a435f02700ae6` 就是本节要的那个修复（openEuler/Huawei 背移植版本，commit message 自称 `mainline inclusion from mainline-v5.19-rc2`，Signed-off-by Guo Xuenan/Yang Erkun/Jialin Zhang），提交时间 2025-08-27，比对的 merge-base（`53b37ccd0bd7`）比当前 develop 的 HEAD（`a8d377cf6862`，2025-11-20 "release: develop 0096"）要早，也就是这个分支落后了 develop 一段时间、尚未 rebase/合并。**注意**：openEuler 这条 `mainline-v5.19-rc2` 标注不准确——用 `git describe`/`merge-base` 对上游原始提交 `5e672cd69f0a53` 核实过，实际落地版本是 **v5.19-rc5**；下面对比表和 agi-bucket 那条修复都遇到了同样的问题（vendor 的 "from mainline-vX.Y" 标注对应的是背移植时参照的上游开发中状态，不一定是最终真实合入版本），已按实测结果统一订正。

**注意**：`a8d377cf6862`（"release: develop 0096"）是 `heads/ctkernel-lts-5.10/develop` 当前已经发布的最新版本，用 `git merge-base --is-ancestor a435f02700ae6 heads/ctkernel-lts-5.10/develop` 核实过，结果是 **0096 里也没有这个修复**——不是"要等到 0096 才修"，而是**0096 目前也是缺这个修复的**。截至目前，任何一个已发布的 develop 版本都不包含它，修复只静静躺在落后主线的 `lchen/xfs-statfs` 分支上。要让它生效，需要把这个分支 rebase 到最新 develop 后合入，落到**下一个**版本（0097 或之后的某个版本），而不是已发布的 0096。

**补充说明**：`xfs_inodegc_push()` 解决的只是"慢盘/长耗时 inodegc 不该拖死无关的 `statfs` 调用方"这个放大效应，不是万能药——如果 `xfs-inodegc` worker 卡住的根因是 buffer 锁上真正的 ABBA 死锁（而不是本次这种"等 I/O 完成"），合入这个补丁之后 inodegc worker 本身该卡还是卡，只是不会再连带拖死一大片调 `statfs` 的监控进程。本次故障没有 vmcore，无法 100% 排除前者。

## inodegc 相关修复全景对比：CTyunOS vs openEuler

以 inodegc 特性（`ab23a7768739`）为起点，upstream 目前一共有 6 个与之相关的后续修复。逐一核对 CTyunOS `ctkernel-lts-5.10/develop` 与 openEuler `OLK-5.10` 分支的合入情况：

| upstream 提交 | 版本 | CTyunOS `ctkernel-lts-5.10/develop` | CTyunOS 合入者（含合入版本） | openEuler `OLK-5.10` | openEuler 合入者 |
|---|---|---|---|---|---|
| `ab23a7768739` per-cpu deferred inode inactivation queues（特性本身） | v5.15-rc1 | ✅ `705fccfbcf0b8` | **release-0090 起含**（2025-03-08；直接复用 openEuler 原始提交对象，未额外背移植），committer **Zheng Zengkai**（华为），原始背移植 SoB **Lihong Kou**（华为） | ✅ 同 `705fccfbcf0b8` | SoB **Lihong Kou**，Reviewed-by **Zhang Yi**，committer **Zheng Zengkai**（均华为） |
| `6191cf3ad59fd` flush inodegc workqueue tasks before cancel | v5.17-rc1 | ❌ 变体 `26b5a22d7a83f` 躺在未合入的 `lchen/xfs-statfs` 分支 | **任何已发布 release 版本都不含**（含最新 release-0096）。committer **Li Chen**（`chenl311`），内容照搬 openEuler 的 `I6WKVJ` 背移植版本（SoB **Guo Xuenan**/Reviewed-by **Yang Erkun**/SoB **Jialin Zhang**，均华为），2025-08-27 搬入 `lchen/xfs-statfs` 分支，至今未合入 develop | ✅ `ad0a103e90651` | SoB **Guo Xuenan**，Reviewed-by **Yang Erkun**，committer **Jialin Zhang**（均华为），2023-04-26 |
| `7cf2b0f9611b9` bound maximum wait time for inodegc work | v5.19-rc5 | ❌ 任何分支都没有，连准备都没准备 | **任何已发布 release 版本都不含**，任何分支都没有——连搬都没搬 | ✅ `cfb1750859363` | SoB **Guo Xuenan**，Reviewed-by **Yang Erkun**，committer **Jialin Zhang**，2023-04-26（和下面 push/cancel-flush 是同一批背移植） |
| `5e672cd69f0a53` **introduce xfs_inodegc_push()** | v5.19-rc5（`7cf2b0f9611b9` 的直接子提交，同批次紧随其后） | ❌ 变体 `a435f02700ae6` 同样躺在 `lchen/xfs-statfs`，未合入 | **任何已发布 release 版本都不含**（含最新 release-0096）。committer **Li Chen**，同样是照搬 openEuler `I6WKVJ` 版本，2025-08-27 搬入但未合入 develop | ✅ `cc10c7d993f28` | SoB **Guo Xuenan**，Reviewed-by **Yang Erkun**，committer **Jialin Zhang**，2023-04-26 |
| `04a98a036cf8` flush inode gc workqueue before clearing agi bucket | v6.0-rc1（注：openEuler 背移植 commit message 里自称 "mainline inclusion from mainline-v5.19-rc2"，但用 `git merge-base --is-ancestor` 对 v5.18~v5.19 全部 rc 及正式版逐一核实，均为 NO，实际到 v6.0-rc1 才是祖先——这是背移植准备时参照的上游开发中状态，不是真实落地版本，已按实测版本更正并调整了在本表中的顺序） | ✅ `ef4894f06a4a4` | **release-0090 起含**（与特性本身一起进的，同样直接复用 openEuler 原始提交对象），committer **Zheng Zengkai** | ✅ 同 `ef4894f06a4a4` | Author **Zhang Yi**，SoB **Guo Xuenan**，Reviewed-by **Zhang Yi**，committer **Zheng Zengkai**（均华为） |
| `62334fab47621` use per-mount cpumask to track nonempty percpu inodegc lists | v6.6-rc3 | ❌ 任何分支都没有 | **任何已发布 release 版本都不含**，`lchen/xfs-statfs` 等任何分支都没有——连搬都没搬。用 `git patch-id` 对动过 `xfs_icache.c`/`xfs_mount.h`/`xfs_mount.c`/`xfs_super.c` 的提交做过内容级扫描，排除了换措辞的重写版本 | ❌ 任何分支都没有 | — |
| `2d873efd174ba` flush inodegc before swapon | v6.14-rc4 | ✅ `6c588c827e2ef` | **release-0094 起含**（2025-07-29；release-0092/2025-04-29 时还不含）。SoB **Li Chen**，Reviewed-by/committer **Bin Lai**（`laib2`），2025-07-16 合入，内部单号 `CTKfeat: #84056`——这个是 CTyunOS **自己单独背移植**的（openEuler 没有这个提交，搬不了） | ❌ 没有 | — |

结论：openEuler 合了 4 个修复只缺 swapon 那个，且这 4 个修复是华为团队（Guo Xuenan/Yang Erkun/Jialin Zhang/Zhang Yi/Zheng Zengkai）在 2023-04-26 一次性批量背移植进去的；CTyunOS 只合了 2 个（agi-bucket 那个 + swapon 那个），前两个（特性本身 + agi-bucket）是直接复用 openEuler 的原始提交对象（连 hash 都没变），没有 CTyunOS 自己的背移植记录，只有 swapon 这一个是 Li Chen/Bin Lai 在 2025-07-16 独立背移植的。本次案例真正相关的 `xfs_inodegc_push`、`cancel 前 flush`、`bound wait time` 三个 CTyunOS 全都没合——其中 push、cancel-flush 已经由 Li Chen 在 2025-08-27 把 openEuler 现成的背移植版本（`I6WKVJ`）搬进了 `lchen/xfs-statfs` 分支，只是没合到 develop；bound-wait-time 目前连搬都没搬。此外 `62334fab47621`（use per-mount cpumask to track nonempty percpu inodegc lists，v6.6-rc3，修复一个 inodegc flush 相关的理论 UAF）是双方都完全没背、都没准备的一个修复，属于新发现的缺口。

**背移植顺序倒挂的问题**：CTyunOS 已合入的 `6c588c827e2ef`（flush inodegc before swapon）commit message 里原样带着 `Fixes: 5e672cd69f0a53 ("xfs: introduce xfs_inodegc_push()")`，也就是说这个 patch 本身是在修复 `xfs_inodegc_push()` 引入之后才会出现的一个竞态（swapon 场景）。但 `xfs_inodegc_push()` 本身（`a435f02700ae6`）在 CTyunOS 里至今没有合入——相当于合了"修复 A 引入的问题"的补丁，却没合 A 本身，背移植顺序是倒着来的。

`6c588c827e2ef` 的合入信息：

- **合入版本**：`release-0092`（2025-04-29）不包含，`release-0094.rc1`（2025-07-29）开始包含，此后一直延续到当前最新的 `release-0096`——即从 **0094** 版本起合入。
- **合入时间**：2025-07-16
- **背移植者**（Signed-off-by）：Li Chen `<chenl311>`
- **Review 者**（Reviewed-by，同时也是 committer）：Bin Lai `<laib2>`
- **内部特性单号**：`CTKfeat: #84056`
- 上游作者链完整保留：Christoph Hellwig（author）→ Darrick J. Wong / Dave Chinner（review）→ Carlos Maiolino（xfs maintainer 合入 upstream）

## 最终待合入清单：CTyunOS vs openEuler（含真正修复本次故障的提交）

把前面「inodegc 相关修复全景对比」表里梳理出的 6 个修复，加上本次 vmcore 复核确认的 AGF vs inode cluster buffer 死锁修复，按**上游合入版本顺序**合并成一张最终的待办清单。格式沿用前面的对比表，额外加一列**明确标出哪一条才是这次故障的真正根因修复**——除了这一条，其余都只是缓解"放大效应"或修复 inodegc 特性的其他子问题，**都不能解决这次真正卡死的 ABBA 死锁**。

| upstream 提交 | 版本 | CTyunOS `ctkernel-lts-5.10/develop` | openEuler `OLK-5.10` | 是否修复本次故障根因 |
|---|---|---|---|---|
| `ab23a7768739` per-cpu deferred inode inactivation queues（特性本身） | v5.15-rc1 | ✅ `705fccfbcf0b8`（release-0090 起含） | ✅ 同 `705fccfbcf0b8` | ❌ 否——这是 inodegc 特性本身，不是修复 |
| `6191cf3ad59fd` flush inodegc workqueue tasks before cancel | v5.17-rc1 | ❌ 变体躺在未合入的 `lchen/xfs-statfs` 分支 | ✅ `ad0a103e90651` | ❌ 否——只影响 unmount/cancel 场景 |
| `7cf2b0f9611b9` bound maximum wait time for inodegc work | v5.19-rc5 | ❌ 任何分支都没有 | ✅ `cfb1750859363` | ❌ 否——只是限制 inodegc 最长等待时间 |
| `5e672cd69f0a53` introduce xfs_inodegc_push() | v5.19-rc5 | ❌ 变体躺在未合入的 `lchen/xfs-statfs` 分支 | ✅ `cc10c7d993f28` | ❌ 否——只消除 statfs 阻塞放大效应，本文重点分析对象 |
| `04a98a036cf8` flush inode gc workqueue before clearing agi bucket | v6.0-rc1 | ✅ `ef4894f06a4a4`（release-0090 起含） | ✅ 同 `ef4894f06a4a4` | ❌ 否——修的是日志恢复阶段的另一个问题 |
| **`82842fee6e5979ca7e2bf4d839ef890c22ffb7aa` xfs: fix AGF vs inode cluster buffer deadlock** | **v6.4-rc6** | **❌ 任何分支都没有** | **❌ 任何分支都没有（含 OLK-6.6 之前的所有 5.10 相关分支）** | **✅ 是——这才是这次 `kworker/2:0` 与 `kworker/u34:3` 之间 ABBA 死锁的真正修复** |
| `62334fab47621` use per-mount cpumask to track nonempty percpu inodegc lists | v6.6-rc3 | ❌ 任何分支都没有 | ❌ 任何分支都没有 | ❌ 否——修的是 CPU 热插拔场景的理论 UAF |
| `2d873efd174ba` flush inodegc before swapon | v6.14-rc4 | ✅ `6c588c827e2ef`（release-0094 起含） | ❌ 没有 | ❌ 否——只影响 swapon 场景 |

**结论**：8 行里除了特性本身，其余 6 个修复都跟本次故障的死锁根因无关，只是同一个 inodegc 子系统里各自独立的问题（放大效应、UAF、swapon 竞态等），合了也解决不了这次故障会不会复发；真正需要合入以修复本次故障根因的，是 **`82842fee6e597`**（v6.4-rc6）——**CTyunOS 和 openEuler 目前都没有背移植这个修复**，这是本次排查最终应该落地为背移植需求单的那一条。
## 后续排查建议

1. ~~对照该主机同一时间窗口（15:09–15:13）的存储后端/hypervisor 侧监控与日志，确认 `vdb` 是否存在慢 I/O、超时重传或宿主机侧异常。~~ **已由 vmcore 复核推翻**：根因不是存储侧慢 I/O，是 XFS 内核代码里的 ABBA 死锁，这条排查方向可以停止，不用再往存储侧查了。
2. ~~检查 `/proc/diskstats` 或历史 `iostat -x` 采样数据，确认 `vdb` 在故障窗口内的 `await`/`svctm` 是否有异常尖峰。~~ 同上，已确认与磁盘本身健康状况无关，不用再查。
3. ~~确认故障是否随宿主机 I/O 恢复而自愈……若后续能拿到 vmcore，建议用 crash 工具进一步确认持锁上下文，排除真正死锁的可能。~~ **已完成**：vmcore 已拿到并复核过，用 `crash` + `b_transp -> t_ticket -> t_task` 确认了是 `kworker/2:0`（持有 AGI + inode cluster buffer）与 `kworker/u34:3`（持有 AGF）之间的真实 ABBA 循环死锁，详见「vmcore 复核」一节，不会自愈，必须靠重启解除。
4. **不需要重新 cherry-pick，直接跟 `remotes/ctkernel-lts-5.10/lchen/xfs-statfs` 这个已有分支的作者对齐、把它 rebase 到最新 develop 后合入即可**：分支上的两个提交 `26b5a22d7a83f`（flush inodegc workqueue tasks before cancel）+ `a435f02700ae6`（introduce xfs_inodegc_push）已经是背移植好的版本，只是落后 develop 一段时间没跟上。**注意**：这两个补丁能缓解放大效应（减少被连带拖死的无关进程数量），但**不能修复这次发现的 ABBA 死锁本身**，二者需要分开立项跟踪。
5. 额外建议补齐 `7cf2b0f9611b9`（bound maximum wait time for inodegc work，v5.19-rc5）：这个修复目前在 CTyunOS 任何分支里都不存在，openEuler `OLK-5.10` 已有对应版本 `cfb1750859363`，可以直接参考背移植。
6. 再额外建议补齐 `62334fab47621`（use per-mount cpumask to track nonempty percpu inodegc lists，v6.6-rc3）：这个修复 openEuler 也没有，没有现成版本可抄，需要直接从 upstream cherry-pick，建议和上面几个一起统一排期背移植。
7. **新增（vmcore 复核后追加，已查明）**：这次确认的 `kworker/2:0`（AGI + inode cluster buffer）与 `kworker/u34:3`（AGF）之间的 ABBA 死锁，**是一个上游早已确认、有名有姓的已知 bug，不是本次新发现的未知问题**：

   - **引入 bug 的提交**：`298f7bec503f`("xfs: pin inode backing buffer to the inode log item")，**v5.9-rc1** 合入（2020-07-07），比 5.10 还早——CTyunOS `ctkernel-lts-5.10/develop` 和 openEuler `OLK-5.10` **都确认包含**这个提交，也就是说两家从 5.10 发布那天起就带着这个 bug。
   - **修复提交**：`82842fee6e5979ca7e2bf4d839ef890c22ffb7aa`("xfs: fix AGF vs inode cluster buffer deadlock"，作者 Dave Chinner)，**v6.4-rc6** 才合入（2023-06-05），比引入 bug 的提交晚了将近 3 年。commit message 描述的锁序倒挂场景（"AGI -> inode cluster buffer -> AGF" vs "AGF -> inode cluster buffer"）跟这次 vmcore 挖出来的两个 worker 的加锁顺序**完全一致**。修复方式是把 `xfs_trans_log_inode()` 里锁 inode cluster buffer 的部分挪到事务提交前的 `->iop_precommit` 阶段，避免过早抢占该锁。
   - 用 `git merge-base --is-ancestor` 核实：这个修复提交在 CTyunOS `ctkernel-lts-5.10/develop` 和 openEuler `OLK-5.10` 里**均不存在**——两家目前都还暴露在这个死锁下。
   - 该修复后来也进了 stable 树（`65fc94fc8774146ab96d7c20f138cfab1a4db6f2`，backport 进 **v6.1.112**，2024-09-30，Leah Rumancik 背的），说明上游认为这个问题足够严重、值得单独 backport 到 stable 分支，不是可以忽略的边角案例。
   - **下一步建议**：把 `82842fee6e597` 这个修复（或对应的 stable 变体）背移植到 CTyunOS 5.10 分支，这是解决这次死锁根因的直接手段，跟 `xfs_inodegc_push` 系列补丁是完全独立的两件事，需要分开立项、分开评审。
   - 死锁一旦形成无法自愈，建议给运维侧一个应急预案：出现类似"大量业务进程 D 状态 + hung_task 告警"且怀疑是这一类死锁时，除了走 `statfs` 放大链路的常规排查，也要考虑直接重启实例，不必浪费时间等待存储侧自愈。
8. 这批 `xfs_inodegc_push` 系列补丁解决的是"故障拖死无关进程"这个放大效应，**不能让死锁本身消失**——死锁的根因排查见第 7 条，跟这批补丁是两件事，评估收益时不要混为一谈。

## 附：同一处代码在另一台主机（长沙42-os）上的独立复现

**注意：这是另一个资源池、另一台主机、另一次独立故障，不是本文前面分析的那次故障，两者互不相关，只是命中了同一处内核代码。** 对应日志文件：`messages-20260724`。

### 故障通报原文

```
【故障名称】2026年7月21日湖南高速信息科技有限公司（企业级）ck数据库无法连接故障进度通报
【故障现象】ck数据库无法连接
【业务影响】路网应急系统（用户反馈市长在现场等待演示）
【客服系统资源池】长沙42-os
【故障原因】待定
【故障发生时间】2026-07-21 08:20:00
【故障升级时间】2026-07-21 08:39:00
【故障恢复时间】2026-07-21 09:37:00
【影响实例信息】
实例信息：43.65.3.106
3d305b5e9e95474f9572491a73e78fca
主机uuid：ae25d315-e9e3-4800-3069-1b9aff4a5d04

#故障进展
08:39 已拉通数据库指挥长上会沟通
09:00 数据库反馈当前底层卡顿，已拉通存储运维、计算运维入会核实
09:05 核实存储底层磁盘时延、挂载、IO正常，但数据库底层无法执行命令，继续排查中
09:15 现测试存储节点到宿主机节点存在丢包，已拉通物理网运维查看，同时正在核实是否与昨晚变更相关
09:30 客户反馈只涉及该问题实例异常，其余实例正常，准备重启实例尝试快恢
09:37 重启实例虚机完成，业务反馈已恢复
10:50 现业务重启后已恢复，观测暂无异常，已收集相关信息反馈线下群内进行分析，会议线上暂时闭环
```

### 日志与故障通报的对照

`messages-20260724` 里的 `xfs-inodegc/vdb xfs_inodegc_worker` 卡在 `xfs_buf_lock -> down()` 这条调用栈，和本文正文分析的那次故障（不同主机）代码路径完全一致，参见前面「相同点」的调用栈对比。

但结合故障通报的时间线重新看这份日志，之前"卡了一下就自愈了"的判断需要修正：

- hung_task 告警一共只出现了 **10 条**（07:32:49 一批 6 条 + 07:34:52 一批 4 条），此后到 09:36:29 重启前再没有任何新的 hung_task 告警。这不是因为卡顿解除了，而是命中了内核 `hung_task_warnings` sysctl 的默认值 **10**——一旦打印满 10 条告警，内核就不再继续打印新的告警，但*被卡的任务本身并不会因此解除阻塞*，这是一个纯粹的日志计数上限，不是恢复的信号。
- 07:37–09:32 之间日志里能看到的 `NetworkManager`（网络心跳）持续正常、以及 09:33 有一次正常的 SSH 登录，说明网络栈和登录/进程调度还能用，但**不代表 XFS/vdb 这条 I/O 路径已经恢复**——故障通报里 09:05 也明确写了"数据库底层无法执行命令"，即数据库自身的 I/O 相关命令始终执行不了，和"网络/SSH 能用但磁盘 I/O 卡住"完全吻合。
- 09:32:38 日志里的 `systemctl start clickhouse-server` 命令，对应的正是故障通报里运维人员 09:30 前后现场排查、尝试拉起服务的动作，而不是业务已经自愈后的常规操作。
- 真正解除阻塞的动作是故障通报里 09:37 的**重启实例虚机**，与日志里 09:36:29 的重启时间点吻合——即这次故障是靠**重启虚机**强制解除的，而不是像本文正文那次故障一样自己等 I/O 恢复后解除。

结论：这台主机的故障持续时间和官方通报的 **08:20–09:37（约 77 分钟）** 基本吻合，比正文那次故障（几分钟量级）严重得多；由于运维现场同时观察到"存储节点到宿主机丢包"，不排除这次是**网络丢包导致 vdb 的 I/O 长时间彻底不返回**（而不是短暂抖动），使得 `xfs-inodegc` worker 卡住的buffer 锁长期得不到 I/O completion 释放，只能靠重启虚机才能打断，正文那次故障后来经 vmcore 复核确认是真实的 ABBA 死锁（见「结论」及「vmcore 复核」一节）——但这台"43"主机目前只做过 dmesg 级别的分析，没有对应的 vmcore，还不能照搬同样的结论，只能说这次的持续时间和最终靠重启才解决，更像是 severity 更高的同一类问题，而不是一次轻微抖动，具体是不是同一种 ABBA 死锁需要 vmcore 才能坐实。

### `hung_task_warnings` 到底是不是 10：两份日志交叉验证 + 内核代码依据

**两份日志都恰好只打印了 10 条 hung_task 告警，批次划分却完全不同，强烈指向同一个日志计数上限，而不是巧合：**

| 日志 | 批次 1 | 批次 2 | 合计 |
|---|---|---|---|
| 正文（`dmesg.click.7.21.1.log`，15:11:00/15:13:03） | 5 条（`19100_node_expo`、`BgSchPool`、`AsyncMetrics`、`kworker/u34:3`、`kworker/2:0`） | 5 条（`telegraf`、`19100_node_expo` 再次、`titanagent`、`BgSchPool` 再次、`AsyncMetrics` 再次） | **10** |
| 本节（`messages-20260724`，07:32:49/07:34:52） | 6 条（`telegraf`、`BgSchPool`、`AsyncMetrics`、`node_exporter`、`kworker/u33:0`、`kworker/2:2`） | 4 条（`telegraf` 再次、`BgSchPool` 再次、`AsyncMetrics` 再次、`dcp-agent`） | **10** |

**内核代码依据**（当前仓库 `kernel/hung_task.c`，对照过故障机实际跑的 `ctkernel-lts-5.10/develop` 分支同一文件，逻辑与默认值完全一致）：

```c
// kernel/hung_task.c:60
static int __read_mostly sysctl_hung_task_warnings = 10;
```

```c
// kernel/hung_task.c:249-268，hung_task_info() 打印每一条 "INFO: task ... blocked" 之前的判断
if (sysctl_hung_task_warnings || hung_task_call_panic) {
        if (sysctl_hung_task_warnings > 0)
                sysctl_hung_task_warnings--;
        pr_err("INFO: task %s:%d blocked%s for more than %ld seconds.\n", ...);
        ...
        if (!sysctl_hung_task_warnings)
                pr_info("Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings\n");
}
```

即每打印一条告警就把这个计数器减 1，减到 0 之后 `if` 条件不再满足，**后续再有任务被判定为 hung，也不会打印任何东西**——是静默停止，不是问题解决了。`ctkernel-lts-5.10/develop` 的 `kernel/hung_task.c` 里同样是 `int __read_mostly sysctl_hung_task_warnings = 10;`，默认值和逻辑与 mainline 一致。

**一个额外的坑**：mainline 后来在 `pr_info("Future hung task reports are suppressed...")` 这行提示上做了改进（commit `b1f712b308dcd`，约 2023 年合入），5.10 vendor 树里**没有**这个提示——所以打光额度后日志上什么提示都不会有，只是安安静静地不再有新的 "INFO: task ... blocked" 出现，这也是本节一开始容易被误判成"卡顿自愈了"的原因。

**这不是"某段时间窗口内 10 次"，而是开机以来的累计余额，打光后永久静默**：翻遍 `kernel/hung_task.c` 全文，`sysctl_hung_task_warnings` 只有上面那一处递减逻辑，**没有任何地方会把它加回去**——不是滑动窗口、不会过一阵子自动恢复，是从**这次开机（或上一次手动重置）以来**的累计计数，减到 0 后永久停止打印，直到：

1. **重启**（静态变量重新初始化为 10），或
2. 手动写 sysctl：`echo 10 > /proc/sys/kernel/hung_task_warnings`（或写 `-1` 变成不限次数）

`messages-20260724` 这台机器的时间戳 `[19665253.xxx]` 折算约 **227 天**开机时长，而这次 07:32-07:34 的两批告警（6+4）恰好精确用满 10——如果这 227 天里更早还触发过别的 hung_task 告警，余额不可能在这次事件里正好被用得一分不剩。也就是说，这是这台机器**这次开机以来第一次、也是唯一一次**触发 hung_task 检测，额度是被这次事件完整、干净地用光的，不是叠加了历史上其他事件的余量。

**两份日志证据强度的区别**（严谨起见需要分开看）：

- `messages-20260724` 是持续写入的 syslog，第 10 条告警之后**继续观测了近 2 小时、期间确实再无任何新的 kernel 标签日志**，这是"额度打光而非已恢复"比较扎实的旁证。
- `dmesg.click.7.21.1.log` 更像是某个时间点抓取的 `dmesg` 快照，文件恰好在第 10 条告警的调用栈之后结束——不能排除"抓取时间点恰好在此附近"的巧合，没有类似"之后持续观测无新增"的独立佐证，只能说两份日志"总数都是 10"这一点本身就是很强的旁证，但对"之后是否被抑制"这件事，两份日志的确定性并不对等。

结合起来看：两次故障大概率都命中了同一个 `hung_task_warnings=10` 的默认额度，日志"没有更多告警"**不能**被当作"卡顿已经解除"的证据——这也是为什么正文「结论」里强调"没有 vmcore 无法 100% 确认持锁上下文"，以及本节现在把"长沙42 故障"的持续时间订正为跟着官方通报走（08:20–09:37），而不是只看 dmesg 里最后一条告警的时间点。

### 这批补丁能不能彻底解决 D 状态问题

> **本节结论需要结合「vmcore 复核」的最新发现重新评估：正文这次故障的根因确认是真实的 ABBA 死锁，不是慢 I/O。下面分两部分说：一是原来"消除放大效应"的判断依然成立、依然值得合入；二是死锁本身这批补丁完全解决不了，需要单独看待。**

**1. `xfs_inodegc_push()` 这类补丁能解决的，依然只是"放大"这一层**：把 `xfs_fs_statfs()` 从阻塞式 `flush` 改成非阻塞的 `push` 之后，即使 `xfs-inodegc` worker 本身还卡着（不管是卡在慢 I/O 还是卡在死锁），只是定时调 `statfs`/`df` 采集容量的监控/agent 进程（`telegraf`、`node_exporter`、`BgSchPool` 等）不会再被一起拖进 D 状态。这个结论不受"根因是死锁还是慢 I/O"影响，依然是应该合入的正确修复。

**2. 但死锁本身，这批补丁完全解决不了**——这是这次 vmcore 复核之后必须补充的新结论：

- `kworker/2:0`（xfs-inodegc）与 `kworker/u34:3`（writeback）之间的 ABBA 循环死锁，一旦形成就是**永久性**的，不会随时间或 I/O 恢复而自愈。`xfs_inodegc_push` 只是让 `statfs` 不再阻塞式等待 inodegc 队列，但 inodegc 队列本身（连同 writeback 那条链路）永远卡死，**该 AG 上所有需要 AGI/AGF/这块 inode cluster buffer 的后续操作都会持续挂起**——包括本文提到的 `HTTPHandler` 建新文件，合了 `xfs_inodegc_push` 之后它照样会卡，因为它卡的是 AGI 被 `kworker/2:0` 攥着，跟 `statfs` 阻塞放大链路完全无关。
- 换句话说：合了 `xfs_inodegc_push` 之后，**表面上看起来会"好一点"**——不会再有一大片监控 agent 进程一起进 D 状态、hung_task 告警的数量会大幅减少——但**这台机器这个 AG 的文件系统操作实质上已经死了**，只是死锁的影响范围从"全站瘫痪"缩小成了"这个 AG 相关的文件操作瘫痪"，业务侧仍然会持续故障，只有重启或者手工干预（比如 kill 掉其中一个卡住的 kworker 触发事务回滚，如果内核支持的话）才能解除。
- 这是否是一个**已知的、上游已经修过的 XFS bug**，还是这台内核版本上一个尚未被发现/修复的问题，需要额外去查（比如对照更新的内核版本源码，看 `xfs_difree`/`xfs_bmap_btalloc` 涉及 AGI/AGF/inode cluster buffer 加锁顺序的部分有没有相关的 fix commit），这部分工作本文目前还没有做，是后续排查建议里需要补的一项。

**3. 长沙42 那次故障（`messages-20260724`）目前还只做了 dmesg 级别的分析，没有对应的 vmcore**，所以还不能确定它是不是同样的 ABBA 死锁，还是真的只是慢 I/O/丢包导致的临时卡顿（`clickhouse-server` 反复重启失败这条证据本身两种情况都能解释）。如果条件允许，建议对这类故障也尽量保留 vmcore，用本节这套方法复核，而不要默认套用"等 I/O，会自愈"的结论。

**结论**：`xfs_inodegc_push` 这类补丁值得合入，但评估收益时不能再说"合了就能解决这次故障"——它只解决了故障的**放大效应**，正文这次事故真正的根因是一个 XFS 内核代码里的 ABBA 死锁，合了这批补丁之后死锁依然会发生，只是波及范围会小很多、告警会少很多，容易被误判成"问题解决了"。这个死锁本身需要作为一个独立的问题去定位、修复，不能和 `xfs_inodegc_push` 这条 statfs 优化混为一谈。

