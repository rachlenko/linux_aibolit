# Ansible Collection — rachlenko.linux_aibolit

[![CI](https://github.com/rachlenko/doctor-linux-aibolit/actions/workflows/ci.yml/badge.svg)](https://github.com/rachlenko/doctor-linux-aibolit/actions/workflows/ci.yml)
[![Ansible Galaxy](https://img.shields.io/badge/galaxy-rachlenko.linux__aibolit-660198.svg)](https://galaxy.ansible.com/ui/repo/published/rachlenko/linux_aibolit/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

**Dr. Aibolit for Linux** — a **read-only** Ansible audit ("doctor") for Linux
servers. It inspects system and component configuration, compares each parameter
against general-purpose server best practices, and writes a self-contained
**HTML report per host** on the control node. Every finding is colour-coded:

- 🟢 **green** — correctly configured for a server
- 🟡 **yellow** — works, but a sub-optimal default worth improving
- 🔴 **red** — wrong/harmful for a server
- ⬜ **grey** — not present / could not be read (skipped, never a failure)

**The tool never changes the target.** All reads are non-mutating (`/proc/sys`,
`/sys/block`, `/proc/mounts`, config files), every collector is defensive
(`failed_when: false` + presence guards), and reports are rendered on the
control node via `delegate_to: localhost`.

## Contents

| Type | Name | Purpose |
|------|------|---------|
| Role | `rachlenko.linux_aibolit.linux_doctor` | The read-only audit engine + modules. |
| Playbook | `rachlenko.linux_aibolit.audit` | Ready-to-run audit over your inventory. |

## Requirements

- **Control node:** `ansible-core >= 2.15` (no other Galaxy collections needed).
- **Target:** Python (for Ansible) and read access to the inspected files. Some
  files (`postgresql.conf`, systemd unit limits, `/proc/1/limits`) may need
  privilege — run with `-e doctor_become=true -K` when required.
- **Supported platforms:** RHEL family (RHEL/CentOS/Rocky/Alma 8–9) and Debian
  family (Debian 11–12, Ubuntu 20.04–24.04).

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
# Audit every host in your inventory (one HTML report per host)
ansible-playbook -i inventory.ini rachlenko.linux_aibolit.audit

# Dry run
ansible-playbook -i inventory.ini rachlenko.linux_aibolit.audit --check

# When restricted config files need privilege
ansible-playbook -i inventory.ini rachlenko.linux_aibolit.audit -e doctor_become=true -K
```

Open the result at `reports/<hostname>.html`.

Or use the role directly in your own playbook:

```yaml
---
- name: Audit my fleet
  hosts: servers
  roles:
    - role: rachlenko.linux_aibolit.linux_doctor
      vars:
        doctor_extra_modules: [postgres, ceph]
        doctor_report_dir: /var/log/aibolit
```

## Role variables (public interface)

These are validated automatically via `meta/argument_specs.yml`.

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `doctor_modules` | list | `[kernel, network, storage, filesystem, cpu, perftune, limits]` | System modules to run, in order. |
| `doctor_extra_modules` | list | `[]` | Opt-in component modules: `postgres`, `mysql`, `oracle`, `ceph`. |
| `doctor_report_dir` | str | `{{ playbook_dir }}/reports` | Where per-host HTML reports are written, **on the control node**. |
| `doctor_become` | bool | `false` | Escalate privileges on the target to read restricted config files. |

### Enabling component modules

```bash
# PostgreSQL OS + config tuning
ansible-playbook -i inventory.ini rachlenko.linux_aibolit.audit \
  -e '{"doctor_extra_modules": ["postgres"]}'

# A Ceph storage node (config + OS prerequisites; never connects to the cluster)
ansible-playbook -i inventory.ini rachlenko.linux_aibolit.audit \
  -e '{"doctor_extra_modules": ["ceph"]}'
```

Component modules read configuration files and OS-level settings only. They do
**not** open a database connection. If the component isn't installed they emit a
single grey "not detected" row instead of failing.

## How it works

Four stages per host: **collect → analyze → render → write-to-control-node**.

1. **collect** reads raw values into the flat fact dict `doctor_facts`.
2. **engine_scalar** loops a module's `module_catalog`, looks up
   `doctor_facts[key]`, applies the rule, and appends a result row.
3. **analyze** (optional) adds dynamic per-item rows the flat catalog can't
   express (per-disk, per-mount, RAM-relative DB sizing, OS prereqs).
4. **render** templates the report to `<doctor_report_dir>/<hostname>.html` on
   the control node. Nothing is written on the target.

### Rule types (the catalog comparator)

| rule     | green                            | yellow            | red       |
|----------|----------------------------------|-------------------|-----------|
| `max`    | `<= recommended`                 | `<= yellow_max`   | otherwise |
| `min`    | `>= recommended`                 | `>= yellow_min`   | otherwise |
| `equals` | `== recommended`                 | in `yellow_set`   | otherwise |
| `in_set` | in `green_set`                   | in `yellow_set`   | otherwise |
| `bool`   | truthiness matches `recommended` | —                 | otherwise |
| `info`   | — (always grey, shows value)     | —                 | —         |

Unknown/unreadable values are always **grey**, never a failure.

### Adding a module (the extension point)

Create `roles/linux_doctor/modules/<name>/` with up to three files: a required
`catalog.yml` (`module_catalog` — a list of scalar check dicts; `[]` if fully
dynamic), an optional self-guarding `collect.yml` (populates `doctor_facts`),
and an optional `analyze.yml` (dynamic per-item rows via the shared
`eval_row.yml`). Then add the name to `doctor_modules` or
`doctor_extra_modules`. No engine changes needed; `postgres` is the worked
example.

## Testing

Integration tests cover the public interfaces. See [`tests/`](tests/) and
[`molecule/`](molecule/).

```bash
# ansible-test integration targets (engine rules, full role run, opt-in guard)
ansible-test integration --docker

# Molecule scenario (runs the role against the local controller)
molecule test

# Lint
ansible-lint
```

## Out of scope (by design)

Remediation/applying fixes, live disk I/O benchmarking, live DB introspection
(`psql`/`mysql`/`sqlplus`), aggregated multi-host dashboards, and selectable
workload profiles. See
[`docs/superpowers/specs/2026-06-02-linux-conf-doctor-design.md`](docs/superpowers/specs/2026-06-02-linux-conf-doctor-design.md)
for the full design.

## License

MIT — see [LICENSE](LICENSE).
