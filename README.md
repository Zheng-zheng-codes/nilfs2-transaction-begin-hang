# INFO: task hung in nilfs_transaction_begin (fs/nilfs2/segment.c:221, v7.4-rc3)

Syzkaller-found NILFS2 hang on Linux v7.4-rc3, reproduced and
independently verified by the KernSight state-primed pipeline on fresh
VMs.

## Summary

A NILFS2 writer blocks for more than 140 seconds acquiring the segment
constructor semaphore for reading (reproducer/repro.report):

```text
INFO: task syz.7.24:13081 blocked for more than 142 seconds.
task:syz.7.24        state:D stack:23176 pid:13081 tgid:13075 ppid:10344
Call Trace:
 rwsem_down_read_slowpath
 down_read
 nilfs_transaction_begin+0x323/0xa00 fs/nilfs2/segment.c:221
 nilfs_write_begin+0x78/0x1b0 fs/nilfs2/inode.c:228
 generic_perform_write
 vfs_write
 ksys_write
```

A second thread of the same process (PID 13109) is blocked on the
`f_pos_lock` held by PID 13081.

An NMI backtrace of the semaphore holder (PID 13190) shows that it is
executing the NILFS2 segment-constructor path through
`nilfs_clean_segments()`, including btree/bmap propagation:

```text
 nilfs_segctor_apply_buffers fs/nilfs2/segment.c:1010
 nilfs_collect_file_data fs/nilfs2/segment.c:589
 nilfs_bmap_propagate fs/nilfs2/bmap.c:329
 nilfs_btree_propagate fs/nilfs2/btree.c:2128
```

The lock dump identifies the holder:

```text
locks held by syz.7.24/13190: 3:
 #0: (sb_writers#17)                at: nilfs_ioctl_clean_segments
                                        fs/nilfs2/ioctl.c:869
 #1: (&nilfs->ns_segctor_sem) WRITE at: nilfs_transaction_lock
                                        fs/nilfs2/segment.c:357
 #2: (&bmap->b_sem)                 at: nilfs_bmap_assign
                                        fs/nilfs2/bmap.c:373
```

So a `NILFS_IOCTL_CLEAN_SEGMENTS` task is inside the segment-constructor
path, holding `ns_segctor_sem` for writing while doing bmap assignment,
and normal writers cannot acquire the semaphore for reading.  The
background `segctord` thread is alive in this report (it is not one of
the blocked tasks).

## Relation to the earlier nilfs_segctor_thread hang

We previously reported a related NILFS2 hang on the same tree and
config:

  INFO: task hung in nilfs_segctor_thread (fs/nilfs2/segment.c:357)
  https://github.com/nilfs-dev/nilfs2/issues/182
  https://github.com/Zheng-zheng-codes/nilfs2-segctor-thread-hang

The two reports involve different blocked tasks:

- In the earlier report, the segctord kernel thread waits for
  `ns_segctor_sem` in write mode in `nilfs_transaction_lock()`.
- In this report, a normal file writer waits for the same semaphore in
  read mode in `nilfs_transaction_begin()`.

In both reports, the lock dump identifies a task executing
`NILFS_IOCTL_CLEAN_SEGMENTS` as the semaphore holder.  The external
verification logs of this report also repeatedly report:

  NILFS (loop0): error -28 cleaning segments

which resembles the repeated -ENOSPC errors observed in the earlier
report.  We suspect that both hangs may be manifestations of the same
GC error-handling problem, but the root cause has not yet been
established.

C reproducer: not generated; the syzkaller recipe below is the primary
reproducer.

## Reproduction

  git clone https://github.com/Zheng-zheng-codes/nilfs2-transaction-begin-hang
  cd nilfs2-transaction-begin-hang

  # boot v7.4-rc3 with the supplied kernel config (Debian trixie x86_64)
  # in the guest, as root:
  syz-execprog -procs=1 -threaded -repeat=1 \
      -executor ./syz-executor reproducer/repro.kernsight.prog

The recipe requires sequential execution with `-procs=1`.  Using
`-procs=8` did not reproduce the hang under our tests.

`repro.kernsight.prog` is a self-contained state-reconstruction recipe:
72 replay programs plus one model-selected state-construction program,
followed by the crashing testcase, in execution order.  Equivalent
step-by-step form: run `reproducer/repro.priming.prog` first, then
`reproducer/repro.prog` on the same VM (see
`reproducer/repro.README.txt`).

The crash depends on kernel state accumulated by the recipe; running the
testcase alone is not sufficient.

## Verification

The target was **independently reproduced five times**.  Each run booted a
fresh VM from the supplied image with a brand-new qcow2 overlay — no
snapshot reuse and no shared VM state — then executed the reproducer
sequentially (`-procs=1 -threaded -repeat=1`).  All five runs hit the hung-task dump:

| run | hit | time to trigger |
|---|---|---|
| run-1 | yes | 187 s |
| run-2 | yes | 187 s |
| run-3 | yes | 187 s |
| run-4 | yes | 188 s |
| run-5 | yes | 194 s |

See `external-verify/` for the per-run serial logs and harness logs.
`-procs=8` did not reproduce the hang under our tests; sequential
execution is required.

## Repository contents

```text
README.md               this document
environment.txt         kernel/guest hashes and the exact QEMU command line
kernel.config           kernel configuration used for the reproduction
description             original crash title
SHA256SUMS              checksums for every file listed here
logs/                   original crash evidence
  log0 — fuzzer log of the original crash
  machineInfo0 — VM/environment record
  report0 — crash report of the original crash
  title-stat — occurrence statistics
reproducer/             the reproducer and its provenance
  repro.kernsight.prog — the reproducer: 72 replay programs plus one model-selected priming program, followed by the crashing testcase (log format)
  repro.prog — the crashing testcase alone (not sufficient)
  repro.priming.prog — the replay/priming prefix
  repro.priming.json — priming trace: state distances and candidate picks
  repro.context.json — reproduction context (verification scope, primed flag)
  repro.achieved_state.json — measured 539-D state before the successful testcase
  repro.report — raw crash report of the reproduction
  repro.stats — phase/timing log of the reproduction
  repro.README.txt — original bundle description
external-verify/        five independent fresh-VM verification runs
  README.md             stability methodology and per-run results
  run-1 ... run-5/
    serial.log — full guest console of the run
    run.log — harness log with hashes and timings
    execprog-out.log — syz-execprog stdout
    HIT — marker written when the target appeared
```

## Kernel / guest details

  kernel: v7.4-rc3, bzImage sha256
    0c16409f8aec508ac44a63e631c2bd352536401f3fd0856df1d839fb953a1882
  guest: Debian trixie x86_64, 6 GiB RAM, 4 vCPU, QEMU/KVM
  boot: oops=panic panic_on_warn=1 hung_task_panic=0
  config: CONFIG_KASAN=y, CONFIG_PROVE_LOCKING=y, CONFIG_NILFS2_FS=y
  syzkaller: commit 853b104bb454141a5b0fc0e4218ec78e9f34ffb1

See `environment.txt` for the full QEMU command line and image hashes.
