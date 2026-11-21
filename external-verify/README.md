# External stability verification (2026-11-21)

Five independent runs of the shipped reproducer on fresh VMs (fresh qcow2
overlay per run, sequential replay).  Every run hit the target warning.

Command per run (in the guest, as root):

    syz-execprog -procs=1 -threaded -repeat=1 \
        -executor ./syz-executor reproducer/repro.kernsight.prog

Detection: the run counts as a hit when the raw serial log contains the
warning frame `__kvm_gpc_refresh+0x` (pfncache.c:267).

| run | hit | time to warning | serial |
|---|---|---|---|
| run-1 | yes | 120 s | run-1/serial.log |
| run-2 | yes | 121 s | run-2/serial.log |
| run-3 | yes | 121 s | run-3/serial.log |
| run-4 | yes | 121 s | run-4/serial.log |
| run-5 | yes | 121 s | run-5/serial.log |

Result: **5/5**, times 120-121 s — the most deterministic reproduction in
this collection.

Each `run-N/` contains `serial.log` (full guest console), `run.log` (harness
log with hashes and timings), `execprog-out.log` (syz-execprog stdout) and
the `HIT` marker.

Note: the reproducer must run with `-procs=1`; `-procs=8` did not reproduce
the warning under our tests.
