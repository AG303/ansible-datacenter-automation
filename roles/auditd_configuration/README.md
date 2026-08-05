# auditd_configuration

**Category:** Security & Compliance
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Installs and configures the Linux Audit Daemon (`auditd`) with a curated, general
best-practice rule set covering identity/account database changes, privilege escalation,
network configuration changes, scheduled task (cron) modification, sudoers changes, and
key log file watches (wtmp/btmp/lastlog/auth log). Configures log rotation thresholds and
disk-space failure actions in `auditd.conf`, and can optionally forward audit events to a
remote syslog/SIEM collector via the audit dispatcher plugin.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `auditd_log_file` | `/var/log/audit/audit.log` | Audit log file path |
| `auditd_max_log_file_mb` | `50` | Max size per log file before rotation |
| `auditd_num_logs` | `10` | Number of rotated log files retained |
| `auditd_space_left_mb` / `auditd_space_left_action` | `100` / `syslog` | Low-disk-space warning threshold and action |
| `auditd_admin_space_left_mb` / `auditd_admin_space_left_action` | `50` / `suspend` | Critical-disk-space threshold and action |
| `auditd_watch_identity` | `true` | Watch `/etc/passwd`, `/etc/group`, `/etc/shadow`, `/etc/gshadow` |
| `auditd_watch_privilege_escalation` | `true` | Watch setuid execs, `su`, `sudo` |
| `auditd_watch_network_config` | `true` | Watch network configuration files |
| `auditd_watch_cron` | `true` | Watch cron configuration paths |
| `auditd_watch_sudoers` | `true` | Watch `/etc/sudoers` and `/etc/sudoers.d` |
| `auditd_watch_log_files` | `true` | Watch auth log, wtmp, btmp, lastlog |
| `auditd_extra_watches` | `[]` | Additional ad-hoc `{path, perms, key}` watches |
| `auditd_forward_to_syslog_enabled` | `false` | Enable remote syslog/SIEM forwarding plugin |
| `auditd_remote_syslog_server` / `_port` / `_protocol` | `""` / `514` / `tcp` | Remote collector coordinates |
| `auditd_rules_immutable` | `false` | Append `-e 2` to lock the ruleset until reboot |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: auditd_configuration
      vars:
        auditd_forward_to_syslog_enabled: true
        auditd_remote_syslog_server: "siem.internal.example.com"
```

## Notes

- This ruleset targets general security hygiene only — it is **not** mapped to any named
  compliance framework's control IDs (no CIS/STIG references).
- Enabling `auditd_rules_immutable` requires a reboot before rules can be changed again;
  leave disabled during iterative rollout and enable only on a stable, final ruleset.
- `auditctl -s` output is surfaced at the end of the run so operators can confirm the
  ruleset loaded without syntax errors.

## References

- [auditd.conf manual](https://man7.org/linux/man-pages/man5/auditd.conf.5.html)
- [audit.rules manual](https://man7.org/linux/man-pages/man7/audit.rules.7.html)
- [ansible.builtin.template module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html)
