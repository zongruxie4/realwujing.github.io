# BUG: kernel NULL pointer dereference, address: 000000000000060e

## crash

```bash
crash> bt
PID: 3996786  TASK: ffff96264e270000  CPU: 70   COMMAND: "ps"
 #0 [ffffa507c034fad8] crash_kexec at ffffffff83baec79
 #1 [ffffa507c034fae8] oops_end at ffffffff83a29a75
 #2 [ffffa507c034fb08] no_context at ffffffff83a7c9ac
 #3 [ffffa507c034fb40] __bad_area_nosemaphore at ffffffff83a7cab2
 #4 [ffffa507c034fb88] exc_page_fault at ffffffff844ce2bc
 #5 [ffffa507c034fbe0] asm_exc_page_fault at ffffffff84600afe
    [exception RIP: __d_lookup+65]
    RIP: ffffffff83db15e1  RSP: ffffa507c034fc98  RFLAGS: 00010206
    RAX: ffffa5078637f048  RBX: 00000000000005f6  RCX: 0000000000000007
    RDX: 0000000000b6fc09  RSI: ffffa507c034fdd0  RDI: ffff964615841518
    RBP: ffff964615841518   R8: 0000000000000000   R9: 0000000400000000
    R10: ffff964615841518  R11: ffffffff868b8b9c  R12: ffff964615841518
    R13: ffffa507c034fdd0  R14: 000000005b7e0484  R15: 0000000000008000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #6 [ffffa507c034fcc8] lookup_fast at ffffffff83d9ef78
 #7 [ffffa507c034fd08] open_last_lookups at ffffffff83da3254
 #8 [ffffa507c034fd68] path_openat at ffffffff83da3e28
 #9 [ffffa507c034fdb8] do_filp_open at ffffffff83da68c0
#10 [ffffa507c034fec8] do_sys_openat2 at ffffffff83d8dc07
#11 [ffffa507c034ff10] __x64_sys_openat at ffffffff83d8e2d4
#12 [ffffa507c034ff38] do_syscall_64 at ffffffff844cac00
#13 [ffffa507c034ff50] entry_SYSCALL_64_after_hwframe at ffffffff84600099
    RIP: 00007fbd0610f8eb  RSP: 00007ffd91e87c70  RFLAGS: 00000246
    RAX: ffffffffffffffda  RBX: 0000557c9d99e2d0  RCX: 00007fbd0610f8eb
    RDX: 0000000000000000  RSI: 00007ffd91e87e70  RDI: 00000000ffffff9c
    RBP: 00007ffd91e87e70   R8: 0000000000000008   R9: 0000000000000001
    R10: 0000000000000000  R11: 0000000000000246  R12: 0000000000000000
    R13: 0000557c9d99e2d0  R14: 0000000000000001  R15: 00000000003cfdc3
    ORIG_RAX: 0000000000000101  CS: 0033  SS: 002b
```

