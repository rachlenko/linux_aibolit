# Linux Conf Doctor — Design

**Date:** 2026-06-02
**Status:** Approved (pending spec review)

## 1. Purpose

A read-only Ansible audit ("doctor") for Linux servers. It inspects system and
component configuration, compares each parameter against general-purpose-server
best practices, and produces a per-host **HTML report** that color-codes every
finding:

- **🟢 green** — correctly configured for a server
- **🟡 yellow** — works, but a sub-optimal default worth improving
- **🔴 red** — wrong/harmful for a server
- **⬜ grey** — not present / could not be read (skipped, never a failure)

The tool **never changes the target**. It is purely diagnostic.

## 2. Confirmed decisions

| Decision | Choice |
|----------|--------|
| Action scope | Audit + report only (read-only). No remediation. |
| Distros | RHEL family (RHEL/CentOS/Rocky/Alma) **and** Debian family (Debian/Ubuntu). |
| Disk method | Config inspection only (sysfs). **No** live I/O benchmarking. |
| Threshold profile | General-purpose server (single baseline, tunable). |
| Execution | Run over SSH against any inventory; localhost works too. |
| Report delivery | One HTML file per host, rendered on the **control node** at `reports/<host>.html`. |
| Dependencies | Pure `ansible-core`. No Galaxy collections. Target needs only Python. |
| Analysis engine | Data-driven catalog (declarative checks) + dynamic per-item analyzers. |
| Architecture | Modular: every check area is a self-contained module; components are opt-in. |

## 3. Architecture

Four stages per host: **collect → analyze → render → write-to-control-node**.

```
linux_conf_doctor/
├── ansible.cfg
├── inventory.ini              # example incl. localhost
├── site.yml                   # plays the role
├── README.md
├── .gitignore                 # ignores reports/
├── reports/                   # output: <hostname>.html (runtime)
└── roles/linux_doctor/
    ├── defaults/main.yml       # doctor_modules, doctor_extra_modules, global opts
    ├── tasks/
    │   ├── main.yml            # loop enabled modules
    │   ├── run_module.yml      # per-module: load catalog → collect → engine → analyze
    │   ├── engine_scalar.yml   # shared catalog comparator (green/yellow/red)
    │   └── render.yml          # template + delegate_to localhost
    ├── modules/
    │   ├── kernel/      { catalog.yml, collect.yml }
    │   ├── network/     { catalog.yml, collect.yml }
    │   ├── storage/     { catalog.yml, collect.yml, analyze.yml }   # per-disk
    │   ├── filesystem/  { catalog.yml, collect.yml, analyze.yml }   # per-mount
    │   ├── cpu/         { catalog.yml, collect.yml }
    │   ├── limits/      { catalog.yml, collect.yml }                # file descriptors
    │   ├── postgres/    { catalog.yml, collect.yml }                # opt-in
    │   ├── mysql/       { catalog.yml, collect.yml }                # opt-in
    │   └── oracle/      { catalog.yml, collect.yml, analyze.yml }   # opt-in
    └── templates/
        └── report.html.j2      # self-contained HTML + inline CSS
```

### Data flow

1. **collect** — each module's `collect.yml` reads raw values into a flat fact
   dict `doctor_facts` (via `/proc/sys` slurps, `/sys/block/*`, `/proc/mounts`,
   `/etc/fstab`, `df`, config files). All reads are non-mutating.
2. **engine_scalar** — shared comparator loops a module's `module_catalog`,
   pulls `doctor_facts[key]`, applies the rule, and appends a result row to
   `doctor_results`.
3. **analyze** *(optional per module)* — dynamic per-item checks the flat
   catalog can't express (per-disk, per-mount, Oracle OS prerequisites).
4. **render** — templates `report.html.j2` from `doctor_results` and writes
   `reports/<inventory_hostname>.html` on the control node via
   `delegate_to: localhost`. No file is written on the target.

## 4. Module contract (the extension point)

