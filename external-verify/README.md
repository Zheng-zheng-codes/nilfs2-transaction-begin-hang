# External stability verification (2026-11-20)

Five independent runs of the shipped reproducer on fresh VMs (fresh qcow2
overlay per run, sequential recipe replay).  Every run hit the target hang.

Command per run (in the guest, as root):

    syz-execprog -procs=1 -threaded -repeat=1 \
        -executor ./syz-executor reproducer/repro.kernsight.prog

Detection: the run counts as a hit when the raw serial log contains the
blocking frame `nilfs_transaction_begin+0x` (the `down_read()` call site at
segment.c:221) together with an `INFO: task ... blocked for more than ...`
dump.  The `NILFS (loopN): error -28 cleaning segments` messages appear in
the runs as well, consistent with the clean-segments ioctl holder seen in
the lock dump.

| run | hit | time to first dump | serial |
|---|---|---|---|
| run-1 | yes | 187 s | run-1/serial.log |
| run-2 | yes | 187 s | run-2/serial.log |
| run-3 | yes | 187 s | run-3/serial.log |
| run-4 | yes | 188 s | run-4/serial.log |
| run-5 | yes | 194 s | run-5/serial.log |

Result: **5/5**, times 187-194 s.

Each `run-N/` contains `serial.log` (full guest console), `run.log` (harness
log with hashes and timings), `execprog-out.log` (syz-execprog stdout) and
the `HIT` marker.

Note: the same recipe run with `-procs=8` (concurrent) did not reproduce --
the crash needs the recipe's state-building sequence in order.  Use
`-procs=1` as above (this matches `syz-execprog -repeat=1 ...` since the
default is one process).