```bash
crash> bt -lsx
PID: 3996786  TASK: ffff96264e270000  CPU: 70   COMMAND: "ps"
 #0 [ffffa507c034fad8] crash_kexec+0x39 at ffffffff83baec79
    /usr/src/debug/kernel-5.10.0-136.12.0.90.ctl3.x86_64/linux-5.10.0-136.12.0.90.ctl3.x86_64/./arch/x86/include/asm/atomic.h: 41
 #1 [ffffa507c034fae8] oops_end+0x95 at ffffffff83a29a75
    /usr/src/debug/kernel-5.10.0-136.12.0.90.ctl3.x86_64/linux-5.10.0-136.12.0.90.ctl3.x86_64/arch/x86/kernel/dumpstack.c: 359
 #2 [ffffa507c034fb08] no_context+0x17c at ffffffff83a7c9ac
    /usr/src/debug/kernel-5.10.0-136.12.0.90.ctl3.x86_64/linux-5.10.0-136.12.0.90.ctl3.x86_64/arch/x86/mm/fault.c: 767
 #3 [ffffa507c034fb40] __bad_area_nosemaphore+0x52 at ffffffff83a7cab2
    /usr/src/debug/kernel-5.10.0-136.12.0.90.ctl3.x86_64/linux-5.10.0-136.12.0.90.ctl3.x86_64/arch/x86/mm/fault.c: 853
 #4 [ffffa507c034fb88] exc_page_fault+0x2dc at ffffffff844ce2bc
    /usr/src/debug/kernel-5.10.0-136.12.0.90.ctl3.x86_64/linux-5.10.0-136.12.0.90.ctl3.x86_64/arch/x86/mm/fault.c: 1354
 #5 [ffffa507c034fbe0] asm_exc_page_fault+0x1e at ffffffff84600afe
    /usr/src/debug/kernel-5.10.0-136.12.0.90.ctl3.x86_64/linux-5.10.0-136.12.0.90.ctl3.x86_64/./arch/x86/include/asm/idtentry.h: 571
    [exception RIP: __d_lookup+65]
    RIP: ffffffff83db15e1  RSP: ffffa507c034fc98  RFLAGS: 00010206
    RAX: ffffa5078637f048  RBX: 00000000000005f6  RCX: 0000000000000007
    RDX: 0000000000b6fc09  RSI: ffffa507c034fdd0  RDI: ffff964615841518
    RBP: ffff964615841518   R8: 0000000000000000   R9: 0000000400000000
    R10: ffff964615841518  R11: ffffffff868b8b9c  R12: ffff964615841518
    R13: ffffa507c034fdd0  R14: 000000005b7e0484  R15: 0000000000008000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
    /usr/src/debug/kernel-5.10.0-136.12.0.90.ctl3.x86_64/linux-5.10.0-136.12.0.90.ctl3.x86_64/fs/dcache.c: 2453
 #6 [ffffa507c034fcc8] lookup_fast+0xb8 at ffffffff83d9ef78
    /usr/src/debug/kernel-5.10.0-136.12.0.90.ctl3.x86_64/linux-5.10.0-136.12.0.90.ctl3.x86_64/fs/namei.c: 1510
 #7 [ffffa507c034fd08] open_last_lookups+0x144 at ffffffff83da3254
    /usr/src/debug/kernel-5.10.0-136.12.0.90.ctl3.x86_64/linux-5.10.0-136.12.0.90.ctl3.x86_64/fs/namei.c: 3203
 #8 [ffffa507c034fd68] path_openat+0x88 at ffffffff83da3e28
    /usr/src/debug/kernel-5.10.0-136.12.0.90.ctl3.x86_64/linux-5.10.0-136.12.0.90.ctl3.x86_64/fs/namei.c: 3423
 #9 [ffffa507c034fdb8] do_filp_open+0x90 at ffffffff83da68c0
    /usr/src/debug/kernel-5.10.0-136.12.0.90.ctl3.x86_64/linux-5.10.0-136.12.0.90.ctl3.x86_64/fs/namei.c: 3453
#10 [ffffa507c034fec8] do_sys_openat2+0x207 at ffffffff83d8dc07
    /usr/src/debug/kernel-5.10.0-136.12.0.90.ctl3.x86_64/linux-5.10.0-136.12.0.90.ctl3.x86_64/fs/open.c: 1180
#11 [ffffa507c034ff10] __x64_sys_openat+0x54 at ffffffff83d8e2d4
    /usr/src/debug/kernel-5.10.0-136.12.0.90.ctl3.x86_64/linux-5.10.0-136.12.0.90.ctl3.x86_64/fs/open.c: 1207
#12 [ffffa507c034ff38] do_syscall_64+0x40 at ffffffff844cac00
    /usr/src/debug/kernel-5.10.0-136.12.0.90.ctl3.x86_64/linux-5.10.0-136.12.0.90.ctl3.x86_64/arch/x86/entry/common.c: 47
#13 [ffffa507c034ff50] entry_SYSCALL_64_after_hwframe+0x61 at ffffffff84600099
    /usr/src/debug/kernel-5.10.0-136.12.0.90.ctl3.x86_64/linux-5.10.0-136.12.0.90.ctl3.x86_64/arch/x86/entry/entry_64.S: 132
    RIP: 00007fbd0610f8eb  RSP: 00007ffd91e87c70  RFLAGS: 00000246
    RAX: ffffffffffffffda  RBX: 0000557c9d99e2d0  RCX: 00007fbd0610f8eb
    RDX: 0000000000000000  RSI: 00007ffd91e87e70  RDI: 00000000ffffff9c
    RBP: 00007ffd91e87e70   R8: 0000000000000008   R9: 0000000000000001
    R10: 0000000000000000  R11: 0000000000000246  R12: 0000000000000000
    R13: 0000557c9d99e2d0  R14: 0000000000000001  R15: 00000000003cfdc3
    ORIG_RAX: 0000000000000101  CS: 0033  SS: 002b
```

