---
title: "XFS shut down with -ENOSPC and 862 GiB free. The reservation was never initialised."
date: 2026-08-08T17:12:00-06:00
draft: false
description: "Six XFS shutdowns, six falsified hypotheses, and a filesystem that took itself down for lack of space on a half-empty disk. The evidence was destroyed by the crash that produced it, until I mapped a trace buffer across the reboot. The bug is one missing assignment in xfs_parent_da_args_init()."
ShowToc: true
ShowReadingTime: true
tags:
  - "linux"
  - "kernel"
  - "xfs"
  - "filesystem"
  - "debugging"
  - "ftrace"
  - "kdump"
  - "yocto"
  - "bitbake"
  - "arch-linux"
categories:
  - "linux"
  - "systems"
outputs:
  - "HTML"
---

`XFS (nvme0n1p2): Internal error at line 721 of file fs/xfs/libxfs/xfs_defer.c`

862 GiB free. A 2 TB NVMe drive at 53% capacity. The filesystem shut itself
down because it could not allocate one block.

## TL;DR

`xfs_parent_da_args_init()` never sets `args->total`. The field stays zero,
`xfs_da_grow_inode_int()` subtracts from it, and it underflows to
`0xFFFFFFFF`. Every downstream guard that should catch an impossible
allocation is defeated by a single `(int)` cast, so `maxlen` gets clamped to
zero and the allocator rejects its own arguments with a bare `-ENOSPC`.
`xfs_defer_finish_noroll()` treats that as fatal and takes the filesystem
down.

It needs parent pointers (`parent=1`) plus an inode with many hard links,
which is why a Yocto `do_package` run reproduces it and nothing else on this
machine does.

