# Design: `perftune` module — NIC + disk IRQ tuning, I/O profiling, fstrim

**Date:** 2026-06-02
**Status:** Approved, ready for implementation plan
**Scope:** Add a new read-only audit module that *checks* and *suggests host-calculated
values* for NIC IRQ affinity / RPS / XPS, storage-controller IRQ affinity, per-device-class
I/O profiling, and the `fstrim.timer` service.

## Motivation

Linux Conf Doctor already audits storage queue knobs (`storage/_disk.yml`: scheduler,
`nr_requests`, `rq_affinity`, `add_random`, read-ahead) and per-mount TRIM strategy
(`filesystem/_mount.yml`). It does **not** audit:

- NIC interrupt placement (IRQ `smp_affinity`, RPS, XPS),
- storage-controller (NVMe/AHCI/virtio) interrupt placement,
- whether the periodic `fstrim.timer` is actually enabled, and
- **host-specific calculated targets** ("best for *this* host") rather than generic constants.

ScyllaDB's `perftune.py` / `iotune` chain solves the same problem by *applying* tuning. This
module brings the same *reasoning* to Linux Conf Doctor as **advice only**, honoring the
project's two hard guarantees: it never mutates the target, and it never runs a live I/O
benchmark. All recommendations are derived from static facts (CPU count, NUMA layout, NIC
queue count, device class, current sysfs/IRQ state).

## Non-goals

- No remediation / applying fixes (audit only — project-wide rule).
- No live I/O benchmark (`fio`/`ioping`/`iotune`) — explicitly out of scope per README.
- No live network throughput probe.
- Not a replacement for the existing `storage`/`network` modules; this is an additional lens.

## Architecture

A new **default-enabled system module** at `roles/linux_doctor/modules/perftune/`, wired in by
appending `perftune` to `doctor_modules` in `defaults/main.yml` (after `cpu`). No engine
changes. The render template already iterates `doctor_modules + doctor_extra_modules`, so the
module renders automatically, using the existing `Current` / `Recommended` / `Status` columns —
the calculated value goes in `Recommended`.

```
roles/linux_doctor/modules/perftune/
├── catalog.yml    # scalar checks (irqbalance state, rps_sock_flow_entries)
├── collect.yml    # self-guarding raw reads → doctor_facts + intermediate lists
├── analyze.yml    # orchestrates dynamic rows; computes the shared CPU-set facts
├── _nic.yml       # per physical NIC: IRQ affinity, RPS, XPS rows
└── _diskirq.yml   # per storage-controller IRQ group + per-class I/O-profile rows
```

All dynamic rows are emitted through the existing shared `tasks/eval_row.yml`, so colour
coding, grey-on-missing, and the `info` rule behave identically to every other module.

## Data collected (all non-mutating)

`collect.yml` is self-guarding (`failed_when: false`, presence guards). It populates
`doctor_facts` (scalars) and intermediate list facts (consumed by `analyze.yml`):

| Fact | Source | Notes |
|---|---|---|
| `perf.ncpu` | `ansible_facts.processor_vcpus` / `processor_count` | logical CPU count |
| `perf.numa_nodes` | count of `/sys/devices/system/node/node[0-9]*` | ≥1; 0/absent → assume 1 |
| `perf.irqbalance` | `systemctl is-active irqbalance` | `active` / `inactive` / `unknown` |
| `perf.fstrim_timer` | `systemctl is-enabled fstrim.timer` | `enabled` / `disabled` / `unknown` |
| `net.core.rps_sock_flow_entries` | sysctl (via `collect_sysctl.yml`) | global RPS flow table |
| `doctor_perf_nics` | `/sys/class/net/*` with a `device` symlink | physical NICs only (skip `lo`, virtual) |
| per-NIC: rx/tx queue counts | count `queues/rx-*` , `queues/tx-*` | multiqueue detection |
| per-NIC: IRQ list + affinity | `/proc/interrupts` match, `/proc/irq/<n>/smp_affinity_list` | current placement |
| per-NIC: `rps_cpus` / `xps_cpus` | `queues/rx-*/rps_cpus`, `queues/tx-*/xps_cpus` | hex masks (all-zero = off) |
| `doctor_perf_diskirq` | `/proc/interrupts` lines `nvme*`/`ahci`/`virtio*` | controller IRQ groups + affinity |
| device class reuse | rotational flag per disk (same read as `storage`) | drives I/O profile + fstrim |

If a subject is absent (e.g. a container with no physical NIC, or no NVMe), the corresponding
analyzer emits exactly **one grey row** rather than failing — matching `storage`'s
"no block devices found" pattern.

