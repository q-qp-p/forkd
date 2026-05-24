# v0.4 Phase 3 spike — Firecracker integration options

**Status:** SPIKE — exploring paths to integrate `WpBranch` into the
real `forkd-controller::branch_sandbox` path. No code yet, just
options analysis.

The library API (`WpBranch::begin/bulk_copy_clean/finalize`) and its
CLI surface (`forkd wp-bench`) are merged on main as of [#157]. The
remaining work is replacing v0.3.4's "FC writes memory.bin
synchronously inside the pause" with v0.4's "WP-arm in pause, FC
writes only vmstate, forkd captures memory async."

## The blocker

[Firecracker's `snapshot_memory_to_file`][fc-snapshot] does:

```rust
let mut file = OpenOptions::new()
    .write(true).create(true).truncate(false).open(mem_file_path)?;
file.set_len(expected_size)?;       // ← the killer
match snapshot_type {
    SnapshotType::Diff => guest_memory.dump_dirty(&mut file, &dirty_bitmap),
    SnapshotType::Full => guest_memory.dump(&mut file),
}
```

`set_len` on the destination file means `mem_file_path` must be a
regular file — character devices like `/dev/null` reject it. FC also
always writes memory contents (whole region for `Full`, dirty pages
for `Diff`) before returning. There is no `VmstateOnly` snapshot
type.

So for v0.4, we cannot just point FC's snapshot output at the void
and use WpBranch for memory. FC will write memory; the question is
where and how to make that write cheap.

## Options

### A. Patch Firecracker upstream — add `SnapshotType::VmstateOnly`

Add a new variant that skips the memory dump entirely. ~20 lines of
FC source. Sent to upstream as a PR, this is the right long-term
answer. But: FC's release cadence is monthly and breaking-change
review is slow; we'd be downstream-vendoring FC for at least a quarter
before this lands.

**Effort:** 1 day code + weeks/months for upstream merge.
**Risk:** Low (clean enhancement, no behavior change for existing users).

### B. Vendor a forkd-patched Firecracker

Same as A but maintain the patch in a fork in this repo. Avoids the
upstream wait. Cost: rebase the patch against FC `main` every release;
forkd's `scripts/build-rootfs.sh` and CI need to pull the fork's
artifact instead of the upstream binary.

**Effort:** 1 day initial + ongoing rebase tax.
**Risk:** Drift between forkd-FC and upstream FC features; users who
want to BYO their own FC binary lose the v0.4 path.

### C. Bypass FC's snapshot/create entirely

The vmstate that FC serializes is per-vCPU `kvm_regs` + `kvm_sregs` +
device state + kvmclock + TSC offset, all reachable via KVM ioctls on
the VM fd. If forkd-controller can get a handle to FC's VM fd, it
could pause + read vmstate + resume without going through FC's HTTP
API.

How forkd-controller gets FC's VM fd: not easy. FC owns the fd
internally; the only ways out are `ptrace` on the FC process or
having FC explicitly share the fd at startup. Neither is clean.

**Effort:** 1+ weeks (need to learn FC's internal vmstate format
to serialize it ourselves).
**Risk:** High — vmstate format is internal to FC, changes without
notice, and reproducing the on-disk format means restore-compatibility
becomes our problem.

### D. tmpfs + discard

Point `mem_file_path` at `/dev/shm/forkd-discarded-<id>.bin` so the
FC write goes to RAM at ~RAM throughput (~2 GB/s on commodity DDR),
then unlink immediately after. Forkd separately uses WpBranch to
capture memory into the real snapshot.

Cost: the FC write still happens *inside* the pause window. For
a 1 GiB parent, ~500 ms inside the critical section (tmpfs is faster
than ext4's 150 ms? Actually tmpfs is faster — but only when the
working set fits in RAM, and writing 1 GiB to tmpfs allocates 1 GiB
of RAM). Net pause is `arm_WP + tmpfs_write + small_overhead` ≈
500 ms + 3 ms. **Worse than v0.3.4** for the parent-pause metric.

Wait — there's a nuance. For `Diff` mode, FC's `dump_dirty` writes
only the dirty pages, not the whole region. If the source VM is
clean immediately before BRANCH (which the agent fan-out pattern
permits — we can flush dirty bitmap right before BRANCH), the FC
write is tiny: maybe a few hundred KB. Diff-mode tmpfs discard
could keep the pause to ~50 ms total. That's a 3× improvement over
v0.3.4 with no FC patch.

**Effort:** 1-2 days.
**Risk:** Medium — depends on dirty bitmap being small at BRANCH time.
For idle parents this is fine; for actively-working parents it
degenerates back to v0.3.4 speeds.

### E. Pre-arm UFFD_WP, accept FC's existing write, dedupe

Arm UFFD_WP on FC's memory region before calling FC's snapshot/create.
FC's writes to its own memory don't pass through user-space (they
read memory and write to mem_file_path), so they won't fault. But
the kernel's MMU notifier path might invalidate EPT entries and slow
down vCPU on resume.

After FC's snapshot/create returns, we already have a full memory.bin
on disk. We don't need WpBranch's capture mechanism at all. The
benefit would be... none for the snapshot path. UFFD_WP only helps
if we use it to defer memory copy out of the pause window.

So E is incoherent. Skip.

## Recommendation

**Tier 1 (immediate, ~1 week)**: Path **D** with `Diff` mode and a
pre-BRANCH dirty bitmap flush. Gets v0.4 to ~50 ms pause for idle
parents without patching FC. Documents the regression for
write-heavy parents as a known limitation.

**Tier 2 (parallel, ~2 weeks)**: Path **A**, submit upstream FC patch
for `SnapshotType::VmstateOnly`. When it lands, switch forkd to use
it; pause window drops to ~3 ms (WpBranch arm only).

**Tier 3 (later, when needed)**: Path **C** if upstream rejects A and
we need sub-10 ms unconditionally.

## What's needed to start Tier 1

1. Add `mem_file_path = /dev/shm/forkd-discard-<id>.bin` plumbing in
   `forkd-vmm::snapshot_to_diff`.
2. After FC's snapshot/create returns, `unlink` the discard file
   (or `mmap` it for WP capture if we want overlap, but probably
   simpler not to).
3. Run WpBranch with the source VM's actual memory region to populate
   the real snapshot file.
4. Add a `--live-fork` feature flag in the controller's BranchSandbox
   request shape. Default off until benchmarked vs v0.3.4.
5. Reproduce the v0.3.4 multi-BRANCH bench
   (`bench/pause-window/sweep-diff.sh`) with `--live-fork` to confirm
   the pause-window claim.

## Open questions for next session

- Can we ask FC for a vCPU pause without going through snapshot/create?
  (FC has `/vm` PATCH with `state: Paused` — does that work standalone?)
- If we pause FC then arm UFFD_WP then resume FC then capture, what's
  the ordering with vmstate serialization? Need to be careful that
  vmstate is captured *while paused* and matches the WP-arm point in
  time.
- Does FC's `Diff` mode reset the dirty bitmap as part of
  snapshot/create, or does that need a separate call? (Affects how
  often we need to flush before BRANCH.)

[#157]: https://github.com/deeplethe/forkd/pull/157
[fc-snapshot]: https://github.com/firecracker-microvm/firecracker/blob/main/src/vmm/src/vstate/vm.rs
