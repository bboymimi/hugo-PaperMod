---
title: "Using Bpftrace to Detect Deadlock in Hugetlb Subsystem"
date: 2025-09-26T13:56:27+08:00
draft: false
description: "The practical application of Bpftrace to identify and analyze deadlock in the hugetlb subsystem, from the initial problem analysis to the final patch."
summary: ""
tags: ["hugetlb", "bpftrace", "linux kernel", "deadlock", "memory management", "tracing"]
categories: ["linux kernel", "memory management", "tracing"]
series: ["Themes Guide"]
ShowToc: true
TocOpen: true
cover:
  image: images/bpftrace.png
  caption: "Using Bpftrace to Detect Deadlock in Hugetlb Subsystem"
social:
  fediverse_creator: "@gavinguo@igalia.com"
---
In this blog post, we'll explore how to use Bpftrace to investigate and analyze a deadlock issue in the Linux kernel's hugetlb subsystem. We'll walk through the process of identifying the root cause using Bpftrace's tracing capabilities, examining kernel function calls and variables in real-time, and finally developing a solution. Let's get started!

## Bpftrace

Bpftrace is a high-level tracing language with scriptable syntax support for Linux that leverages eBPF (Extended Berkeley Packet Filter) technology for powerful kernel observability. Created by Alastair Robertson and significantly advanced by [Brendan Gregg](https://www.brendangregg.com/), it's inspired by awk, C, and predecessors like DTrace and SystemTap.

Bpftrace is good at quickly prototyping and tracing kernel issues, allowing developers to print out specific kernel variables on the fly and conveniently address complex system problems. For a deeper look into bpftrace, see [Brendan Gregg's A thorough introduction to bpftrace](https://www.brendangregg.com/blog/2019-08-19/bpftrace.html).

Some useful oneliners to get started with Bpftrace:


Print out the Linux kernel version banner directly from kernel memory using kaddr() to access the linux_banner symbol:
```bash
$ sudo BPFTRACE_MAX_STRLEN=200 bpftrace -q -e 'BEGIN { printf("banner: %s\n", str(kaddr("linux_banner"))); exit() }'
banner: Linux version 6.14.0-27-generic (buildd@lcy02-amd64-120) (x86_64-linux-gnu-gcc-13 (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0, GNU ld (GNU Binutils for Ubuntu) 2.42) #27~24.04.1-Ubuntu SMP PREEMPT_DYNAMIC ..
```

Print out the min_free_kbytes value from procfs:
```bash
$ cat /proc/sys/vm/min_free_kbytes
90112

$ sudo bpftrace -e 'BEGIN { printf("min_free_kbytes: %d\n", *(int32 *)kaddr("min_free_kbytes"));exit()}'
Attaching 1 probe...
min_free_kbytes: 90112
```


## The Problem
A deadlock occurred in the hugetlb subsystem where two tasks were mutually waiting for each other - one task was blocked trying to acquire a mutex in hugetlb_wp() while the other task was stuck waiting for a page in hugetlb_fault(). This circular dependency between the tasks resulted in both being unable to make progress for over 60 seconds, triggering the hung_task watchdog to print out the potential lock dependencies. While this is quite specific to the hugetlb subsystem, if you want to skip directly to learning how to use bpftrace to debug kernel issues, feel free to jump ahead to the [bpfscript to detect deadlock in hugetlb subsystem](https://gavinguo.cc/posts/bpftrace-detect-deadlock-hugetlb/#bpftrace-script-to-detect-deadlock-in-hugetlb-subsystem) section.

The [deadlock](https://drive.google.com/file/d/1bWq2-8o-BJAuhoHWX7zAhI6ggfhVzQUI/view) was reproduced and triggered by an internal [syzkaller reproducer](https://drive.google.com/file/d/1DVRnIW-vSayU5J1re9Ct_br3jJQU6Vpb/view?usp=drive_link):
```bash
INFO: task repro_20250402_:13229 blocked for more than 64 seconds.
      Not tainted 6.15.0-rc3+ #24
"echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
task:repro_20250402_ state:D stack:25856 pid:13229 tgid:13228 ppid:3513   task_flags:0x400040 flags:0x00004006
Call Trace:
 <TASK>
 __schedule+0x1755/0x4f50
 schedule+0x158/0x330
 schedule_preempt_disabled+0x15/0x30
 __mutex_lock+0x75f/0xeb0
 hugetlb_wp+0xf88/0x3440
 hugetlb_fault+0x14c8/0x2c30
 trace_clock_x86_tsc+0x20/0x20
 do_user_addr_fault+0x61d/0x1490
 exc_page_fault+0x64/0x100
 asm_exc_page_fault+0x26/0x30
RIP: 0010:__put_user_4+0xd/0x20
 copy_process+0x1f4a/0x3d60
 kernel_clone+0x210/0x8f0
 __x64_sys_clone+0x18d/0x1f0
 do_syscall_64+0x6a/0x120
 entry_SYSCALL_64_after_hwframe+0x76/0x7e
RIP: 0033:0x41b26d
 </TASK>
INFO: task repro_20250402_:13229 is blocked on a mutex likely owned by task repro_20250402_:13250.
task:repro_20250402_ state:D stack:28288 pid:13250 tgid:13228 ppid:3513   task_flags:0x400040 flags:0x00000006
Call Trace:
 <TASK>
 __schedule+0x1755/0x4f50
 schedule+0x158/0x330
 io_schedule+0x92/0x110
 folio_wait_bit_common+0x69a/0xba0
 __filemap_get_folio+0x154/0xb70
 hugetlb_fault+0xa50/0x2c30
 trace_clock_x86_tsc+0x20/0x20
 do_user_addr_fault+0xace/0x1490
 exc_page_fault+0x64/0x100
 asm_exc_page_fault+0x26/0x30
RIP: 0033:0x402619
 </TASK>
INFO: task repro_20250402_:13250 blocked for more than 65 seconds.
      Not tainted 6.15.0-rc3+ #24
"echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
task:repro_20250402_ state:D stack:28288 pid:13250 tgid:13228 ppid:3513   task_flags:0x400040 flags:0x00000006
Call Trace:
 <TASK>
 __schedule+0x1755/0x4f50
 schedule+0x158/0x330
 io_schedule+0x92/0x110
 folio_wait_bit_common+0x69a/0xba0
 __filemap_get_folio+0x154/0xb70
 hugetlb_fault+0xa50/0x2c30
 trace_clock_x86_tsc+0x20/0x20
 do_user_addr_fault+0xace/0x1490
 exc_page_fault+0x64/0x100
 asm_exc_page_fault+0x26/0x30
RIP: 0033:0x402619
 </TASK>

Showing all locks held in the system:
1 lock held by khungtaskd/35:
 #0: ffffffff879a7440 (rcu_read_lock){....}-{1:3}, at: debug_show_all_locks+0x30/0x180
2 locks held by repro_20250402_/13229:
 #0: ffff888017d801e0 (&mm->mmap_lock){++++}-{4:4}, at: lock_mm_and_find_vma+0x37/0x300
 #1: ffff888000fec848 (&hugetlb_fault_mutex_table[i]){+.+.}-{4:4}, at: hugetlb_wp+0xf88/0x3440
3 locks held by repro_20250402_/13250:
 #0: ffff8880177f3d08 (vm_lock){++++}-{0:0}, at: do_user_addr_fault+0x41b/0x1490
 #1: ffff888000fec848 (&hugetlb_fault_mutex_table[i]){+.+.}-{4:4}, at: hugetlb_fault+0x3b8/0x2c30
 #2: ffff8880129500e8 (&resv_map->rw_sema){++++}-{4:4}, at: hugetlb_fault+0x494/0x2c30
```


## Analysis

This is a ABBA deadlock involving two locks:
- `Lock A`: pagecache_folio lock
- `Lock B`: hugetlb_fault_mutex_table lock

```bash
Process 1                              Process 2
---				       ---
hugetlb_fault
  mutex_lock(B) // take B
  filemap_lock_hugetlb_folio
    filemap_lock_folio
      __filemap_get_folio
        folio_lock(A) // take A
  hugetlb_wp
    mutex_unlock(B) // release B
    ...                                hugetlb_fault
    ...                                  mutex_lock(B) // take B
                                         filemap_lock_hugetlb_folio
                                           filemap_lock_folio
                                             __filemap_get_folio
                                               folio_lock(A) // blocked
    unmap_ref_private
    ...
    mutex_lock(B) // retake and blocked
```

The deadlock occurs between two processes as follows:
- The first process (let’s call it **Process 1**) is handling a
*copy-on-write (COW)* operation on a hugepage via hugetlb_wp. Due to
insufficient reserved hugetlb pages, **Process 1**, owner of the reserved
hugetlb page, attempts to *unmap* a hugepage owned by another process
(non-owner) to satisfy the reservation. Before unmapping, **Process 1**
acquires `lock B` (*hugetlb_fault_mutex_table lock*) and then `lock A`
(*pagecache_folio lock*). To proceed with the unmap, it releases `Lock B`
but retains `Lock A`. After the unmap, **Process 1** tries to reacquire `Lock
B`. However, at this point, `Lock B` has already been acquired by another
process.

- The second process (**Process 2**) enters the *hugetlb_fault* handler
during the unmap operation. It successfully acquires `Lock B`
(*hugetlb_fault_mutex_table* lock) that was just released by **Process 1**,
but then attempts to acquire `Lock A` (*pagecache_folio* lock), which is
still held by **Process 1**.

As a result, **Process 1** (holding `Lock A`) is blocked waiting for `Lock B`
(held by **Process 2**), while **Process 2** (holding `Lock B`) is blocked waiting
for `Lock A` (held by **Process 1**), constructing a ABBA deadlock scenario.

## Bpftrace script to detect deadlock in hugetlb subsystem

I'll go through the script step by step. This is the [bpftrace script](https://github.com/bboymimi/bpftracer/blob/master/scripts/hugetlb_lock_debug.bt) for reference.

### Part 1: Initialize the "*gate*"

```bash{linenos=table,hl_lines=[4,7,9,12,14]}
BEGIN
{
    printf("Tracing hugetlb lock ordering... Hit Ctrl-C to end.\n");
    @hugetlb_fault_entry = 0;
}

kprobe:hugetlb_fault
{
    @hugetlb_fault_entry++;
}

kretprobe:hugetlb_fault
{
    @hugetlb_fault_entry--;
}
```
This section sets up a "**gate**" to ensure we only trace events happening inside the *hugetlb_fault* function.

- `BEGIN`: This block runs once when the script starts. It initializes a global counter variable `@hugetlb_fault_entry` to zero.

- `kprobe:hugetlb_fault`: A kprobe triggers at the entry of the *hugetlb_fault* function. Here, every time the *hugetlb_fault* kernel function is called, the counter is incremented.

- `kretprobe:hugetlb_fault`: A kretprobe triggers on the exit of the *hugetlb_fault* function. When *hugetlb_fault* finishes, the counter is decremented.

By doing this, `@hugetlb_fault_entry` will be greater than zero only when the code is executing within the scope of the *hugetlb_fault* function. This ensures the following deadlock detection code only runs when the code is executing within the scope of the *hugetlb_fault* function. The reason to only capture the events within the *hugetlb_fault* function is that the mutex and the folio locks are acquired and released within the *hugetlb_fault* function. We don't want to capture the events outside of the *hugetlb_fault* function, which would be too noisy and distract the analysis, noticeably slowing down the reproducer.

### Part 2: Tracing Mutex Locks

```bash{linenos=table,hl_lines=[1,2,22,23]}
kprobe:__mutex_lock
/@hugetlb_fault_entry && !strncmp(comm, "repr", 4)/
{
    // If this mutex is already being tracked, we've found a potential deadlock.
    if (@mutex_ts[arg0]) {
        printf("%s pid:%d tid:%d is blocked for mutex %lx\n", comm, pid, tid, arg0);
        print(kstack);
        printf("The mutex has been locked by: pid: %d, tid: %d, comm: %s, stack:%s\n",
                @mutex_pid[arg0], @mutex_tid[arg0],
                @mutex_comm[arg0], @mutex_stack[arg0]);
        return;
    }

    // First time seeing this mutex lock, so record who locked it.
    @mutex_ts[arg0] = nsecs;
    @mutex_stack[arg0] = kstack;
    @mutex_pid[arg0] = pid;
    @mutex_tid[arg0] = tid;
    @mutex_comm[arg0] = comm;
}

kprobe:mutex_unlock
/@hugetlb_fault_entry && !strncmp(comm, "repr", 4)/
{
    // Clean up the tracking maps when the mutex is released.
    delete(@mutex_ts[arg0]);
    delete(@mutex_stack[arg0]);
    delete(@mutex_pid[arg0]);
    delete(@mutex_tid[arg0]);
    delete(@mutex_comm[arg0]);
}
```
This is the logic for the first lock type: *mutexes*.

- `kprobe:__mutex_lock`. This triggers whenever a thread attempts to acquire a mutex.

- `/@hugetlb_fault_entry && !strncmp(comm, "repr", 4)`. This is crucial. The code <mark>only</mark> runs if both conditions are met:

  - `@hugetlb_fault_entry` is non-zero (it's the "*gate*" as mentioned in Part 1, which means we're inside hugetlb_fault).

  - `!strncmp(comm, "repr", 4)`: The command name of the current process starts with "repr", which is the name of the syzkaller reproducer.

- **Deadlock Detection**:

The script checks if the mutex's memory address (`arg0` holds the first argument to `__mutex_lock`) already exists as a key in the map `@mutex_ts`.

If it does exist, it means another thread has already locked this mutex within our traced context. Then, the script prints out detailed information about both the blocked thread and the thread that is holding the lock, including their process/thread IDs and kernel call stacks.

If it does not exist, this is the first time we encounter this lock. The script stores the current timestamp, stack trace, PID, and command name in several maps, using the mutex address (arg0) as the key.

`kprobe:mutex_unlock`: When a mutex is unlocked, the tracking information is **removed** from maps.

### Part 3: Tracing Folio Locks

```bash{linenos=table,hl_lines=[1,2,25,26]}
kretprobe:filemap_get_entry
/@hugetlb_fault_entry && !strncmp(comm, "repr", 4)/
{
    $folio = (uint64)retval;
    // ... error checking omitted for brevity ...

    // If this folio is already being tracked, we've found a potential deadlock.
    if (@folio_ts[$folio]) {
        printf("%s pid:%d tid:%d is blocked for folio %lx\n", comm, pid, tid, $folio);
        print(kstack);
        printf("The folio has been locked by: pid: %d, tid: %d, comm: %s, stack:%s\n",
                @folio_pid[$folio], @folio_tid[$folio],
                @folio_comm[$folio], @folio_stack[$folio]);
        return;
    }

    // First time seeing this folio lock, so record it.
    @folio_ts[$folio] = nsecs;
    @folio_stack[$folio] = kstack;
    @folio_pid[$folio] = pid;
    @folio_tid[$folio] = tid;
    @folio_comm[$folio] = comm;
}

kprobe:folio_unlock
/@hugetlb_fault_entry && !strncmp(comm, "repr", 4)/
{
    // Clean up when the folio is unlocked.
    $folio = (uint64)arg0;
    delete(@folio_ts[$folio]);
    // ... delete other folio maps ...
}
```
This section mirrors the mutex logic but for the second lock type: *folios*. A folio is a structure in the kernel for managing physical pages.

- `kretprobe:filemap_get_entry`. This function is used to find and lock a page cache entry. The locked folio is returned by the function.

The remaining logic is identical to the mutex tracing.

### Part 4: Tracing log of deadlocks

```bash
repro_20250402_ pid:13228 tid:13250 is blocked for folio ffffea0000b28000

        __filemap_get_folio+111
        hugetlb_fault+2640
        handle_mm_fault+6210
        do_user_addr_fault+2766
        exc_page_fault+100
        asm_exc_page_fault+38

The folio has been locked by: ffffea0000b28000, pid: 13228, tid: 13229, comm: repro_20250402_, stack:
        __filemap_get_folio+111
        hugetlb_fault+2640
        handle_mm_fault+6210
        do_user_addr_fault+1565
        exc_page_fault+100
        asm_exc_page_fault+38
        __put_user_4+13
        copy_process+8010
        kernel_clone+528
        __x64_sys_clone+397
        do_syscall_64+106
        entry_SYSCALL_64_after_hwframe+118

repro_20250402_ pid:13228 tid:13229 is blocked for mutex ffff888000fec7e0

        __mutex_lock+5
        hugetlb_wp+3976
        hugetlb_fault+5320
        handle_mm_fault+6210
        do_user_addr_fault+1565
        exc_page_fault+100
        asm_exc_page_fault+38
        __put_user_4+13
        copy_process+8010
        kernel_clone+528
        __x64_sys_clone+397
        do_syscall_64+106
        entry_SYSCALL_64_after_hwframe+118

The mutex has been locked by: ffff888000fec7e0, pid: 13228, tid: 13250, comm: repro_20250402_, stack:
        __mutex_lock+5
        hugetlb_fault+952
        handle_mm_fault+6210
        do_user_addr_fault+2766
        exc_page_fault+100
        asm_exc_page_fault+38
```

### DEPT

I would like to call out Byungchul Park for his kind help with the DEPT lock detection mechanism. [DEPT](https://lore.kernel.org/linux-mm/20250514064729.GA17622@system.software.com/) is a modern lock detection tool that automatically detects lock dependencies and potential deadlocks at runtime, which helps a lot confirming the deadlock between the folio lock and the mutex lock.

## Patch to fix the deadlock in hugetlb subsystem

### [[PATCH] mm/hugetlb: fix a deadlock with pagecache_folio and hugetlb_fault_mutex_table](https://lore.kernel.org/linux-mm/20250513093448.592150-1-gavinguo@igalia.com/)

## A take-home Lesson
To further explore the power of bpftrace, I would recommend playing around with different probes. Specifically, you can try to trace the `readline` function in the bash shell to see how it works. The following example shows how to monitor the input from the bash shell.

To do the experiment, you would like to open two terminals in parallel, one for the bash shell and one for the bpftrace script. Then, you can start typing in the terminal. The bpftrace script will monitor the input from the bash shell and print out the input.

```bash
$ sudo apt update; sudo apt install bash-dbgsym
$ sudo bpftrace -e 'uretprobe:/bin/bash:readline { printf("readline: \"%s\"\n", str(retval)); }'
Attaching 1 probe...
readline: ";;"
readline: "ll"
readline: "lscpu"
readline: "sudo bpftrace -lv 'uretprobe:/bin/bash"
readline: "sudo bpftrace -lv 'uretprobe:/bin/bash'"
readline: "sudo bpftrace -lv uprobe:/bin/bash"
readline: "sudo bpftrace -lv 'uprobe:/bin/bash'"
readline: "sudo bpftrace -lv 'uprobe:/bin/bash:readline'"
readline: "sudo bpftrace -lv 'uretprobe:/bin/bash:readline'"

```

For further information, please refer to [A Brief Introduction to bpftrace by Louis McCormack](https://louismccormack.com/introduction-to-bpftrace)