# ntp_time_sync

**Category:** Security & Compliance
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Installs and configures `chronyd` as the default NTP time-synchronization provider across
the fleet, sets the system timezone, and verifies sync status as a health-check task after
deployment. On Debian-family hosts, a lightweight `systemd-timesyncd` fallback is available
for minimal images that don't need full chrony features (`ntp_use_systemd_timesyncd_fallback:
true`). RedHat-family hosts always use chrony, matching upstream RHEL/Rocky/Alma defaults.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `ntp_use_systemd_timesyncd_fallback` | `false` | Debian-only: use `systemd-timesyncd` instead of chrony |
| `ntp_servers` | `pool.ntp.org` quartet | List of NTP server lines (chrony syntax, e.g. `"0.pool.ntp.org iburst"`) |
| `ntp_timezone` | `UTC` | System timezone |
| `ntp_makestep_threshold` / `ntp_makestep_limit` | `1.0` / `3` | chrony `makestep` — allow immediate correction for first N large offsets |
| `ntp_rtcsync` | `true` | Sync the hardware RTC from the system clock |
| `ntp_allow_networks` | `[]` | CIDRs allowed to query this host as an NTP peer (chrony `allow`) |
| `ntp_verify_sync_after_deploy` | `true` | Run a post-deploy health check for sync status |
| `ntp_sync_check_retries` / `_delay` | `5` / `10` | Retry/delay for the sync verification health check |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: ntp_time_sync
      vars:
        ntp_timezone: "America/Chicago"
        ntp_servers:
          - "ntp1.internal.example.com iburst"
          - "ntp2.internal.example.com iburst"
```

## Notes

- The health-check task uses `ansible.builtin.command` with `retries`/`until` against
  `chronyc tracking` (or `timedatectl show --property=NTPSynchronized` for the timesyncd
  fallback) rather than assuming sync succeeded immediately after service start.
- `ntp_allow_networks` should only be populated on hosts intentionally serving time to other
  internal hosts (e.g. a designated stratum-2 relay).

## References

- [chrony.conf manual](https://chrony-project.org/doc/4.5/chrony.conf.html)
- [systemd-timesyncd.service documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd-timesyncd.service.html)
- [community.general.timezone module docs](https://docs.ansible.com/ansible/latest/collections/community/general/timezone_module.html)