```bash
crash> files 3996786
PID: 3996786  TASK: ffff96264e270000  CPU: 70   COMMAND: "ps"
ROOT: /    CWD: /home/secure
 FD       FILE            DENTRY           INODE       TYPE PATH
  0 ffff9645f2196800 ffff9646df18af30 ffff9645a19738c8 FIFO
  1 ffff9646a8ff1e00 ffff96460c2e90e0 ffff9586fbf5eca0 FIFO
  2 ffff9646778ce1c0 ffff9646df18b290 ffff9645a19758e0 FIFO
  3 ffff964603e63980 ffff95861b502288 ffff95861b516b88 DIR  /proc/
```

```c
// vim fs/namei.c +1510

1504         if (unlazy_child(nd, dentry, seq))
1505             return ERR_PTR(-ECHILD);
1506         if (unlikely(status == -ECHILD))
1507             /* we'd been told to redo it in non-rcu mode */
1508             status = d_revalidate(dentry, nd->flags);
1509     } else {
1510         dentry = __d_lookup(parent, &nd->last);
1511         if (unlikely(!dentry))
1512             return NULL;
1513         status = d_revalidate(dentry, nd->flags);
1514     }
```

```c
// vim fs/dcache.c +2421

2421 struct dentry *__d_lookup(const struct dentry *parent, const struct qstr *name)
2422 {
2423     unsigned int hash = name->hash;
2424     struct hlist_bl_head *b = d_hash(hash);
```

```c
// vim fs/dcache.c +101

 101 static inline struct hlist_bl_head *d_hash(unsigned int hash)
 102 {
 103     return dentry_hashtable + (hash >> d_hash_shift);
 104 }
```

```c
// vim fs/dcache.c +2453

2451     hlist_bl_for_each_entry_rcu(dentry, node, b, d_hash) {
2452
2453         if (dentry->d_name.hash != hash)
2454             continue;
```

```bash
crash> struct dentry.d_iname ffff964615841518
  d_iname = "3997123\000ervice\000ce\000\000tty.slice\000\000\000",
crash> struct dentry.d_parent ffff964615841518
  d_parent = 0xffff95861b502288,
crash> struct dentry.d_iname 0xffff95861b502288
  d_iname = "/\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000",
crash> struct dentry.d_parent 0xffff95861b502288
  d_parent = 0xffff95861b502288,
```

```bash
crash> struct dentry.d_name ffff964615841518
  d_name = {
    {
      {
        hash = 1624861078,
        len = 7
      },
      hash_len = 31689632150
    },
    name = 0xffff964615841550 "3997123"
  },
```

```bash
crash> struct dentry.d_inode ffff964615841518
  d_inode = 0xffff9626fdc5d2c8,
crash> struct inode.i_rdev 0xffff9626fdc5d2c8
  i_rdev = 0,
```

```bash
crash> struct qstr ffffa507c034fdd0
struct qstr {
  {
    {
      hash = 1534985348,
      len = 4
    },
    hash_len = 18714854532
  },
  name = 0xffff96270489b02e "ctty"
}
```

```bash
crash> struct qstr.hash ffffa507c034fdd0
      hash = 1534985348,
```

```bash
crash> eval 16
hexadecimal: 10
    decimal: 16
      octal: 20
     binary: 0000000000000000000000000000000000000000000000000000000000010000
crash> eval 0xffffa507c034fdd0 - 0x10
hexadecimal: ffffa507c034fdc0
    decimal: 18446644051440893376  (-100022268658240)
      octal: 1777775120370015176700
     binary: 1111111111111111101001010000011111000000001101001111110111000000
```

