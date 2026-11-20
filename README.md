# INFO: task hung in nilfs_transaction_begin (fs/nilfs2/segment.c:221, v7.4-rc3)

Syzkaller-found NILFS2 hang on Linux v7.4-rc3, reproduced and
independently verified by the KernSight state-primed pipeline on fresh
VMs.

## Summary

A NILFS2 writer blocks for more than 140 seconds acquiring the segment
constructor semaphore for reading:

```text
INFO: task syz.5.2623:50371 blocked for more than 143 seconds.
task:syz.5.2623 state:D
Call Trace:
 rwsem_down_read_slowpath
 __down_read
 down_read
 nilfs_transaction_begin+0x323/0xa00 fs/nilfs2/segment.c:221
 nilfs_write_begin+0x78/0x1b0 fs/nilfs2/inode.c:228
 generic_perform_write
 vfs_write
 ksys_write
```

A second blocked task sits inside the construction path itself
(`nilfs_ioctl` -> `nilfs_segctor_construct` ->
`nilfs_segctor_do_construct+0x16ef/0x74f0 fs/nilfs2/segment.c:2139`,
which is the `nilfs_segctor_assign()` step).

The lock dump identifies who holds the semaphore:

```text
locks held by syz.7.24/13190: 3:
 #0: (sb_writers#17)                 at: nilfs_ioctl_clean_segments
                                         fs/nilfs2/ioctl.c:869
 #1: (&nilfs->ns_segctor_sem) WRITE  at: nilfs_transaction_lock
                                         fs/nilfs2/segment.c:357
 #2: (&bmap->b_sem)                  at: nilfs_bmap_assign
                                         fs/nilfs2/bmap.c:373
```

So a `NILFS_IOCTL_CLEAN_SEGMENTS` task is inside the segment-constructor
path, holding `ns_segctor_sem` for writing while doing bmap assignment,
and normal writers cannot acquire the semaphore for reading.  The
background `segctord` thread is alive in this report (it is not one of
the blocked tasks).

## Relation to the previously reported nilfs_segctor_thread hang

A related report was filed earlier for the same tree and config:

  INFO: task hung in nilfs_segctor_thread (fs/nilfs2/segment.c:357)
  https://github.com/Zheng-zheng-codes/nilfs2-segctor-thread-hang

The two reports differ in call chain and blocking point:

| | segctor_thread hang | this report |
|---|---|---|
| blocked task | `segctord` kthread | normal writer(s) via `nilfs_write_begin` |
| blocking site | `down_write(ns_segctor_sem)` in `nilfs_transaction_lock()` (segment.c:357) | `down_read(ns_segctor_sem)` in `nilfs_transaction_begin()` (segment.c:221) |
| holder seen in dump | task in `nilfs_ioctl_clean_segments()` holding `sb_writers` | task in `nilfs_ioctl_clean_segments()` holding `ns_segctor_sem` (write) + `bmap->b_sem` at `nilfs_bmap_assign()` |

Both are two opposite sides of the same `ns_segctor_sem` starving behind
an extended hold by the clean-segments ioctl path.  We therefore suspect
a shared root cause (the `NILFS_IOCTL_CLEAN_SEGMENTS` path holding the
semaphore for too long, e.g. across the bmap assignment / segment
cleaning work; the earlier report's serial log showed repeated
`NILFS (loop6): error -28 cleaning segments`), while the two hang
reports themselves are separate symptoms and are filed as such.

C reproducer: not generated; the syzkaller recipe below is the primary
reproducer.

## Reproduction

  git clone https://github.com/Zheng-zheng-codes/nilfs2-transaction-begin-hang
  cd nilfs2-transaction-begin-hang

  # boot v7.4-rc3 with the supplied kernel config (Debian trixie x86_64)
  # in the guest, as root:
  syz-execprog -repeat=1 reproducer/repro.kernsight.prog

`repro.kernsight.prog` is a self-contained state-reconstruction recipe:
72 replay programs plus one model-selected state-construction program,
followed by the crashing testcase, in execution order.  Equivalent
step-by-step form: run `reproducer/repro.priming.prog` first, then
`reproducer/repro.prog` on the same VM (see
`reproducer/repro.README.txt`).

The crash depends on kernel state accumulated by the recipe; running the
testcase alone is not sufficient.

## Verification

External unlimited-budget reproduction of the pipeline crash
(b938319ea34db0b459c1fd8e695defec4dc1c8fc):

- pipeline (20-minute budget): triggered on 11-19/11-20, never verified
- external run, attempt 1: `confirmed=true`, `status=verified`,
  72 replay + 1 priming program, 2026-11-20T07:17:00Z
- external run, attempt 3: `confirmed=true`, `status=verified`,
  72 replay + 1 priming program, 2026-11-20T08:05:31Z
- independent fresh-VM verification inside the external run:
  **2/2 target hits** (2026-11-20T07:46:46Z, 07:50:15Z)
- evidence: `reproducer/repro.outcomes.jsonl`, `logs/external-manager-session.log`

## Kernel / guest details

  kernel: v7.4-rc3, bzImage sha256
    0c16409f8aec508ac44a63e631c2bd352536401f3fd0856df1d839fb953a1882
  guest: Debian trixie x86_64, 6 GiB RAM, 4 vCPU, QEMU/KVM
  boot: oops=panic panic_on_warn=1 hung_task_panic=0
  config: CONFIG_KASAN=y, CONFIG_PROVE_LOCKING=y, CONFIG_NILFS2_FS=y
  syzkaller: commit 853b104bb454141a5b0fc0e4218ec78e9f34ffb1

See `environment.txt` for the full QEMU command line and image hashes.
