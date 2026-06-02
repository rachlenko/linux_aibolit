# Dr. Aibolit for Linux — `rachlenko.linux_aibolit`

[![CI](https://github.com/rachlenko/doctor-linux-aibolit/actions/workflows/ci.yml/badge.svg)](https://github.com/rachlenko/doctor-linux-aibolit/actions/workflows/ci.yml)
[![Ansible Galaxy](https://img.shields.io/badge/galaxy-rachlenko.linux__aibolit-660198.svg)](https://galaxy.ansible.com/ui/repo/published/rachlenko/linux_aibolit/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![ansible-lint: production](https://img.shields.io/badge/ansible--lint-production-success.svg)](https://ansible.readthedocs.io/projects/lint/)

> A **read-only** Ansible collection that audits a Linux host against
> general-purpose-server best practices and writes a self-contained, colour-coded
> **HTML report** — telling you, parameter by parameter, what is wrong, what the
> value *should* be for *this* host, and why. **It never changes the target.**

Like a doctor, it diagnoses — it does not operate. You stay in control of every
change.

---

## Table of contents

- [Why this exists](#why-this-exists)
- [What problems it solves](#what-problems-it-solves)
- [What a report looks like](#what-a-report-looks-like)
- [What it audits](#what-it-audits)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick start](#quick-start)
- [Using the role directly](#using-the-role-directly)
- [Configuration](#configuration)
- [How it works](#how-it-works)
- [Design guarantees](#design-guarantees)
- [Extending it: add your own checks](#extending-it-add-your-own-checks)
- [Development & testing](#development--testing)
- [Out of scope](#out-of-scope)
- [License](#license)

---

## Why this exists

Linux ships **conservative, compatibility-first defaults** — tuned so a kernel
boots on a laptop, a VM, or a 40-year-old SCSI controller, *not* so your database
server performs. Left untouched, those defaults quietly cap throughput and
reliability:

| Default you probably still have | Why it hurts a server |
|---|---|
| `vm.swappiness = 60` | Swaps out hot pages under memory pressure |
| `net.core.netdev_max_backlog = 1000` | Drops packets during traffic bursts |
| `net.core.rmem_max / wmem_max ≈ 208 KB` | Caps TCP windows on 10GbE+ links |
| `fs.aio-max-nr = 65536` | Starves async-I/O databases / BlueStore |
| `fs.inotify.max_user_instances = 128` | Breaks containers and file-watchers |
| Transparent Huge Pages = `always` | Latency stalls for PostgreSQL/MySQL/Ceph/Oracle |
| `fstrim.timer` disabled | SSD/NVMe performance degrades over time |

The *correct* values are **scattered** across kernel docs, vendor prerequisites
(Oracle, Ceph), tuning blogs, and tribal knowledge — and the right number often
**depends on the host** (RAM, CPU count, disk class, NUMA layout). Applying all
of that by hand, consistently, across a fleet, is error-prone.

Most existing tools fall into one of two camps:

- **They change the system** (tuning roles, `tuned`, `perftune.py`) — risky to
  point at a production box you don't fully understand yet.
- **They only collect facts** — dumping raw values with no judgement about what
  is good, bad, or host-appropriate.

**Dr. Aibolit sits in the gap:** it *reasons* about your configuration and tells
you what to fix — without touching anything.

## What problems it solves

- **"Is this server tuned correctly?"** — one read-only pass grades every
  parameter 🟢/🟡/🔴 with the recommended value and a one-line rationale.
- **"What's the right value for *this* host?"** — recommendations are
  **host-calculated**, not generic constants: PostgreSQL `shared_buffers` from
  RAM, NIC IRQ affinity from CPU topology, I/O profile from disk class, Ceph RAM
  budget from OSD count.
- **"I can't risk changing production yet."** — the audit is **provably
  non-mutating** and runs no I/O benchmark, so it's safe to run anywhere,
  anytime, including during incidents.
- **"Best practices live in ten different places."** — OS, PostgreSQL, MySQL,
  Oracle and Ceph prerequisites are encoded as declarative, reviewable catalogs
  in one collection.
- **"I need something to hand to a client / manager."** — the output is a
  single, self-contained HTML file per host (inline CSS, no JS, no external
  assets) you can email or archive.

## What a report looks like

`reports/<hostname>.html` opens to a scored banner (host facts + 🔴/🟡/🟢/⬜
counts) and one collapsible section per module. Real findings from a 32-core
Debian/Proxmox node:

```
🩺 Linux Conf Doctor — node3        Debian 12 · kernel 6.8 · 32 CPU · 128 GB
Score 59/100      🔴 5 critical   🟡 19 warning   🟢 35 good   ⬜ 22 skipped

Parameter                     Current     Recommended                 Status   Why
───────────────────────────────────────────────────────────────────────────────────
Swap or zram present          false       true                        🔴 RED   No swap → hard OOM kills
net.core.rmem_max             212992      16777216                     🔴 RED   Caps TCP window on 10GbE
Swappiness                    60          10                           🟡 WARN  Swaps out hot pages
Congestion control            cubic       bbr                          🟡 WARN  BBR improves lossy links
NIC eth0 RPS                  rps off     enable rps_cpus=ffffff00     🟡 WARN  4 rx-queues < 32 CPUs
Computed IRQ-core set         ncpu=32     IRQ cores 0-7 (mask 00ff)    ⬜ INFO  Host-calculated target
```

The **Recommended** column carries the concrete, per-host value — e.g.
`cores 0-7 (mask 00ff)` — not just generic advice.

## What it audits

### System modules (run by default)

| Module | Checks | Audits |
|---|---|---|
| `kernel` | 14 | swappiness, dirty ratios, overcommit, THP, `pid_max`, `file-max`, `nr_open`, `aio-max-nr`, inotify limits, `vfs_cache_pressure`, swap |
| `network` | 14 | socket backlogs, TCP buffers, ephemeral ports, TIME-WAIT reuse, congestion control (BBR), `default_qdisc` (fq), MTU probing |
| `storage` | per-disk | I/O scheduler vs rotational class, `nr_requests`, `rq_affinity`, `add_random`, read-ahead, block size |
| `filesystem` | per-mount | `noatime`/`relatime`, continuous vs periodic TRIM, fstab UUID vs device name, fill level, `/tmp` backing |
| `cpu` | 4 | frequency governor, CPU count, active `tuned` profile, time sync |
| `perftune` | 2 + dynamic | **host-calculated** NIC IRQ/RPS/XPS, storage-controller IRQ spread, per-class I/O-profile targets, `fstrim.timer`, irqbalance |
| `limits` | 8 | system & per-process FD ceilings, `limits.conf` soft/hard `nofile`, systemd `DefaultLimitNOFILE`, PID 1 effective limit, `pam_limits` |

### Component modules (opt-in, self-guarding)

Enable per run via `doctor_extra_modules`. Each reads **config files + OS
settings only** — it never opens a database/cluster connection — and emits a
single grey *"not detected"* row when the component is absent.

| Module | Checks | Highlights |
|---|---|---|
| `postgres` | 12 + dynamic | `shared_buffers`/`effective_cache_size` from RAM, `random_page_cost` for SSD, WAL, autovacuum, `listen_addresses`, THP/overcommit cross-refs |
| `mysql` | 10 + dynamic | `innodb_buffer_pool_size` from RAM, flush method/durability, `io_capacity`, `skip_name_resolve`, legacy query cache |
| `oracle` | 18 + dynamic | the well-known OS prerequisites: `shmmax/shmall/shmmni/sem`, `aio-max-nr`, `file-max`, oracle-user limits, HugePages vs SGA, text pfile params |
| `ceph` | 21 + dynamic | `cephx` auth, `size`/`min_size`, cluster/public network separation, `pid_max`/`aio-max-nr`, RAM-vs-OSD budget, swap/THP/time-sync — grounded in Ceph + Proxmox best practices |

> **103 declarative scalar checks** plus dynamic per-disk / per-mount / per-NIC /
> per-OSD analyzers, all colour-coded through one shared rule engine.

## Requirements

- **Control node:** `ansible-core >= 2.15` (no other Galaxy collections needed).
- **Target:** Python (for Ansible) and read access to the inspected files. A few
  files (`postgresql.conf`, systemd unit limits, `/proc/1/limits`) may need
  privilege — pass `-e doctor_become=true -K` when required.
- **Supported platforms:** RHEL family (RHEL/CentOS/Rocky/Alma 8–9) and Debian
  family (Debian 11–12, Ubuntu 20.04–24.04). Degrades gracefully (grey rows) on
  minimal/container hosts rather than failing.

## Installation

From Ansible Galaxy:

```bash
ansible-galaxy collection install rachlenko.linux_aibolit
```

Or pin it in a `requirements.yml`:

```yaml
---
collections:
  - name: rachlenko.linux_aibolit
    version: ">=1.0.0"
```

```bash
ansible-galaxy collection install -r requirements.yml
```

## Quick start

```bash
# 1. List the hosts to audit (localhost works too)
cat > inventory.ini <<'EOF'
[servers]
web01 ansible_host=10.0.0.5 ansible_user=admin
EOF

# 2. Run the read-only audit — one HTML report per host
ansible-playbook -i inventory.ini rachlenko.linux_aibolit.audit

# 3. Open the result
open reports/web01.html      # macOS  (xdg-open on Linux)
```

Common variations:

```bash
# Dry run (no reads are mutating anyway, but useful for plumbing)
ansible-playbook -i inventory.ini rachlenko.linux_aibolit.audit --check

# Audit a single host
ansible-playbook -i inventory.ini rachlenko.linux_aibolit.audit --limit web01

# When restricted config files need privilege escalation
ansible-playbook -i inventory.ini rachlenko.linux_aibolit.audit -e doctor_become=true -K

# Add opt-in component modules
ansible-playbook -i inventory.ini rachlenko.linux_aibolit.audit \
  -e '{"doctor_extra_modules": ["postgres", "ceph"]}'
```

## Using the role directly

Prefer to wire it into your own play? Use the role:

```yaml
---
- name: Audit my fleet
  hosts: servers
  gather_facts: true
  roles:
    - role: rachlenko.linux_aibolit.doctor
      vars:
        doctor_extra_modules: [postgres, ceph]
        doctor_report_dir: /var/log/aibolit
```

## Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `doctor_modules` | list | `[kernel, network, storage, filesystem, cpu, perftune, limits]` | System modules to run, in order. |
| `doctor_extra_modules` | list | `[]` | Opt-in component modules: `postgres`, `mysql`, `oracle`, `ceph`. |
| `doctor_report_dir` | str | `{{ playbook_dir }}/reports` | Where per-host HTML reports are written, **on the control node**. |
| `doctor_become` | bool | `false` | Escalate privileges on the target to read restricted config files. |

The thresholds themselves live in each module's `catalog.yml` — they are plain
data, so you can fork the baseline to match a stricter or workload-specific
profile without touching any logic.

## How it works

Four stages per host: **collect → analyze → render → write-to-control-node**.

1. **collect** — each module reads raw values (`/proc/sys`, `/sys`, config
   files, `systemctl` status) into a flat fact dict `doctor_facts`. Every read is
   defensive (`failed_when: false` + presence guards).
2. **engine_scalar** — a shared comparator loops the module's `module_catalog`,
   looks up `doctor_facts[key]`, applies the rule, and appends a colour-coded
   result row.
3. **analyze** *(optional)* — dynamic per-item checks the flat catalog can't
   express: per-disk, per-mount, per-NIC, RAM-relative DB sizing, OS
   prerequisites — all funnelled through the same shared row evaluator.
4. **render** — a single Jinja template writes `reports/<hostname>.html` on the
   **control node** via `delegate_to: localhost`. Nothing is written on the
   target.

### Rule types (the catalog comparator)

| rule     | green                            | yellow            | red       |
|----------|----------------------------------|-------------------|-----------|
| `max`    | `<= recommended`                 | `<= yellow_max`   | otherwise |
| `min`    | `>= recommended`                 | `>= yellow_min`   | otherwise |
| `equals` | `== recommended`                 | in `yellow_set`   | otherwise |
| `in_set` | in `green_set`                   | in `yellow_set`   | otherwise |
| `bool`   | truthiness matches `recommended` | —                 | otherwise |
| `info`   | — (always grey, shows value)     | —                 | —         |

Unknown/unreadable values are always **grey** — skipped, never a failure.

## Design guarantees

- **Read-only.** No task mutates the target; there is no remediation path. The
  audit is safe to run on production, mid-incident.
- **No benchmarks.** Recommendations are derived from static facts (CPU count,
  RAM, NUMA, device class, current sysfs/IRQ state) — never from a live I/O or
  network probe.
- **Control-node rendering.** Reports are written on the controller, so the
  target's filesystem is never touched.
- **No surprises on minimal hosts.** Missing components/paths degrade to grey
  rows; the play does not fail in containers or stripped-down VMs.
- **No heavy dependencies.** Only `ansible.builtin` plus shell reads of `/proc`,
  `/sys` and `systemctl` status.

## Extending it: add your own checks

A module is a directory under `roles/doctor/modules/<name>/` with up to
three files:

- **`catalog.yml`** *(required)* — defines `module_catalog`, a list of scalar
  check dicts (`[]` if the module is fully dynamic). One entry:

  ```yaml
  - key: vm.swappiness          # also the doctor_facts lookup key
    category: Kernel / Memory
    label: Swappiness
    source: sysctl              # 'sysctl' values are auto-read for you
    rule: max
    recommended: 10
    yellow_max: 60
    rationale: "High swappiness makes a server swap out hot pages; 1–10 favours RAM."
  ```

- **`collect.yml`** *(optional)* — populates `doctor_facts`; must be
  self-guarding (skip or emit one grey row when the subject is absent).
- **`analyze.yml`** *(optional)* — dynamic per-item rows via the shared
  `tasks/eval_row.yml`.

Then add the name to `doctor_modules` (default) or `doctor_extra_modules`
(opt-in). **No engine changes are required** — `postgres` is the worked example,
`perftune` shows host-calculated recommendations.

## Development & testing

```bash
# Static analysis — passes at ansible-lint's strict 'production' profile
ansible-lint

# Playbook plumbing
ansible-playbook --syntax-check site.yml

# Local end-to-end run against the controller (Molecule)
molecule test
```

The repository also ships a `site.yml` + `inventory.ini` for running straight
from a clone (without installing the collection) during development.

## Out of scope (by design)

Remediation / applying fixes, live disk-I/O benchmarking, live DB introspection
(`psql`/`mysql`/`sqlplus`), aggregated multi-host dashboards, and selectable
workload profiles. See the design specs under
[`docs/superpowers/specs/`](docs/superpowers/specs/) for the full rationale.

## License

MIT — see [LICENSE](LICENSE). Maintained by
[Evgeny Rachlenko](https://github.com/rachlenko).