Each module is a directory under `roles/linux_doctor/modules/<name>/` with up to
three files — two optional:

- **`catalog.yml`** (required) — defines `module_catalog`, a list of scalar
  check dicts (data only).
- **`collect.yml`** (optional) — populates `doctor_facts` with the raw values the
  catalog references. **Self-guarding**: if the subject isn't present (e.g.
  Postgres not installed), the module skips or emits one grey "not detected" row
  rather than failing the play.
- **`analyze.yml`** (optional) — dynamic per-item checks (loops over discovered
  disks, mounts, databases, etc.) appending their own result rows.

**Engine loop** (`run_module.yml`) for each name in
`doctor_modules + doctor_extra_modules`:
`include_vars` catalog → `include_tasks collect.yml` (if present) →
`engine_scalar.yml` over the catalog → `include_tasks analyze.yml` (if present) →
rows tagged with the module name.

**Enablement.** `doctor_modules` defaults to the system set
`[kernel, network, storage, filesystem, cpu, limits]`. Component modules are
opt-in via `doctor_extra_modules: [postgres]` (or `-e`). No engine changes needed
to add a module. README documents this with `postgres` as the worked example.

### Catalog entry shape

```yaml
- key: vm.swappiness
  category: Kernel / Memory
  label: "Swappiness"
  source: sysctl          # how collect.yml stored it in doctor_facts
  recommended: 10
  rule: max               # green if <=10
  yellow_max: 60          # yellow if 11..60, red if >60
  rationale: "High swappiness makes a server swap out hot pages; 1–10 favors RAM."
```

**Rule types:** `max`, `min`, `equals`, `in_set`, `bool`. Each maps a current
value to green/yellow/red using the entry's thresholds. Unknown/unreadable →
grey.

## 5. Check catalog

### System modules (enabled by default)

**kernel — Kernel / Memory:** `vm.swappiness`, `vm.dirty_ratio`,
`vm.dirty_background_ratio`, `vm.overcommit_memory`, `vm.max_map_count`,
`kernel.pid_max`, `fs.file-max`, `fs.nr_open`, `fs.inotify.max_user_watches`,
Transparent Huge Pages mode, swap present / zram.

**network:** `net.core.somaxconn`, `net.core.netdev_max_backlog`,
`net.ipv4.tcp_max_syn_backlog`, `net.core.rmem_max`/`wmem_max`,
`net.ipv4.tcp_rmem`/`tcp_wmem`, `net.ipv4.ip_local_port_range`, `tcp_tw_reuse`,
`tcp_fin_timeout`, `tcp_congestion_control` (recommend bbr → yellow),
`net.core.default_qdisc` (fq), `tcp_slow_start_after_idle`, `tcp_mtu_probing`.

**storage — per-disk (dynamic):** I/O scheduler vs rotational type
(NVMe/SSD → `none`/`mq-deadline`; HDD → `mq-deadline`/`bfq`), `read_ahead_kb`,
`nr_requests` (queue depth), `rq_affinity`, `add_random`, physical vs logical
block size, filesystem block size. Disk-type + CPU-count proportion guidance
(queue depth / `rq_affinity`) lives here.

**filesystem — per-mount (dynamic):** atime vs `noatime`/`relatime` (red on
strictatime), continuous `discard` vs periodic `fstrim.timer` (recommend timer),
fstab device-name vs UUID (yellow), ext4 reserved-blocks % on large data fs,
fill level from `df` (yellow >80%, red >90%), tmp on tmpfs.

**cpu — CPU / system:** CPU frequency governor (`powersave` on server → yellow
`performance`), CPU count (context for storage recs), active `tuned` profile if
present, NTP/chrony time sync.

**limits — File descriptors:** `fs.file-max` (system-wide), `fs.nr_open`
(per-process ceiling), `fs.file-nr` (current alloc, grey/info), soft/hard
`nofile` from `/etc/security/limits.conf` + `limits.d/*`, systemd
`DefaultLimitNOFILE` (`system.conf`/`user.conf`) and PID 1 effective limit,
`pam_limits` enabled. Server default `nofile` soft 1024 → recommend 65535+.

