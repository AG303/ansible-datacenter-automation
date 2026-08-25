# patch_inventory_facts

**Category:** Patch Management
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Read-only fleet-wide reporting role. It gathers full installed-package facts
(`ansible.builtin.package_facts`), runs a `changed_when: false` availability
check (`dnf check-update` on RedHat family, `apt list --upgradable` on Debian
family), computes an available-updates count per host, and — once every host
in the play has reported — aggregates the results into a single JSON and CSV
summary written to a persistent Linux host via
`delegate_to: "{{ patch_inventory_report_host }}"` / `run_once: true`.
AAP 2.6 execution nodes are ephemeral pods, so reports are never written to
`localhost`. Designed to run ahead of any actual patch rollout so operators
know the blast radius before touching anything.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `patch_inventory_report_host` | `"{{ report_archive_host }}"` | Persistent Linux host reports are delegated/written to (AAP 2.6 execution nodes are ephemeral — never `localhost`) |
| `patch_inventory_report_dir` | `{{ report_archive_base_dir }}/patch_inventory` | Directory on `patch_inventory_report_host` where reports are written |
| `patch_inventory_report_file` | timestamped path under report dir | JSON summary output path |
| `patch_inventory_write_csv` | `true` | Also write a flat CSV summary |
| `patch_inventory_report_csv_file` | timestamped path under report dir | CSV summary output path |
| `patch_inventory_include_package_facts` | `true` | Collect full installed-package inventory via `package_facts` |
| `patch_inventory_fail_on_check_error` | `false` | Whether a non-zero exit from the check-update command should fail the play (dnf returns 100 when updates ARE found — treated as success) |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: patch_inventory_facts
      vars:
        patch_inventory_report_host: "reports01.dc1.example.com"
        patch_inventory_report_dir: "/opt/patch_reports"
```

## Secrets manager pattern

This role does not require any secrets — it only reads local package state
and check-update output. No Vault/AWS Secrets Manager lookups are used.

## Notes

- Run this role before `os_patch_rolling_update` or `security_only_patching`
  to get a pre-flight count of pending updates across the fleet.
- The aggregation step uses `ansible_play_hosts_all` and `hostvars` so it must
  run in the same play as the fact-gathering tasks (do not split into a
  separate play without re-gathering `patch_inventory_host_summary`).
- `dnf check-update` legitimately exits `100` when updates are available;
  this is treated as a non-failure by default.
- Override `report_archive_host` in `inventory/group_vars/all.yml` to repoint
  every report-writing role in the collection at once, or set
  `patch_inventory_report_host` here to give this role's reports a different
  home than the rest.

## References

- [ansible.builtin.package_facts module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/package_facts_module.html)
- [ansible.builtin.command module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/command_module.html)
- [DNF check-update documentation](https://dnf.readthedocs.io/en/latest/command_ref.html#check-update-command)