```bash
crash> struct nameidata ffffa507c034fdc0
struct nameidata {
  path = {
    mnt = 0xffff96059259f4a0,
    dentry = 0xffff964615841518
  },
  last = {
    {
      {
        hash = 1534985348,
        len = 4
      },
      hash_len = 18714854532
    },
    name = 0xffff96270489b02e "ctty"
  },
  root = {
    mnt = 0xffff966592a20de0,
    dentry = 0xffff96858e777a28
  },
  inode = 0xffff9626fdc5d2c8,
  flags = 257,
  state = 2,
  seq = 0,
  m_seq = 19851860,
  r_seq = 1071248,
  last_type = 0,
  depth = 0,
  total_link_count = 0,
  stack = 0xffffa507c034fe20,
  internal = {{
      link = {
        mnt = 0x7ffc09853ef0,
        dentry = 0x6
      },
      done = {
        fn = 0xffffffff83b7b103 <rcu_nocb_try_bypass+499>,
        arg = 0xffffffff83d69a6f <page_counter_try_charge+47>
      },
      name = 0x4 <error: Cannot access memory at address 0x4>,
      seq = 3119488576
    }, {
      link = {
        mnt = 0x0,
        dentry = 0x9fff6
      },
      done = {
        fn = 0x40,
        arg = 0xffffffff83e2491c <files_cgroup_alloc_fd+92>
      },
      name = 0x0,
      seq = 2085869312
    }},
  name = 0xffff96270489b000,
  saved = 0x0,
  root_seq = 2,
  dfd = -100,
  dir_uid = {
    val = 1000
  },
  dir_mode = 16749
}
```

判断 `internal` 栈是否被踩（即内存是否被非法覆盖或破坏）的结论，是通过综合分析你提供的各种数据（`struct nameidata`、内存转储、调用栈等）得出的。以下是我逐步分析的过程和得出结论的依据：

---

### 分析过程与证据

#### 1. `internal` 字段的异常值
从 `struct nameidata` 的 `internal` 数组（位于 `0xffffa507c034fe20` - `0xffffa507c034fe77`）中，我们观察到以下内容：
```
internal = {{
    link = {
      mnt = 0x7ffc09853ef0,       // 用户空间地址，异常
      dentry = 0x6                // 低地址，异常
    },
    done = {
      fn = 0xffffffff83b7b103 <rcu_nocb_try_bypass+499>, // 有效内核函数
      arg = 0xffffffff83d69a6f <page_counter_try_charge+47> // 有效参数
    },
    name = 0x4 <error: Cannot access memory at address 0x4>, // 异常值
    seq = 3119488576            // 有效序列号
  }, {
    link = {
      mnt = 0x0,                 // 空，正常
      dentry = 0x9fff6           // 低地址，异常
    },
    done = {
      fn = 0x40,                 // 异常值
      arg = 0xffffffff83e2491c <files_cgroup_alloc_fd+92> // 有效参数
    },
    name = 0x0,                  // 空，正常
    seq = 2085869312            // 有效序列号
  }},
```
- **异常点**：
  - **`internal[0].name = 0x4`**：路径名指针应指向有效字符串（如 `0xffff96270489b02e "ctty"`），但 `0x4` 是一个低地址，且不可访问（`<error: Cannot access memory at address 0x4>`）。
  - **`internal[1].done.fn = 0x40`**：回调函数指针应为内核地址（如 `0xffffffff83b7b103`），但 `0x40` 是一个无效的低地址。
  - **`internal[0].link.dentry = 0x6` 和 `internal[1].link.dentry = 0x9fff6`**：目录项指针应为内核地址（如 `0xffff964615841518`），这些低值异常。
- **初步判断**：这些字段的值不符合预期（内核数据结构中不应出现如此低的地址），表明它们可能被非法覆盖。

