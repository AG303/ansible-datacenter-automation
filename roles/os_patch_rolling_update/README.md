# os_patch_rolling_update

**Category:** Patch Management
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Performs a full OS package upgrade fleet-wide (`dnf upgrade` on RedHat family,
`apt full-upgrade` on Debian family) while protecting production availability:
a pre-update health check confirms the host is healthy before touching
packages, the upgrade runs, connectivity is re-established with
`ansible.builtin.wait_for_connection`, and a post-update health check must
pass before the host is considered successfully patched. The role itself does
not set `serial`/`max_fail_percentage` — those are rollout controls that must
live on the **play** (see the wrapper playbook), because Ansible does not
allow `serial` to be declared inside a role. Packages can be excluded from
the upgrade (e.g. to defer a risky `kernel*` bump to the dedicated
`kernel_update_reboot` role).

## Key variables

| Variable | Default | Description |
|---|---|---|
| `rolling_batch_size` | `"30%"` | Batch size for the play-level `serial` directive (set in the wrapper playbook) |
| `max_fail_percentage` | `10` | Play-level circuit breaker; halts the rollout if this percentage of a batch fails |
| `os_patch_exclude_packages` | `[]` | Packages/globs to exclude from this run (e.g. `["kernel*"]`) |
| `os_patch_autoremove` | `true` | Remove orphaned dependency packages after upgrade |
| `os_patch_update_cache` | `true` | Refresh dnf/apt metadata before upgrading |
| `os_patch_pre_check_enabled` / `os_patch_post_check_enabled` | `true` | Toggle pre/post health checks |
| `os_patch_health_check_type` | `"service"` | `"service"` (systemd unit) or `"http"` (URI check) |
| `os_patch_health_check_service` | `"sshd"` | Service checked when type is `service` |
| `os_patch_health_check_url` | `http://localhost/healthz` | URL checked when type is `http` |
| `os_patch_wait_for_connection_timeout` / `_delay` | `300` / `10` | Reconnection wait tuning after upgrade |
| `os_patch_pause_between_batches` | `false` | Require an operator ENTER-to-continue gate between serial batches |

## Example play

```yaml
- name: Roll out full OS patching across the fleet
  hosts: "{{ target_hosts | default('all') }}"
  become: true
  gather_facts: true
  serial: "{{ rolling_batch_size | default('30%') }}"
  max_fail_percentage: "{{ max_fail_percentage | default(10) }}"
  roles:
    - role: os_patch_rolling_update
      vars:
        os_patch_exclude_packages: ["kernel*"]
        os_patch_pause_between_batches: true
```

## Secrets manager pattern

This role does not require any secrets — it only interacts with the local
package manager. No Vault/AWS Secrets Manager lookups are used.

## Notes

- Always run `patch_inventory_facts` first to size the blast radius before a
  fleet-wide rollout.
- Kernel packages should typically be excluded here and handled by
  `kernel_update_reboot`, which manages the reboot safely with `serial: 1`
  and an approval gate.
- `max_fail_percentage` and `serial` MUST be declared on the play, not passed
  as role vars — Ansible ignores `serial`/`max_fail_percentage` set at the
  role level.

## References

- [ansible.builtin.dnf module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/dnf_module.html)
- [ansible.builtin.apt module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)
- [ansible.builtin.wait_for_connection module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/wait_for_connection_module.html)
- [Ansible rolling update / serial documentation](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_strategies.html#setting-the-batch-size-with-serial)
