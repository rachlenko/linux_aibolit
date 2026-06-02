# doctor

Read-only Linux configuration audit ("Dr. Aibolit"). Part of the
[`rachlenko.linux_aibolit`](https://galaxy.ansible.com/ui/repo/published/rachlenko/linux_aibolit/)
collection.

Compares kernel, network, storage, filesystem, CPU, performance-tuning and
limits settings (plus opt-in PostgreSQL / MySQL / Oracle / Ceph modules) against
general-purpose server best practices and renders one colour-coded HTML report
per host on the control node. **It never mutates the target.**

## Role variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `doctor_modules` | list | `[kernel, network, storage, filesystem, cpu, perftune, limits]` | System modules to run, in order. |
| `doctor_extra_modules` | list | `[]` | Opt-in component modules: `postgres`, `mysql`, `oracle`, `ceph`. |
| `doctor_report_dir` | str | `{{ playbook_dir }}/reports` | Output directory on the control node. |
| `doctor_become` | bool | `false` | Escalate on the target to read restricted config files. |

## Example

```yaml
---
- hosts: servers
  roles:
    - role: rachlenko.linux_aibolit.doctor
      vars:
        doctor_extra_modules: [postgres]
```

See the [collection README](../../README.md) for full documentation, the rule
engine reference, and the module extension guide.

## License

MIT