#### 2. 内存转储的对比
从 `rd -x 0xffffa507c034fe00 32` 的输出中，`internal` 区域（`0xffffa507c034fe20` - `0xffffa507c034fe77`）与上述结构一致：
```
ffffa507c034fe20:  00007ffc09853ef0 0000000000000006  // link.mnt, link.dentry
ffffa507c034fe30:  ffffffff83b7b103 ffffffff83d69a6f  // done.fn, done.arg
ffffa507c034fe40:  0000000000000004 ffff9587b9ef9e40  // name, seq
ffffa507c034fe50:  0000000000000000 000000000009fff6  // link.mnt, link.dentry
ffffa507c034fe60:  0000000000000040 ffffffff83e2491c  // done.fn, done.arg
ffffa507c034fe70:  0000000000000000 52e37d4e7c53d700  // name, seq
```
- **确认异常**：
  - `0xffffa507c034fe40: 0000000000000004` → `name = 0x4`
  - `0xffffa507c034fe60: 0000000000000040` → `done.fn = 0x40`
  - `0xffffa507c034fe28: 0000000000000006` → `link.dentry = 0x6`
  - `0xffffa507c034fe58: 000000000009fff6` → `link.dentry = 0x9fff6`
- **其他字段正常**：`done.fn` 和 `done.arg` 的其他值（如 `0xffffffff83b7b103`）是有效的内核地址，`seq` 也合理。
- **推测**：部分字段被破坏，而其他字段保持完整，表明覆盖是局部的。

#### 3. 栈布局与溢出可能性
- **栈地址**：
  - `struct nameidata` 位于 `0xffffa507c034fdc0`。
  - `stack` 指针为 `0xffffa507c034fe20`，包含 `internal`。
  - 调用栈从 `0xffffa507c034fec8`（`do_sys_openat2+519`）开始。
- **溢出方向**：x86_64 栈从高地址向低地址增长。如果溢出发生在更高地址（如 `0xffffa507c034ffxx`），可能向下覆盖 `0xffffa507c034fe40`（`name`）和 `0xffffa507c034fe60`（`done.fn`）。
- **重复值**：
  - `0x4` 在 `0xffffa507c034fec0` 再次出现：
    ```
    ffffa507c034fec0:  0000000000000004 ffffffff83d8dc07
    ```
  - 这可能是溢出源，暗示更高地址的数据（如局部变量）覆盖了 `internal`。

#### 4. 调用栈与上下文
调用栈（`bt`）显示：
```
 #5 [ffffa507c034fbe0] asm_exc_page_fault
    [exception RIP: __d_lookup+65]
 #6 [ffffa507c034fcc8] lookup_fast
 #8 [ffffa507c034fd68] path_openat
#10 [ffffa507c034fec8] do_sys_openat2
#13 [ffffa507c034ff50] entry_SYSCALL_64_after_hwframe
```
- **崩溃点**：`__d_lookup+65` 触发页面错误，可能与 `struct qstr`（`0xffffa507c034fdd0`，`"ctty"`）或 `dentry`（`0xffff964615841518`）相关。
- **栈使用**：`path_openat` 使用 `struct nameidata` 解析路径，`internal` 是其临时栈。如果栈分配或操作不当，可能导致覆盖。
- **异常值来源**：`0x4` 和 `0x40` 不像是正常初始值（通常为 `0` 或有效指针），更像是溢出或错误写入的结果。

#### 5. 模式与一致性
- **局部破坏**：`internal` 中只有特定字段（`name`, `done.fn`, `link.dentry`）异常，其他字段（如 `done.arg`, `seq`）正常，表明不是整个栈被清空，而是局部覆盖。
- **重复数据**：`0x4` 和 `0x52e37d4e7c53d700`（`seq`）在栈中多次出现，可能反映溢出或未清理的栈状态。
- **用户态痕迹**：`internal[0].link.mnt = 0x7ffc09853ef0` 是用户空间地址，可能被错误写入内核栈。

---

### 得出结论的依据
基于以上分析，我得出“`internal` 栈被踩”的结论，理由如下：

1. **字段值异常**：
   - `name = 0x4` 和 `done.fn = 0x40` 是无效的低地址，与内核数据结构的预期（高地址指针）不符。
   - `link.dentry = 0x6` 和 `0x9fff6` 同样异常。

2. **局部覆盖特征**：
   - 只有部分字段被破坏，其他字段（如 `done.arg`）保持有效，符合栈被踩的典型模式（溢出通常只影响部分区域）。

