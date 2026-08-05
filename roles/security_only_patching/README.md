# security_only_patching

**Category:** Patch Management
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Applies only security-classified updates instead of a full OS upgrade, for
teams that want a lighter-touch, faster-cadence patch cycle between full
maintenance windows. On RHEL family this uses `ansible.builtin.dnf` with
`security: true` (equivalent to `dnf update --security`), optionally
including bugfix errata. On Debian family, since APT has no native
`--security`-only flag, the role installs/uses `unattended-upgrades`, runs a
`--dry-run` preview first for audit purposes, and applies updates restricted
to an allow-listed set of security-pocket origins (e.g.
`jammy-security`, `bookworm-security`). Health checks and connection-recovery
wait wrap the update, matching the safety pattern used by
`os_patch_rolling_update`.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `rolling_batch_size` / `max_fail_percentage` | `"30%"` / `10` | Play-level rollout safety controls |
| `security_patch_update_cache` | `true` | Refresh dnf metadata before checking for security errata |
| `security_patch_bugfix_also` | `false` | RedHat only: also include bugfix-classified errata |
| `security_patch_debian_allowed_origins` | distro-derived `*-security` origins | Allowlist of APT origins treated as "security" |
| `security_patch_debian_dry_run_first` | `true` | Run `unattended-upgrade --dry-run` and log output before applying for real |
| `security_patch_pre_check_enabled` / `_post_check_enabled` | `true` | Toggle pre/post health checks |
| `security_patch_health_check_service` | `"sshd"` | Service checked to confirm host health |
| `security_patch_wait_for_connection_timeout` / `_delay` | `180` / `5` | Reconnection wait tuning |

## Example play

```yaml
- name: Apply security-only patches fleet-wide
  hosts: "{{ target_hosts | default('all') }}"
  become: true
  gather_facts: true
  serial: "{{ rolling_batch_size | default('30%') }}"
  max_fail_percentage: "{{ max_fail_percentage | default(10) }}"
  roles:
    - role: security_only_patching
      vars:
        security_patch_bugfix_also: false
```

## Secrets manager pattern

This role does not require any secrets — it only interacts with the local
package manager and `unattended-upgrades`. No Vault/AWS Secrets Manager
lookups are used.

## Notes

- This role is intended for a higher-frequency, lower-risk patch cadence
  (e.g. weekly) compared to `os_patch_rolling_update` (full upgrade, less
  frequent maintenance windows).
- On Debian family, the security-pocket allowlist must match your
  distribution's actual origin label (`apt-cache policy <pkg>` shows the
  exact `origin,codename` strings available).
- Pair with `unattended_upgrades_config` to make this cadence fully automatic
  instead of Ansible-driven.

## References

- [ansible.builtin.dnf module docs (security option)](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/dnf_module.html)
- [unattended-upgrades documentation (Debian wiki)](https://wiki.debian.org/UnattendedUpgrades)
- [ansible.builtin.apt module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)