The fix is one line. It reproduces deterministically in about 0.2 seconds on a
512 MiB scratch filesystem, there is an xfstests case that fails without the
fix and passes with it, and the series has been
[applied to for-next](https://lore.kernel.org/linux-xfs/20260810230611.2859909-8-floss@jetm.me/)
by XFS maintainer Carlos Maiolino.

{{< update date="September 3, 2026" >}}
Carlos Maiolino applied the full v3 series to for-next. The fix landed as
`47531ec00c81` (`xfs: initialise args->total for parent pointer updates`).
See the
[applied series on lore](https://lore.kernel.org/linux-xfs/20260810230611.2859909-8-floss@jetm.me/)
and the commit list in References.
{{< /update >}}

## The same thing without the jargon

If you do not carry XFS internals around in your head, here is the shape of it.
The rest of the post assumes you have read this much and no more.

First, the error message is a red herring. The kernel log says "Corruption of
in-memory data", and nothing was corrupt. That string is a generic label
attached to one exit path, not a diagnosis, and it cost me half a day of
chasing memory faults that were never there.

What actually happens: there is a counter meaning "how many blocks may this
operation still use." One path forgot to set it before using it, so it started
at zero. The code then subtracted from it.

The path in question writes a parent pointer, which is the record that a file
belongs to a directory. One gets written on every hardlink, which is why a
packaging step that hardlinks the same file into dozens of staging directories
is what found this.

Subtracting one from zero in that counter does not give minus one. It is
unsigned, so it rolls over to the largest value it can hold, about 4.3 billion,
the way an odometer wound backwards past zero reads 999999 rather than -1.

Everything after that behaved correctly and arrived somewhere absurd. A safety
check read 4.3 billion, concluded there was effectively unlimited room, and
waved the request through. The next layer did its own arithmetic on the same
poisoned value and decided the operation could use **zero** blocks. Then a
third check compared "needs at least one block" against "may use zero blocks",
found that impossible, and returned "out of disk space."

Nothing was out of disk space. The disk was half empty. The filesystem could
not distinguish "this request is impossible because the disk is full" from
"this request is impossible because I computed nonsense," so it assumed its own
in-memory state was corrupt and shut down to avoid writing damage to disk. That
is the correct thing to do when your bookkeeping contradicts itself. It was
just reasoning from one bad number.

Two assertions exist in the source specifically to catch this, and both are
compiled out of every kernel anyone actually runs.

## The machine

| | |
|---|---|
| Board | ASUS ROG Crosshair X870E Hero |
| CPU | AMD Ryzen 9 9950X, 16 cores, 32 threads |
| Memory | 96 GB |
| Root | XFS on NVMe, `crc=1 reflink=1 rmapbt=1 parent=1` |
| Kernel | 7.1.5, CachyOS |
| Workload | BitBake building a full image with Chromium |

Two weeks of crashes. The first week went into BIOS settings and hardware
swaps, because a machine that hard-reboots under load looks like a hardware
problem and I treated it as one. Fourteen crashes produced a saved dump; the
earliest ones happened before I had kdump armed at all. Six carried the XFS
shutdown signature.

Always under a heavy build. Never at idle, never under synthetic I/O load,
never under `fio`.

## Phase 1: everything that was not the problem

I want to lead with the dead ends, because they took most of the five days
and because the shape of the error sent me somewhere wrong.

`-ENOSPC` with 862 GiB free reads like an accounting bug. So I went looking
for one.

**Reservation sizing.** I worked through four candidate paths where an
attribute-fork operation could under-reserve. All four compute their
reservations correctly. More decisively, an over-run reservation does not
produce `-ENOSPC` at all: it trips `xfs_trans_mod_sb()` at
`fs/xfs/xfs_trans.c:350-351` with a different failure entirely. The errno
was telling me it was not this, and I did not listen for two days.

**Reflink and extent churn.** The filesystem has `reflink=1`, and BitBake's
`do_package` hardlinks aggressively. A plausible story about extent-count
explosion. Falsified: the reproducer triggers the crash with reflink traffic
absent.

**Pre-existing extended attributes.** My best hypothesis for a while was
that files arriving with user xattrs pushed the attribute fork over a
threshold. I built a `--no-xattr` control arm into the reproducer and it
crashed anyway. I read the argv back out of the vmcore to be certain the
flag was actually in effect. It was. Hypothesis dead.

**Memory pressure.** The shutdown reason is `SHUTDOWN_CORRUPT_INCORE`,
which prints as "Corruption of in-memory data". That phrasing cost me half
a day. It is the generic `out_shutdown:` label at `xfs_defer.c:721` and it
implies nothing whatsoever about memory. PSI counters showed `mem_full` at
0.00% and zero `pswpout` across the entire build.

**Allocation-group locking order.** XFS refuses allocations that would
violate AG lock ordering, and it reports that refusal as `-ENOSPC` too. A
real candidate. Ruled out by tracing: the terminal event was never
`xfs_alloc_vextent_skip_deadlock`.

**Rogue DMA.** This one I tested rather than argued. The box boots with
`iommu=pt`, which means no DMA translation for most devices, which means a
misbehaving device could scribble on kernel memory with no fault and no log
line. I rebooted with `iommu.strict=1` to force full translation and ran the
reproducer again. `IO_PAGE_FAULT` count: 0. The crash reproduced identically.
Not DMA.

Six hypotheses, six refutations, and I still had no idea what was happening.

## Phase 2: the crash destroys its own evidence

The obvious move is to trace the allocator and read the trace after the
crash. That does not work here, and the reasons are worth writing down.

The shutdown is configured to panic (`fs.xfs.panic_mask=16`) so kdump fires
and writes a vmcore. But a vmcore is a snapshot of memory at the moment of
death, and what I needed was the sequence of allocator decisions leading up
to it. That lives in the ftrace ring buffer.

`ftrace_dump_on_oops` is the standard answer: on an oops, flush the trace
ring into the kernel log, where `vmcore-dmesg` will capture it. It failed
twice, in two different ways.

**Mode 2 dumped the wrong CPU.** `ftrace_dump_on_oops=2` dumps only the
crashing CPU's buffer. I reasoned that the failing allocation runs in the
same task, therefore the same CPU. That conflates task with CPU. Under
`PREEMPT_DYNAMIC=full` with 16 workers and a build across 32 threads, the
task migrates mid-syscall. The capture held CPU 10, and the crashing task
had exactly one event in it: the shutdown itself. 1,133 trace lines, none
of them the ones I needed.

**Mode 1 blocked kdump entirely.** Dumping all 32 CPUs produces roughly
2.1 MB of text. `ftrace_dump_on_oops` runs from the `DIE_OOPS` notifier,
which fires *before* `crash_kexec()`. Every line goes through `printk`, and
`printk` paints the framebuffer console. At this resolution that is a few
hundred lines per second. The machine sat there painting a trace dump
instead of writing a vmcore, and I lost the crash.

I also learned that the trace volume itself was the problem. I had enabled
the entire `xfs_alloc_*` family; the cursor-walk events accounted for 95% of
the output. And `xfs_buf_ioerror`, despite the name, is not a failure-path
tracepoint: XFS calls it with `error 0` from `xfs_buf_get_map()` on
essentially every buffer lookup. I measured 402,082 ring overruns on one CPU
in 15 seconds. Unfiltered, it evicted the window I cared about before the
failure ever happened.

## Phase 3: a ring buffer that survives the reboot

The fix for all of this is a kernel feature I had not used before:
persistent, boot-mapped trace buffers.

```text
reserve_mem=64M:4096:trace trace_instance=boot_map@trace
```

This reserves 64 MB at a fixed physical address and backs a named trace
instance with it. The contents survive kexec and a full reboot. Nothing
needs to be flushed through `printk`, nothing competes with the panic path,
and kdump is untouched.

One sharp edge: the *contents* survive, the *configuration* does not. After
every boot the instance exists and holds the previous run's data, but no
events are enabled. The arming script has to run again, and it must not
clear the buffer before I have read it. I put a guard in mine after nearly
wiping the only good capture I ever got.

That guard had its own bug worth mentioning, because it is a classic:

```bash
retained=$(grep -cve '^#' "$I/trace" || echo 0)
```

`grep -c` exits 1 when the count is zero, so on an empty buffer the command
substitution produced `"0\n0"` and the numeric comparison broke. The guard
protecting my evidence was itself unable to count. `|| true` plus an
explicit default fixed it.

With the persistent instance armed and the event list trimmed to terminal
events only, I ran the build and the reproducer together and waited.

## Phase 4: four lines

```text
531.418881: xfs_alloc_vextent_loopfailed: agno=0x1 minlen=0x1 maxlen=0x1
531.435265: xfs_alloc_size_nominleft:     agno=0x2 minlen=0x1 maxlen=0x0 total=0xffffffff (-1)
531.435266: xfs_alloc_vextent_badargs:    agno=-1  minlen=0x1 maxlen=0x0 total=0x1
531.440550: xfs_force_shutdown: ptag=0x10 flags=0x8 fname=fs/xfs/libxfs/xfs_defer.c line_num=721
```

`total=0xffffffff`, 5.3 milliseconds before the shutdown, in the crashing
task. `maxlen` goes from 1 to 0 between the first and second line.

That is the whole bug, and the rest is reading source.

## The mechanism

`xfs_parent_da_args_init()` at `fs/xfs/libxfs/xfs_parent.c:151-171` fills in
every field of the `xfs_da_args` it builds. Every field except `total`. The
struct is zeroed, so `total` starts at 0.

`xfs_da_grow_inode_int()` then treats it as a running remainder at
`fs/xfs/libxfs/xfs_da_btree.c:2357`:

```c
args->total -= dp->i_nblocks - nblks;
```

`args->total` is `xfs_extlen_t`, which is unsigned. Subtracting from zero
underflows it to `0xFFFFFFFF`.

Now the interesting part, in `xfs_alloc_space_available()` at
`fs/xfs/libxfs/xfs_alloc.c:2523-2535`:

```c
available = (int)(pagf_freeblks + agflcount - reservation
                  - min_free - args->minleft);

if (available < (int)max(args->total, alloc_len))
        return false;

if (available < (int)args->maxlen && !(flags & XFS_ALLOC_FLAG_CHECK)) {
        args->maxlen = available;
        ASSERT(args->maxlen > 0);
        ASSERT(args->maxlen >= args->minlen);
}
```

`max(args->total, alloc_len)` is `0xFFFFFFFF`. Cast to `int`, that is `-1`.
The guard becomes `available < -1`, which cannot be true for any
non-negative `available`. **A gate designed to reject allocations that lack
free space can no longer reject anything.**

Execution falls into the clamp, `args->maxlen` is assigned `available`, and
when `available` is 0 the result is `maxlen = 0`. The two `ASSERT` calls
that exist precisely to catch this are compiled out: `CONFIG_XFS_DEBUG` is
not set on any distribution kernel.

`xfs_alloc_vextent_check_args()` then does its job correctly. It sees
`minlen=1 > maxlen=0`, an impossible request, and returns `-ENOSPC` with a
tracepoint and no message. `xfs_defer_finish_noroll()` treats any non-`EAGAIN`
error as fatal, calls `xfs_force_shutdown(SHUTDOWN_CORRUPT_INCORE)`, and the
filesystem is gone.

Every layer behaved as written. The only defect is the missing assignment.

## Why this needs parent pointers and hard links

`parent=1` stores one parent-pointer xattr per directory entry, so an inode
with N hard links carries N of them. I confirmed this directly: 31 links
produced 31 parent pointers.

BitBake's `do_package` hardlinks the same file into many package staging
directories. Each `link()` writes another parent pointer, the attribute fork
grows past the point where it needs a leaf-to-node conversion, and
`xfs_da_grow_inode_int()` runs on a `xfs_da_args` that never had `total`
set. Without parent pointers there is no such xattr traffic. Without many
links there is no fork growth. That is why this box crashes and my other
machines do not.

## The fix

```c
args->total = xfs_attr_calc_size(args, &local);
```

One line, in `xfs_parent_da_args_init()`.

The reason to write it exactly that way, rather than clamping the subtraction
or picking a safe constant, is that XFS already does this correctly somewhere
else. When the filesystem replays a parent-pointer insert from the log after a
crash, `xfs_attri_recover_work()` sets the same field from the same function
(`xfs_attr_item.c:706`). So replaying the operation from the log was always
right, and performing it live was always wrong. The fix makes the normal path
behave like the recovery path instead of contradicting it, which is a much
easier thing to defend than "I added a missing line."

I argued to myself that this is both necessary and sufficient before
building anything, because "the patch makes the symptom go away" is a weak
claim. `maxlen = 0` requires the clamp to execute with `available == 0`,
which requires the gate above it to pass. With a correct `total` of roughly
25 blocks and `alloc_len` of 1, the gate demands `available >= 25`, so any
`available` small enough to clamp `maxlen` to zero would have failed the
gate first. The failing state is unreachable. With `total = 0xFFFFFFFF` the
gate compares against `-1` and always passes.

I carry four more patches alongside it, and only one of them is a fix: a
genuinely separate bug where `xfs_defer_finish_one()` returns an
uninitialised `error` on the item-less barrier path, reachable through
`xfs_defer_add_barrier()` on any kernel built with
`CONFIG_XFS_ONLINE_REPAIR`. The other three are diagnostics, including one
that prints the actual errno at the shutdown site so the next person does
not have to do any of this.

## Validation: making the bug fire on demand

For most of this investigation the only way to see the failure was to run a
Yocto build and a hardlink stressor next to three desktop applications and wait
five to seven minutes for the machine to die. That is a terrible experiment. It
takes a reboot per data point, it destroys the evidence that explains it, and
the moment you ask "how many failures did the patched kernel avoid?" you find
you never measured a baseline rate to compare against.

The way out was to stop treating the failure as stochastic. Reading the source,
it needs exactly two things to happen at once:

**N1**, two `xfs_da_grow_inode()` calls sharing one `xfs_da_args`. That is the
`XFS_DAS_LEAF_ADD` -> `xfs_attr3_leaf_to_node()` -> `XFS_DAS_NODE_ADD` ->
`xfs_da3_split()` path, and enough long-named hardlinks on one inode produce it
reliably.

**N2**, at the second of those calls, an allocation group where

```text
available = pagf_freeblks + agflcount - reservation - min_free - args->minleft
```

is **exactly zero**. Not negative, not one: at `available == -1` the same code
assigns `args->maxlen = 0xFFFFFFFF` and takes a different path entirely, and at
`available >= 1` the allocation just succeeds.

My stressor supplied N1 all day long and never once supplied N2. That is the
whole reason it could not crash anything on its own. On a 1.9 TB filesystem with
862 GiB free, waiting for one of 32 allocation groups to sit at exactly zero is
a coincidence you wait hours for, and the BitBake build and the browsers were
only ever churning allocation until it happened.

So construct N2 instead of waiting for it. Build a 512 MiB filesystem with two
allocation groups and parent pointers, fill it to within a few hundred blocks,
then walk that margin down one block at a time, hardlinking at every step:

```text
mkfs.xfs -f -m crc=1 -n parent=1 -d agcount=2 /var/tmp/pptr.img
fallocate -l 411M /mnt/scratch/ballast      # free blocks now 464
# punch one 4 KiB hole per step, 64 hardlinks per step
```

The whole free pool is now a few hundred blocks, so the sweep passes through
zero within a step or two rather than never.

It fires on the second step. Roughly 0.2 seconds of link activity:

```text
xfs_alloc_cur_lookup       minlen 1 maxlen 0 ... total 4294967295
xfs_alloc_cur_right        minlen 1 maxlen 0 ... total 4294967295
xfs_alloc_size_neither     minlen 1 maxlen 0 ... total 4294967295
xfs_alloc_vextent_first_ag minlen 1 maxlen 0 ... total 1
xfs_alloc_vextent_badargs  minlen 1 maxlen 0 ... total 1

XFS (loop0): Corruption of in-memory data (0x8) detected at
xfs_defer_finish_noroll+0x29a/0x4b0 (fs/xfs/libxfs/xfs_defer.c:721)
```

Read those five lines in order and the whole bug is there. `maxlen` is already
clamped to zero while `total` still reads `0xFFFFFFFF`. Then `total` becomes 1,
because `xfs_bmap_btalloc_low_space()` sets `args->total = ap->minlen` for a
last-ditch attempt across every allocation group. But **nothing resets
`args->maxlen`**, so the poisoned zero survives into
`xfs_alloc_vextent_check_args()`, which rejects `minlen(1) > maxlen(0)` and
returns a bare `-ENOSPC`. The fallback that exists to rescue a too-large
reservation cannot rescue a clamped `maxlen`.

There is also a detail worth pausing on. Earlier in the same capture, several
hundred milliseconds before the shutdown, another `ln` carried
`total 4294967295` with `maxlen 1` and **allocated successfully**. The underflow
is not rare and is usually harmless. It only kills you when it lands on an
allocation group sitting at exactly zero.

### The A/B

Same machine, same script, same geometry, only the kernel differs:

| | Unpatched | Patched |
|---|---|---|
| Outcome | **shutdown at step 1** | **400 steps, 25,024 links, clean** |
| `total 4294967295` | 8 | **0** |
| Attr fork | `0` -> `4294967295` | `25` -> `24` |
| Data fork (control) | `78` -> `77` | `78` -> `77` |

The attr-fork row is the fix. `xfs_attr_calc_size()` returns
`XFS_DAENTER_SPACE_RES(mp, XFS_ATTR_FORK)`, which for a parent pointer is a
constant 25 on a 4 KiB-block filesystem, and the 24 is that same
`xfs_da_args` at its second growth after one block was subtracted. Every one of
those events is a trip through the exact code that used to wrap, and none of
them wrapped.

The data-fork row is why I believe the comparison. `78 -> 77` is the identical
subtraction happening correctly, and it is unchanged across both kernels. The
workload was the same; the patch moved only the population it aimed at.

I also ran the same test on a second, headless machine on its unpatched kernel.
It produced a byte-identical event distribution: 52 events at `total 0`, 30 at
`78`, 8 at `77`, 8 at `4294967295`, 2 at `1`. Two different CPUs, two different
boards, the same counts. Whatever this is, it is not timing-dependent.

That headless result also killed a hypothesis I had held for a week. I thought
the desktop applications contributed something causal, because the stressor
alone never reproduced anything. They did not. They were one way of producing
N2, and a nearly-full 512 MiB image is a better one.

### A regression test that fails without the fix

The reproducer above is a shell script that lives on my machine. The version
that belongs upstream is an xfstests case, and writing one turned out to be
mostly a matter of stating the two conditions in the harness's own vocabulary:

```bash
_scratch_mkfs_sized $((512 * 1024 * 1024))    # two AGs, parent=1
# ballast to leave ~460 free blocks
# per step: punch one 4 KiB hole, then 64 long-named hardlinks
```

It needs no explicit assertion. xfstests runs `_check_dmesg` and
`_check_xfs_filesystem` after every test, so a shutdown fails it automatically.

The part that matters is that I ran it on both kernels:

| kernel | result |
|---|---|
| unpatched | **fails** - dirty log, filesystem inconsistent, `xfs_defer.c:721` in dmesg, dead by step 2 |
| patched | **passes** - 400 steps, 25,024 links, 25 seconds |

Until that first row existed the test was worthless. A regression test that has
only ever been observed passing tells you nothing, and I nearly shipped one:
my first version left 128 free blocks and swept up to 192, entirely below the
band where the bug fires. It passed on a patched kernel for the wrong reason.

One trap worth repeating, because it will cost someone a machine. If
`fs.xfs.panic_mask` is non-zero, the shutdown becomes a `BUG()`, kdump fires,
and the host reboots instead of reporting a failure. XFS defaults to 0. Mine
was set to 16 for crash capture, which is exactly the configuration a person
debugging this bug would have.

### Regression coverage

`./check -g parent -g attr` on the patched kernel: 53 of 55 passing. The two
failures are environmental, a `setfattr` deprecation warning newer than the
golden output and an `O_TMPFILE` `EOVERFLOW` inside the test harness. Neither
mentions parent pointers, allocation or ENOSPC, and no XFS shutdown appeared in
`dmesg` for the duration. I have not run those two on an unpatched kernel, so
"environmental" is an inference from what they say, not a demonstration.

### Eight days

The patched kernel has since carried ordinary Yocto build load for eight days
with no filesystem shutdown and no crash dump. The longest single uptime in
that window was three days. Before the patch the same machine died within five
to seven minutes once the conditions coincided.

I am wary of that number carrying more weight than it deserves, which is why it
comes last. Eight days of not-crashing is consistent with a fix and also
consistent with luck; the argument that does the work is still the one in the
table above, where the underflow is present 8 times on one kernel and 0 times
on the other under an identical script.

Nineteen tests did not run. Several skipped for a reason that is its own small
lesson: `xfs/433` and `xfs/532` report `[not run] requires CONFIG_XFS_DEBUG`.
The error-injection machinery that would have been the obvious tool for this
whole investigation is compiled out of every distribution kernel, for the same
reason the two `ASSERT`s guarding this bug are.

The series went to `linux-xfs` on 2026-08-08 as `[PATCH 0/5]`, grew a sixth
patch in review, and went out as
[`[PATCH v3 0/6] xfs: fix filesystem shutdown from parent pointer reservation
underflow`](https://lore.kernel.org/linux-xfs/20260810230611.2859909-8-floss@jetm.me/)
on 2026-08-10. Six patches: the fix (`args->total` in
`xfs_parent_da_args_init()`), an unrelated uninitialised-`error` bug found
while reading the same code, an assert that catches this class of failure at
the point it happens instead of three layers downstream, a naming cleanup,
a comment correction, and a diagnostic that prints the real errno at the
shutdown site.

Carlos Maiolino applied all six to for-next on 2026-09-03. The fix landed as
[`47531ec00c81`](https://lore.kernel.org/linux-xfs/20260810230611.2859909-8-floss@jetm.me/)
(`xfs: initialise args->total for parent pointer updates`); the full commit
list is in the References section below.
## A variant I have not proven

While reading the gate I noticed a second path through it that is worse than
the one I hit.

If `available` evaluates to `-1` rather than 0, the gate still passes, and
the clamp assigns `args->maxlen = (xfs_extlen_t)-1`, which is `0xFFFFFFFF`.
That does *not* trip `xfs_alloc_vextent_check_args()`, because `minlen` is
no longer greater than `maxlen`. Instead of a loud shutdown, a request for a
colossal extent proceeds.

I have not reproduced this and I am not claiming it happens. If it does, it
would move the missing assignment from "causes a shutdown" to "can corrupt
data", which is a materially different bug.

## Still unexplained

The root cause accounts for the shutdowns. It does not account for
everything I saw.

Three of the six crashes had `systemd-journal` segfaulting with `error 15`
(a reserved-bit page-table fault), the faulting address equal to the
instruction pointer, and a zeroed code page at the same `0x7e0` offset each
time. Those segfaults *precede* the filesystem shutdown, so they are not
downstream of it. One crash showed an AIL inconsistency. Two showed
`xfs_ialloc_read_agi()` returning `-5` with zero block-layer errors. On one
occasion the initramfs was corrupt at rest and I had to rebuild it from a
rescue USB.

The crash that produced the trace above had the clean XFS bug and none of
these. So the XFS bug happens without the corruption. Whether the corruption
happens without the XFS bug was the open question, and eight days on the
patched kernel have now answered it as far as I can:

```text
Jul 20 - Jul 30, unpatched   journald error 15 / RSVD faults: 2 (a floor;
                             several crashes destroyed their own logs)
Jul 30 - Aug 7,  patched     journald error 15 / RSVD faults: 0
```

Twelve segfaults did occur in that window, all of them `ld-linux` probe
failures from the build and one `virsh`. None carried `error 15`, a
reserved-bit fault, or journald.

So the likeliest reading is now that those faults were downstream of the
filesystem dying rather than a second, independent problem. I am not calling it
settled. The pre-patch rate was low enough that eight clean days is suggestive
rather than conclusive, and it does not explain the thing that made me suspect
independence in the first place: in dmesg the segfaults *preceded* the
shutdown. Either the ordering is misleading or there is still something here I
have not understood.

## Why you never see the panic

A smaller finding, but it wasted real time.

Every crash froze the display on the compositor's last frame and rebooted
minutes later with nothing on screen. `CONFIG_DRM_PANIC=y` is set,
`DRM_PANIC_SCREEN` is `kmsg`, and amdgpu wires up `get_scanout_buffer`. It
should draw.

It does not, because `drm_panic` registers as a kmsg dumper at
`drivers/gpu/drm/drm_panic.c:1041`, and kmsg dumpers run from inside
`panic()`. For a `BUG()` with kdump armed, `oops_end()` calls
`crash_kexec()` as its first action at `arch/x86/kernel/dumpstack.c:377` and
never returns. `panic()` is never reached, so the panic screen never draws.
Wayland holds DRM master, so the framebuffer console is not visible either.

`crash_kexec_post_notifiers=1` makes `kexec_should_crash()` return 0 at
`kernel/crash_core.c:97-98`, so `oops_end()` skips the early kexec, falls
through to `panic()`, draws the screen, and still writes the vmcore from
`__crash_kexec()` at `kernel/panic.c:695`. The cost is that notifiers now
run before the dump, so a hung notifier loses it. That is why it is not the
default.

## Open claims

1. **The missing `args->total` assignment is necessary and sufficient for
   this shutdown.** Falsified by reproducing an `xfs_alloc_vextent_badargs`
   with `minlen > maxlen` on a kernel carrying patch 5, or by a shutdown at
   `xfs_defer.c:721` with `-ENOSPC` where the trace shows a correctly
   initialised `total`.
2. **The trigger requires `parent=1` plus a high link count.** Falsified by
   reproducing the shutdown on a filesystem formatted without parent
   pointers, or with a workload that never exceeds a few links per inode.
3. **The `available == -1` variant can drive a `0xFFFFFFFF` allocation
   request.** Unproven. Confirmed by a trace showing `maxlen 4294967295`
   passing `check_args`; refuted by a source argument showing `available`
   cannot reach `-1` on this path.
4. **The journald reserved-bit faults were downstream of the filesystem
   shutdown, not independent of it.** I claimed the opposite for a fortnight.
   Revised on eight days of patched-kernel build load producing zero of them
   against a pre-patch floor of two. Falsified by a single `error 15`
   reserved-bit fault on a patched kernel, which would put independence back on
   the table; also weakened by anyone explaining why, in dmesg, the segfaults
   appear *before* the shutdown they are supposedly caused by.
5. **`available == 0` is a necessary condition, and the value is exactly
   zero rather than a range.** Supported by the reproducer firing within one
   or two single-block steps of the sweep and not before. Falsified by a
   trace showing `maxlen` clamped to zero while `available` was some other
   value.
6. **`xfs_attr_calc_size()` returns a constant 25 for every parent pointer on
   a 4 KiB-block filesystem, so the 24 in the patched traces is the second
   growth rather than a second reservation size.** Falsified by a
   parent-pointer allocation carrying any `total` other than 25 or 24 on a
   patched kernel, or by `m_bm_maxlevels[XFS_ATTR_FORK] != 5` here.
7. **The desktop applications contributed nothing causal.** They were one
   stochastic route to `available == 0`, and a nearly-full image is a
   deterministic one. I believed the opposite for a week. Falsified by a
   workload that reproduces the shutdown with an idle allocator, or by
   showing a browser or camera path that reaches XFS attribute forks.
8. **The two fstests failures are environmental rather than regressions.**
   Falsified if `generic/062` and `generic/633` pass on an unpatched kernel
   of the same version, since that would make the patch the only difference.
   I have not run that comparison; the argument so far is only that neither
   failure mentions parent pointers, allocation or ENOSPC.
9. **`tests/xfs/842` reproduces this bug and only this bug.** Falsified by the
   test failing on a kernel carrying patch 5, or by it passing on an unpatched
   kernel in a configuration where the standalone reproducer still fires.

## What I would do differently

Trust the errno. `-ENOSPC` on a half-empty filesystem is not an accounting
bug, it is an argument-validation bug, and the codebase says so: five of the
conditions in `xfs_alloc_vextent_check_args()` are impossible-argument
checks that report as `-ENOSPC` with nothing but a tracepoint. I spent two
days on space accounting because the errno had a familiar meaning.

Reach for persistent tracing sooner. I burned two crashes on
`ftrace_dump_on_oops` before switching to a boot-mapped instance that was
always going to be the right tool for a failure that destroys its own logs.

Distrust asserts in shipped kernels. Two `ASSERT` calls sit directly on this
bug and neither has ever executed on a distribution kernel, because
`CONFIG_XFS_DEBUG` is off everywhere it matters. An assert that only runs on
developer builds is documentation, not a guard.

## References

- [The v3 series on lore](https://lore.kernel.org/linux-xfs/20260810230611.2859909-8-floss@jetm.me/)
  - `[PATCH v3 0/6] xfs: fix filesystem shutdown from parent pointer
  reservation underflow`, posted 2026-08-10, applied to for-next 2026-09-03
- Applied commits (for-next):
  - `08e6a7587c0a` - xfs: initialise error in xfs_defer_finish_one()
  - `6c81238928c1` - xfs: give the deferred barrier op type a name
  - `17783da399db` - xfs: report the error that made deferred work shut down the fs
  - `410822bd69ef` - xfs: correct the parent pointer space reservation comment
  - `47531ec00c81` - xfs: initialise args->total for parent pointer updates
  - `21a0382da84f` - xfs: assert the reservation covers each da fork growth
- `fs/xfs/libxfs/xfs_parent.c` - `xfs_parent_da_args_init()`
- `fs/xfs/libxfs/xfs_da_btree.c:2357` - the underflowing subtraction
- `fs/xfs/libxfs/xfs_alloc.c:2523-2535` - `xfs_alloc_space_available()`
- `fs/xfs/libxfs/xfs_defer.c:721` - the shutdown site
- `arch/x86/kernel/dumpstack.c:377` - `oops_end()` calling `crash_kexec()`
- `kernel/crash_core.c:97-98` - `kexec_should_crash()`
- `drivers/gpu/drm/drm_panic.c:1041` - `drm_panic` kmsg dumper registration
- `Documentation/admin-guide/kernel-parameters.txt` - `reserve_mem`,
  `trace_instance`, `crash_kexec_post_notifiers`