3. **溢出证据**：
   - `0x4` 在 `0xffffa507c034fec0` 重复出现，位于调用栈附近，可能是从更高地址溢出的结果。
   - 栈向下增长的特性支持从 `0xffffa507c034fexx` 覆盖到 `0xffffa507c034fe40` 的可能性。

4. **上下文支持**：
   - `path_openat` 和 `lookup_fast` 操作 `struct nameidata`，若栈分配过大或指针越界，可能导致 `internal` 被覆盖。
   - 崩溃在 `__d_lookup` 表明路径解析出错，`internal` 的异常可能是同一问题的副作用。

---

### 结论
**`internal` 栈被踩了**，表现为 `name = 0x4`、`done.fn = 0x40` 等字段被非法覆盖。覆盖是局部的，可能是从更高地址（如 `0xffffa507c034fec0`）的栈溢出导致。根本原因可能与 `path_openat` 或 `lookup_fast` 中的栈操作有关，建议进一步检查 `__d_lookup` 的崩溃点和相关数据结构（`dentry`, `qstr`）以确认溢出源。

```bash
crash> p dentry_hashtable
dentry_hashtable = $2 = (struct hlist_bl_head *) 0xffffa50780801000
```

```bash
crash> p d_hash_shift
d_hash_shift = $3 = 7
```

```bash
crash> struct -o hlist_bl_head
struct hlist_bl_head {  [0] struct hlist_bl_node *first;
}SIZE: 8
```

```bash
python3 -c "print(hex(0xffffa50780801000 + (1534985348 >> 7) * 8))"
0xffffa5078637f048
```

```bash
crash> struct hlist_bl_head 0xffffa5078637f048
struct hlist_bl_head {
  first = 0x5f6
}
```

`first = 0x5f6`与`RBX: 00000000000005f6`相同。

```bash
crash> struct -o dentry
struct dentry {
    [0] unsigned int d_flags;
    [4] seqcount_spinlock_t d_seq;
    [8] struct hlist_bl_node d_hash;
   [24] struct dentry *d_parent;
   [32] struct qstr d_name;
   [48] struct inode *d_inode;
   [56] unsigned char d_iname[32];
   [88] struct lockref d_lockref;
   [96] const struct dentry_operations *d_op;
  [104] struct super_block *d_sb;
  [112] unsigned long d_time;
  [120] void *d_fsdata;
        union {
  [128]     struct list_head d_lru;
  [128]     wait_queue_head_t *d_wait;
        };
  [144] struct list_head d_child;
  [160] struct list_head d_subdirs;
        union {
            struct hlist_node d_alias;
            struct hlist_bl_node d_in_lookup_hash;
            struct callback_head d_rcu;
  [176] } d_u;
  [192] atomic_t d_neg_dnum;
  [200] u64 kabi_reserved1;
  [208] u64 kabi_reserved2;
}
SIZE: 216
```

```bash
struct hlist_bl_node {
   [0] struct hlist_bl_node *next;
   [8] struct hlist_bl_node **pprev;
}
SIZE: 16
```

```bash
crash> struct -o hlist_bl_head
struct hlist_bl_head {
  [0] struct hlist_bl_node *first;
}
SIZE: 8
```

```bash
struct qstr {
       union {
           struct {
   [0]         u32 hash;
   [4]         u32 len;
           };
   [0]     u64 hash_len;
       };
   [8] const unsigned char *name;
}
SIZE: 16
```

```bash
/usr/src/debug/kernel-5.10.0-136.12.0.90.ctl3.x86_64/linux-5.10.0-136.12.0.90.ctl3.x86_64/fs/dcache.c: 2453
0xffffffff83db15e1 <__d_lookup+0x41>:   cmp    %r14d,0x18(%rbx)
0xffffffff83db15e5 <__d_lookup+0x45>:   jne    0xffffffff83db15d9 <__d_lookup+0x39>
```

```bash
crash> eval 0x5f6 + 0x18
hexadecimal: 60e
    decimal: 1550
      octal: 3016
     binary: 0000000000000000000000000000000000000000000000000000011000001110
```

```bash
crash> rd 60e
rd: invalid user virtual address: 60e  type: "64-bit UVADDR"
```