## The heuristic engine (computed once in `analyze.yml`)

Two CPU sets are derived from `perf.ncpu` and NUMA layout — the perftune.py idea, simplified
and made explicit:

- **IRQ cores**: all cores if `ncpu <= 8`; otherwise `cores 0 .. k-1` where
  `k = max(2, ncpu // 4)` (≈ one NUMA node's worth on large hosts). Rendered as both a CPU
  list (`0-3`) and a hex mask (`000f`).
- **Compute cores**: the complement of the IRQ-core set (used for the RPS suggestion / app
  shards). On `ncpu <= 8` hosts the two sets coincide (no isolation recommended — too few
  cores to dedicate).

Helper outputs computed in Jinja and reused by the per-subject rows:
`_irq_list` (e.g. `0-3`), `_irq_mask` (e.g. `000f`), `_compute_mask`, `_ncpu`.

### Per-subject rows

| Row | Calculated `Recommended` | Status rule |
|---|---|---|
| **NIC `<n>` IRQ affinity** | `cores {{_irq_list}} (mask {{_irq_mask}})` | `in_set`-style: green if current affinity ⊆ IRQ-core set; yellow otherwise |
| **NIC `<n>` RPS** | if `rx_queues < ncpu` → `enable rps_cpus={{_compute_mask}}`; else `not needed (multiqueue covers all CPUs)` | green if (RPS-on & needed) or (RPS-off & not-needed); yellow if recommended but absent |
| **NIC `<n>` XPS** | mirror of RPS for tx queues | `info` (advisory) |
| **Disk IRQ `<ctrl>` affinity** | `spread across all cores ({{0..ncpu-1}})` for NVMe parallelism | green if spread over ≥2 cores; yellow if pinned to a single core |
| **I/O profile — NVMe** | `nr_requests 1023, read_ahead_kb 128` | `info` (computed target vs current) |
| **I/O profile — SATA SSD** | `nr_requests 256, read_ahead_kb 128` | `info` |
| **I/O profile — HDD** | `nr_requests 128, read_ahead_kb 4096` | `info` |
| **fstrim.timer** | SSD/NVMe present + timer off + no continuous discard → `enable fstrim.timer (weekly)`; else `enabled` | green if enabled; yellow if off with SSD present; grey if HDD-only or no disks |
| **irqbalance** | running is a sound default | `bool`/info: green if active; rationale notes it **overrides manual IRQ pins**, so manual masks above are advisory |

### Design decisions (resolved)

- **IRQ-core sizing** uses `k = max(2, ncpu // 4)` for hosts with `>8` CPUs. Documented and
  easy to tune in one place; not claimed to be optimal for every workload.
- **irqbalance** is treated as **green when active** (a sound general-server default), not
  flagged for disabling. ScyllaDB disables it because it pins IRQs by hand; that is
  app-specific, not a general baseline. The row carries a note that manual masks are advisory
  while irqbalance runs.
- **I/O-profile overlap with `storage`** is intentional: `storage` checks generic floors;
  `perftune` shows the **device-class-aware computed target** as `info` rows. Rationale text
  cross-references the `storage` rows so a reader understands the relationship.

## Error handling & invariants

- Every external read uses `failed_when: false` + `changed_when: false`; nothing is written on
  the target; rendering stays on the control node (unchanged).
- Missing/unreadable values resolve to grey via the shared `eval_row.yml` (`__MISSING__`).
- No new collections or external dependencies; only `ansible.builtin` + shell reads of
  `/proc`, `/sys`, and `systemctl` status, consistent with existing modules.
- `systemctl` reads degrade to `unknown` (grey) on non-systemd or restricted hosts.

## Testing

- `ansible-playbook --syntax-check site.yml` passes with `perftune` enabled.
- `ansible-lint` clean.
- `ansible-playbook -i inventory.ini site.yml --limit localhost` produces a report with a
  **Perftune** section: physical NIC(s) with computed IRQ/RPS suggestions, disk-controller IRQ
  rows, per-class I/O-profile `info` rows, and an fstrim row — and degrades to grey rows in a
  container / NIC-less / disk-less environment without failing the play.
- Manual check that the `Recommended` column shows concrete per-host values
  (e.g. `cores 0-3 (mask 000f)`), not just generic text.

## Out of scope (future, if wanted)

- Applying any of the suggestions (would break the read-only guarantee).
- Per-queue (multi-IRQ) individual affinity layout beyond the aggregate set.
- Workload-profile selection (DB vs web vs storage) — single general-server baseline for now.
