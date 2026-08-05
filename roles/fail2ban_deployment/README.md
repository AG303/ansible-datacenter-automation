# fail2ban_deployment

**Category:** Security & Compliance
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Installs and configures `fail2ban` to automatically block brute-force login attempts. Ships
with an `sshd` jail enabled by default (log path auto-selected per OS family) plus disabled
`nginx-http-auth` and `apache-auth` jail templates that can be turned on where those services
run. Ban timing (bantime/findtime/maxretry), progressive ban-time escalation for repeat
offenders, and a trusted CIDR whitelist (`fail2ban_ignoreip`) are all variable-driven.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `fail2ban_bantime` | `1h` | Base ban duration |
| `fail2ban_findtime` | `10m` | Window in which `maxretry` failures trigger a ban |
| `fail2ban_maxretry` | `5` | Failed attempts before ban |
| `fail2ban_bantime_increment` | `true` | Progressively increase ban time for repeat offenders |
| `fail2ban_bantime_factor` | `2` | Multiplier applied per repeat offense |
| `fail2ban_bantime_maxtime` | `1w` | Ceiling on escalated ban time |
| `fail2ban_ignoreip` | `[127.0.0.1/8, ::1]` | Trusted CIDR/IP whitelist never banned |
| `fail2ban_jails` | sshd enabled; nginx/apache disabled | List of jail definitions: `name, enabled, port, filter, logpath, maxretry, bantime, findtime` |
| `fail2ban_destemail` / `fail2ban_sender` / `fail2ban_action` | see defaults | Notification behavior on ban events |
| `fail2ban_backend` | `auto` | Log-monitoring backend (`auto`, `systemd`, `pyinotify`, `polling`) |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: fail2ban_deployment
      vars:
        fail2ban_ignoreip:
          - 127.0.0.1/8
          - ::1
          - 10.0.0.0/8
        fail2ban_jails:
          - name: sshd
            enabled: true
            port: ssh
            filter: sshd
            logpath: /var/log/secure
            maxretry: 3
            bantime: 1h
            findtime: 10m
```

## Notes

- Always include your management/monitoring CIDR ranges in `fail2ban_ignoreip` before rollout
  to avoid self-locking automation or jump hosts out.
- The `nginx-http-auth` and `apache-auth` jails are disabled by default; enable them per-host
  only where the corresponding web server is actually installed, otherwise fail2ban will warn
  about a missing log file.

## References

- [fail2ban jail.conf documentation](https://github.com/fail2ban/fail2ban/blob/master/config/jail.conf)
- [ansible.builtin.template module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html)
- [ansible.builtin.package module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/package_module.html)
