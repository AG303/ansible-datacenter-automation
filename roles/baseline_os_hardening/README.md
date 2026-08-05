# baseline_os_hardening

**Category:** Security & Compliance
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Applies general security best-practice OS hardening that is not mapped to any named
compliance framework (no CIS/STIG control IDs anywhere in this role): blacklists unused
filesystem kernel modules (cramfs, freevxfs, jffs2, hfs/hfsplus, squashfs, udf — `usb-storage`
is opt-in only), applies sysctl network/kernel hardening (reverse-path filtering, disabling
source routing/ICMP redirects, ASLR, dmesg/kptr restriction, ptrace scope), disables IP
forwarding unless the host is explicitly marked as a router, restricts core dump generation,
removes insecure legacy services (telnet, rsh, tftp, ypserv), and hardens `/tmp` mount options
when `/tmp` is already a separate filesystem.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `baseline_disable_filesystems` | cramfs, freevxfs, jffs2, hfs, hfsplus, squashfs, udf | Filesystem kernel modules to blacklist |
| `baseline_is_router` | `false` | Skip IP-forwarding disablement if this host is an intentional router |
| `baseline_sysctl_settings` | see defaults | Core sysctl hardening key/value map |
| `baseline_router_sysctl_settings` | `ip_forward: 0`, etc. | Applied only when `baseline_is_router` is false |
| `baseline_disable_core_dumps` | `true` | Restrict core dump generation via limits.conf and `kernel.core_pattern` |
| `baseline_remove_insecure_packages` | telnet, rsh, tftp, ypserv, etc. | Packages removed if present |
| `baseline_disable_insecure_services` | telnet/rsh/rlogin/rexec sockets | Services stopped and disabled if present |
| `baseline_harden_tmp_mount` | `true` | Apply `nodev,nosuid,noexec` to an existing `/tmp` mount |
| `baseline_tmp_mount_options` | `defaults,nodev,nosuid,noexec` | Mount options applied to `/tmp` |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: baseline_os_hardening
      vars:
        baseline_disable_filesystems:
          - cramfs
          - freevxfs
          - usb-storage   # opt-in on hosts with no legitimate USB storage use case
```

## Notes

- This role intentionally does **not** create a new `/tmp` filesystem — it only hardens the
  mount options on hosts where `/tmp` is already a dedicated mount point, to avoid a
  disruptive filesystem layout change.
- `usb-storage` is excluded from the default blacklist because many datacenter hosts rely on
  it for legitimate maintenance/imaging; add it explicitly per-environment where appropriate.
- General security practice only — no task in this role references any named certification
  or control-ID framework.

## References

- [sysctl.conf manual](https://man7.org/linux/man-pages/man5/sysctl.conf.5.html)
- [ansible.posix.sysctl module docs](https://docs.ansible.com/ansible/latest/collections/ansible/posix/sysctl_module.html)
- [ansible.posix.mount module docs](https://docs.ansible.com/ansible/latest/collections/ansible/posix/mount_module.html)
- [community.general.modprobe module docs](https://docs.ansible.com/ansible/latest/collections/community/general/modprobe_module.html)
