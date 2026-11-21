# WARNING in __kvm_gpc_refresh (virt/kvm/pfncache.c:267, v7.4-rc3)

Syzkaller-found warning on Linux v7.4-rc3, reproduced and independently
verified by the KernSight state-primed pipeline on a freshly replayed VM.

## Summary

The KVM guest page cache invariant "either gpa or uhva must be valid, but
not both" is violated while KVM emulates a Xen event-channel send:

```text
kvm_is_error_gpa(gpa) == kvm_is_error_hva(uhva)
WARNING: arch/x86/kvm/../../../virt/kvm/pfncache.c:267 at __kvm_gpc_refresh+0x1692/0x1e40
RIP: 0010:__kvm_gpc_refresh+0x1692/0x1e40 virt/kvm/pfncache.c:267
Call Trace:
 kvm_gpc_refresh+0xed/0x160 virt/kvm/pfncache.c:382
 kvm_xen_set_evtchn.part.0+0x1b3/0x280 arch/x86/kvm/xen.c:2014
 kvm_xen_set_evtchn arch/x86/kvm/xen.c:2115 [inline]
 kvm_xen_hvm_evtchn_send+0x22f/0x290 arch/x86/kvm/xen.c:2105
```

The code at the warning site:

```c
/* Either gpa or uhva must be valid, but not both */
if (WARN_ON_ONCE(kvm_is_error_gpa(gpa) == kvm_is_error_hva(uhva)))
        return -EINVAL;
```

`kvm_xen_set_evtchn()` retries `kvm_xen_set_evtchn_fast()`, and on
`-EWOULDBLOCK` calls `kvm_gpc_refresh(&kvm->arch.xen.shinfo_cache,
PAGE_SIZE)`, which re-refreshes the gpc with whatever gpa/uhva state the
cache currently holds.  In the failing run both representations were in
the same error state, so the refresh warned and returned `-EINVAL`.

The guest was fuzzing KVM via nested virtualization (`kvm-intel.nested=1`)
with Xen HVM emulation (shared-info cache + event-channel delivery
ioctls).  No memory corruption was reported; the observable effect is the
warning plus the failed event-channel write.

C reproducer: not generated; the syzkaller recipe is the primary
reproducer.

## Reproduction

  git clone https://github.com/Zheng-zheng-codes/kvm-gpc-refresh-warning
  cd kvm-gpc-refresh-warning

  # boot v7.4-rc3 with the supplied kernel config (Debian trixie x86_64)
  # in the guest, as root:
  syz-execprog -procs=1 -threaded -repeat=1 \
      -executor ./syz-executor reproducer/repro.kernsight.prog

The recipe is a state-reconstruction sequence (74 replay programs followed
by the crashing testcase).  The warning depends on the kernel state built
by the recipe; the testcase alone is not sufficient.
`-procs=8` did not reproduce the warning under our tests.

## Verification

- pipeline auto-repro: attempt 1 triggered but was not verified within
  the 20-minute budget; attempt 2 verified (`confirmed=true`,
  `status=verified`, fresh-VM recipe replay, 74 replay programs,
  2026-11-21T14:59:08Z)
- stability verification on fresh VMs: **5/5** reproductions, 120-121 s to
  the warning in every run (2026-11-21); see `external-verify/`

## Kernel / guest details

  kernel: v7.4-rc3, bzImage sha256
    0c16409f8aec508ac44a63e631c2bd352536401f3fd0856df1d839fb953a1882
  guest: Debian trixie x86_64, 6 GiB RAM, 4 vCPU, QEMU/KVM, nested KVM
  boot: oops=panic panic_on_warn=1 hung_task_panic=0 kvm-intel.nested=1
  config: CONFIG_KASAN=y, CONFIG_PROVE_LOCKING=y, CONFIG_KVM=y,
    CONFIG_KVM_XEN=y
  syzkaller: commit 853b104bb454141a5b0fc0e4218ec78e9f34ffb1

See `environment.txt` for the full QEMU command line and image hashes.