```bash
crash> p dentry_hashtable
dentry_hashtable = $4 = (struct hlist_bl_head *) 0xffffa50780801000
```

```bash
crash> struct hlist_bl_head ffffa50780801000
struct hlist_bl_head {
  first = 0x0
}
```

```text
[dentry_hashtable] (数组，每个元素是一个 hlist_bl_head)
        |
        |  index = hash >> d_hash_shift
        v
+--------------------------+
| struct hlist_bl_head    |   <- dentry_hashtable[index]
|--------------------------|
| *first --> d_hash node  |   -> 错误地指向了 0x5f6
+--------------------------+
        |
        v
+--------------------------+
| struct hlist_bl_node    |   <- dentry->d_hash
|--------------------------|
| *next                   |
| **pprev                 |
+--------------------------+
        |
        v
+--------------------------+
| struct dentry           |
|--------------------------|
| d_flags                 |
| d_seq                   |
| d_hash (内嵌结构体)     | <- 用于散列链表
| d_parent                |
| d_name (qstr)           | <- 用于名称匹配
|   |- hash               |
|   |- len                |
|   |- *name              |
| d_inode                 |
| d_op                    |
| d_sb                    |
| ...                     |
+--------------------------+
```

```c
// vim fs/namei.c +2154

2146         if (likely(type == LAST_NORM)) {
2147             struct dentry *parent = nd->path.dentry;
2148             nd->state &= ~ND_JUMPED;
2149             if (unlikely(parent->d_flags & DCACHE_OP_HASH)) {
2150                 struct qstr this = { { .hash_len = hash_len }, .name = name };
2151                 err = parent->d_op->d_hash(parent, &this);
2152                 if (err < 0)
2153                     return err;
2154                 hash_len = this.hash_len;
2155                 name = this.name;
2156             }
2157         }
```

```bash
crash> struct dentry.d_op,d_sb ffff964615841518
  d_op = 0xffffffff84a53740 <pid_dentry_operations>,
  d_sb = 0xffff9625911ca000,
```

```bash
crash> struct dentry_operations 0xffffffff84a53740
struct dentry_operations {
  d_revalidate = 0xffffffff83e3e8b0 <pid_revalidate>,
  d_weak_revalidate = 0x0,
  d_hash = 0x0,
  d_compare = 0x0,
  d_delete = 0xffffffff83e3a3d0 <pid_delete_dentry>,
  d_init = 0x0,
  d_release = 0x0,
  d_prune = 0x0,
  d_iput = 0x0,
  d_dname = 0x0,
  d_automount = 0x0,
  d_manage = 0x0,
  d_real = 0x0,
  kabi_reserved1 = 0,
  kabi_reserved2 = 0,
  kabi_reserved3 = 0,
  kabi_reserved4 = 0
}
```

```bash
crash> struct super_block.s_type,s_id 0xffff9625911ca000
  s_type = 0xffffffff856a89e0 <proc_fs_type>,
  s_id = "proc\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000",
```

```bash
crash> struct file_system_type 0xffffffff856a89e0
struct file_system_type {
  name = 0xffffffff84d57275 "proc",
  fs_flags = 24,
  init_fs_context = 0xffffffff83e39f80 <proc_init_fs_context>,
  parameters = 0xffffffff84a50820 <proc_fs_parameters>,
  mount = 0x0,
  kill_sb = 0xffffffff83e39c70 <proc_kill_sb>,
  owner = 0x0,
  next = 0xffffffff85636c40 <cgroup_fs_type>,
  fs_supers = {
    first = 0xffff9666b22298e0
  },
  s_lock_key = {<No data fields>},
  s_umount_key = {<No data fields>},
  s_vfs_rename_key = {<No data fields>},
  s_writers_key = 0xffffffff856a8a28 <proc_fs_type+72>,
  i_lock_key = {<No data fields>},
  i_mutex_key = {<No data fields>},
  i_mutex_dir_key = {<No data fields>},
  kabi_reserved1 = 0,
  kabi_reserved2 = 0,
  kabi_reserved3 = 0,
  kabi_reserved4 = 0
}
```

```bash
crash> struct dentry.d_lockref.count ffff964615841518
  d_lockref.count = 1
```
