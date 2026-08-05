# kernel_update_reboot

**Category:** Patch Management
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Updates the kernel package to the latest available version, determines
whether a reboot is actually required by comparing the currently-running
kernel (`uname -r`) against the newest kernel installed on disk, and — if a
reboot is needed — reboots the host using the **`ansible.builtin.reboot`**
module (never a raw shell `reboot` command), with `reboot_timeout` and
`post_reboot_delay` tuned for datacenter hardware boot times. The role
defaults to `serial: 1` at the play level (one host at a time) with an
operator approval `pause` gate after each successful, verified reboot,
before Ansible proceeds to the next host/batch.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `kernel_reboot_serial` | `1` | Play-level `serial` value — reboots one host at a time by default |
| `kernel_update_cache` | `true` | Refresh dnf/apt metadata before updating the kernel package |
| `kernel_package_name_override` | `""` | Override the OS-default kernel meta-package name if needed |
| `kernel_reboot_enabled` | `true` | Set `false` to update the kernel package only, without rebooting (staged rollout) |
| `kernel_reboot_timeout` | `600` | Seconds `ansible.builtin.reboot` waits for the host to come back |
| `kernel_reboot_connect_timeout` | `5` | Per-attempt SSH connect timeout during reboot polling |
| `kernel_reboot_post_reboot_delay` | `30` | Seconds to wait after connectivity returns before continuing |
| `kernel_reboot_pre_reboot_delay` | `5` | Seconds to wait before issuing the reboot |
| `kernel_reboot_test_command` | `"uptime"` | Command used by the reboot module to confirm the host is fully back |
| `kernel_reboot_pause_enabled` | `true` | Require operator ENTER-to-continue after each successful reboot |
| `kernel_reboot_health_check_service` | `"sshd"` | Service checked post-reboot to confirm host health |

## Example play

```yaml
- name: Update kernel and reboot one host at a time with approval gates
  hosts: "{{ target_hosts | default('all') }}"
  become: true
  gather_facts: true
  serial: "{{ kernel_reboot_serial | default(1) }}"
  roles:
    - role: kernel_update_reboot
      vars:
        kernel_reboot_pause_enabled: true
```

## Secrets manager pattern

This role does not require any secrets — it only interacts with the local
package manager and reboot facility. No Vault/AWS Secrets Manager lookups
are used.

## Notes

- Never replace `ansible.builtin.reboot` with a raw `shell`/`command` reboot
  call — the module correctly waits for SSH to drop, tracks the boot ID
  change, and reconnects, which a bare `reboot` command cannot do safely
  under Ansible's control flow.
- `serial: 1` is the safe default for this role; only raise it after the
  first several batches have proven the kernel update is safe for the fleet.
- Run `patch_rollback_snapshot` beforehand if you need a fast downgrade path
  in case a new kernel fails to boot cleanly.

## References

- [ansible.builtin.reboot module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/reboot_module.html)
- [ansible.builtin.dnf module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/dnf_module.html)
- [ansible.builtin.apt module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)