### Component modules (opt-in, read-only, self-guarding)

All inspect config files + OS-level settings (no DB connection by default) and
cross-reference the system findings that matter most for that engine (THP,
swappiness, hugepages, FD limits, I/O scheduler). RAM/CPU/disk-dependent
recommendations are computed from gathered facts.

**postgres** — detect via `postgresql.conf` / `postmaster`. Reads
`postgresql.conf` (+ `conf.d`, `postgresql.auto.conf`). Checks: `shared_buffers`
(~25% RAM), `effective_cache_size` (~50–75% RAM), `work_mem`,
`maintenance_work_mem`, `max_connections` (warn if very high), `random_page_cost`
(1.1 SSD vs default 4 → yellow), `effective_io_concurrency`,
`checkpoint_completion_target`, `wal_compression`/`max_wal_size`,
`synchronous_commit`, `huge_pages`, `autovacuum` on, `listen_addresses`
exposure. OS cross-refs: THP off, `vm.overcommit_memory`.

**mysql** (MySQL & MariaDB) — detect via `my.cnf` / `mysqld`. Reads
`/etc/my.cnf`, `/etc/mysql/**`, `my.cnf.d`. Checks: `innodb_buffer_pool_size`
(~50–70% RAM), `innodb_log_file_size`, `innodb_flush_log_at_trx_commit`,
`innodb_flush_method` (O_DIRECT), `innodb_file_per_table`, `innodb_io_capacity`
(SSD), `max_connections`, `table_open_cache`, `tmp_table_size`,
`skip_name_resolve`, legacy `query_cache` off.

**oracle** — detect via `oratab` / `ORACLE_HOME` / `pmon`. Strength is the
well-known **OS prerequisites** (`analyze.yml`): kernel `shmmax`/`shmall`/
`shmmni`/`sem`, `fs.aio-max-nr`, `fs.file-max`, `ip_local_port_range`,
`rmem/wmem`, HugePages sizing vs SGA, and `oracle` user limits (`nofile` 65536,
`nproc`, `stack`, `memlock`). Plus `init<SID>.ora`/pfile params when present
(`sga_target`, `pga_aggregate_target`, `db_block_size`, `processes`). SPFILE is
binary; live values would need `sqlplus` — **deferred, off by default**. We
inspect text pfile + OS only.

## 6. Report

`report.html.j2` — one self-contained HTML file, inline CSS, no external assets
or JS.

- **Banner:** hostname, distro + version, kernel, CPU count, RAM, date, and a
  **score** with summary counts (🔴 N / 🟡 N / 🟢 N / ⬜ N).
- **Sections:** one collapsible block per module. Each is a table —
  *Parameter · Current · Recommended · Status · Why*. Status cell background is
  red/yellow/green/grey. Red rows sort to the top within each section.

## 7. Error handling

Every collector is defensive. Missing sysctl keys, absent `/sys` paths
(containers/VMs), unreadable files, or absent components mark that check **grey
(unknown/skipped)** rather than failing the play. `failed_when: false` + presence
guards throughout. The audit never mutates the target and degrades gracefully on
minimal systems.

## 8. Testing

Read-only and environment-dependent, so:

1. `ansible-playbook --syntax-check`
2. `ansible-lint`
3. `ansible-playbook --check` dry run
4. Real run against `localhost` plus containers (one Ubuntu, one Rocky) to
   confirm the HTML renders and degrades gracefully on minimal systems.

README documents how to run all four, how to enable component modules, and how to
add a new module via the contract in §4.

## 9. Out of scope (YAGNI)

- Remediation / applying fixes.
- Live disk I/O benchmarking.
- Live DB introspection via `psql`/`mysql`/`sqlplus` (config + OS inspection
  only).
- Aggregated multi-host dashboard (per-host reports only).
- Selectable workload profiles (single general-purpose baseline).
