# patch_compliance_report

**Category:** Patch Management
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Cross-references currently installed package versions (via
`ansible.builtin.package_facts`) against a `required_patch_baseline` dict of
`package -> minimum version`, using Ansible's `version()` Jinja test for
robust version comparison across both RPM and dpkg version schemes. Hosts
missing a baseline package that is marked in `patch_baseline_required_packages`
are treated as non-compliant; hosts with an installed version below the
baseline minimum are also flagged. Produces a per-host debug summary plus an
aggregated JSON and CSV compliance report delegated to a persistent Linux
host at play end — never to `localhost`, since AAP 2.6 execution nodes are
ephemeral pods with no durable storage.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `required_patch_baseline` | sample dict (openssl/openssh-server/kernel) | Package name -> minimum acceptable version |
| `patch_baseline_required_packages` | `[]` | Subset of baseline packages that MUST be installed (missing = non-compliant); others are skipped if not installed |
| `patch_compliance_report_host` | `"{{ report_archive_host }}"` | Persistent Linux host reports are delegated/written to (AAP 2.6 execution nodes are ephemeral — never `localhost`) |
| `patch_compliance_report_dir` | `{{ report_archive_base_dir }}/patch_compliance` | Directory on `patch_compliance_report_host` where reports are written |
| `patch_compliance_report_json` / `_csv` | timestamped paths | Report output file paths |
| `patch_compliance_fail_play_on_noncompliance` | `false` | Set `true` to fail the play immediately for any non-compliant host (strict gating mode) |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: patch_compliance_report
      vars:
        required_patch_baseline:
          openssl: "3.0.7"
          openssh-server: "8.7"
        patch_baseline_required_packages: ["openssl", "openssh-server"]
```

## Secrets manager pattern

This role does not require any secrets — it only compares locally installed
package facts against a supplied baseline dict. No Vault/AWS Secrets Manager
lookups are used.

## Notes

- Version comparison uses Jinja's `version()` test, which correctly handles
  both RPM-style (`8.0-5.el9`) and dpkg-style (`1:8.9p1-3ubuntu0.6`) version
  strings for `>=` comparisons.
- Pair with `patch_maintenance_window` and `os_patch_rolling_update` to close
  the loop: report -> patch -> re-report to confirm compliance improved.
- Set `patch_compliance_fail_play_on_noncompliance: true` only in pipelines
  where a hard compliance gate (e.g. pre-audit) is desired; the default is a
  non-blocking reporting mode.
- Override `report_archive_host` in `inventory/group_vars/all.yml` to repoint
  every report-writing role in the collection at once, or set
  `patch_compliance_report_host` here to give this role's reports a different
  home than the rest.

## References

- [ansible.builtin.package_facts module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/package_facts_module.html)
- [Ansible version comparison Jinja test docs](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_tests.html#version-comparison)
