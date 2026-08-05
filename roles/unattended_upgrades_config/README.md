# unattended_upgrades_config

**Category:** Patch Management
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Configures a fully automated patching cadence so hosts self-patch on a
schedule without a human running an Ansible play every time. On RHEL family
this installs and configures `dnf-automatic`, templating `/etc/dnf/automatic.conf`
and overriding the systemd timer's `OnCalendar` schedule via a drop-in unit.
On Debian family this installs `unattended-upgrades` + `apt-listchanges`,
templates `/etc/apt/apt.conf.d/50unattended-upgrades` (origin-pattern
allowlist, auto-reboot window, mail reporting) and
`/etc/apt/apt.conf.d/20auto-upgrades` (enables the daily `apt-daily`/
`apt-daily-upgrade` timer cadence). Both paths support an optional
auto-reboot window and email/webhook notification of applied updates.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `auto_patch_enabled` | `true` | Master on/off switch for the automated cadence |
| `auto_patch_apply_security_only` | `true` | Restrict the automatic cadence to security-classified updates |
| `auto_patch_schedule_calendar` | `"*-*-* 03:00:00"` | systemd `OnCalendar` expression for the dnf-automatic timer override |
| `auto_patch_auto_reboot` | `false` | Whether the automated cadence itself may reboot the host |
| `auto_patch_reboot_window_start` / `_end` | `02:00` / `04:00` | Reboot window (HH:MM), used when `auto_patch_auto_reboot: true` |
| `auto_patch_notify_email` | `""` | Email address for update-applied notifications (blank disables) |
| `auto_patch_notify_webhook_url` | `""` | Webhook URL for update-applied notifications (blank disables) |
| `auto_patch_notify_webhook_secret_enabled` | `false` | Set `true` if the webhook URL must be pulled from a secrets manager |
| `dnf_automatic_upgrade_type` | `"security"` | `"default"` (all) or `"security"` for dnf-automatic |
| `apt_daily_upgrade_enabled` | `true` | Enable the `apt-daily`/`apt-daily-upgrade` systemd timers |
| `unattended_upgrades_origins_patterns` | distro-security origins | Allowlist of APT origins unattended-upgrades is permitted to install from |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: unattended_upgrades_config
      vars:
        auto_patch_schedule_calendar: "*-*-* 02:30:00"
        auto_patch_auto_reboot: true
        auto_patch_notify_email: "patching-alerts@example.com"
```

## Secrets manager pattern (webhook URL, if treated as sensitive)

```yaml
# HashiCorp Vault (community.hashi_vault)
auto_patch_notify_webhook_url: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                                    engine_mount_point=vault_kv_mount, url=vault_addr).secret.webhook_url }}"

# AWS Secrets Manager (amazon.aws)
auto_patch_notify_webhook_url: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

Set `auto_patch_notify_webhook_secret_enabled: true` and mark the task that
consumes this value `no_log: true` in your local override if you adopt this
pattern, since webhook URLs often embed a bearer token.

## Notes

- `auto_patch_auto_reboot: true` should only be enabled on non-critical tiers
  or hosts behind a load balancer with health-checked draining; for anything
  requiring an approval gate use `kernel_update_reboot` instead.
- This role complements — it does not replace — `os_patch_rolling_update` and
  `security_only_patching`, which are for Ansible-orchestrated, on-demand
  rollouts with health checks and rollout safety controls.
- The webhook notify script referenced in the templates
  (`/usr/local/bin/patch-webhook-notify.sh`) is expected to be deployed by
  the `patch_notification_alerting` role or an equivalent local script; this
  role only wires the config hook, matching the aggregated fleet-wide POST
  performed by `patch_notification_alerting`.

## References

- [dnf-automatic documentation](https://dnf.readthedocs.io/en/latest/automatic.html)
- [unattended-upgrades documentation (Debian wiki)](https://wiki.debian.org/UnattendedUpgrades)
- [ansible.builtin.template module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html)
- [ansible.builtin.systemd module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/systemd_module.html)
