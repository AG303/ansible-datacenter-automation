# selinux_apparmor_management

**Category:** Security & Compliance
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Provides a single interface for managing the mandatory access control (MAC) subsystem
across both OS families. On RedHat-family hosts it manages the SELinux mode (enforcing/
permissive/disabled), policy type, custom booleans, and custom file contexts (with an
automatic `restorecon` pass). On Debian-family hosts it manages AppArmor profile mode
(enforce/complain) via `community.general.apparmor`, falling back to the `aa-enforce`/
`aa-complain` CLI tools with proper `changed_when` logic when the module is unavailable
or fails for a given profile.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `selinux_state` | `enforcing` | SELinux mode: `enforcing`, `permissive`, or `disabled` |
| `selinux_policy` | `targeted` | SELinux policy type |
| `selinux_booleans` | `[]` | List of `{name, state}` SELinux booleans to enforce |
| `selinux_file_contexts` | `[]` | List of `{target, setype}` custom fcontext rules |
| `selinux_restorecon_after_fcontext` | `true` | Run `restorecon -Rv` after applying a changed fcontext |
| `selinux_disable_requires_ack` | `true` | Guard rail requiring explicit acknowledgement to disable SELinux |
| `selinux_disable_acknowledged` | `false` | Set `true` to confirm intentional SELinux disable |
| `apparmor_default_mode` | `enforce` | Default mode applied to `apparmor_profiles` entries without an explicit mode |
| `apparmor_profiles` | `[]` | List of `{name, mode}` AppArmor profiles to manage |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: selinux_apparmor_management
      vars:
        selinux_state: enforcing
        selinux_booleans:
          - name: httpd_can_network_connect
            state: true
        selinux_file_contexts:
          - target: "/srv/myapp(/.*)?"
            setype: httpd_sys_content_t
        apparmor_profiles:
          - name: usr.sbin.nginx
            mode: enforce
```

## Notes

- Setting `selinux_state: disabled` is a high-impact, reboot-required change and is
  blocked by default unless `selinux_disable_acknowledged: true` is explicitly set;
  prefer `permissive` for troubleshooting instead of a full disable.
- The AppArmor CLI fallback path only runs for profiles where the
  `community.general.apparmor` module task reported a failure, keeping the module path
  as the primary, idempotent mechanism.

## References

- [ansible.posix.selinux module docs](https://docs.ansible.com/ansible/latest/collections/ansible/posix/selinux_module.html)
- [community.general.seboolean module docs](https://docs.ansible.com/ansible/latest/collections/community/general/seboolean_module.html)
- [community.general.sefcontext module docs](https://docs.ansible.com/ansible/latest/collections/community/general/sefcontext_module.html)
- [community.general.apparmor module docs](https://docs.ansible.com/ansible/latest/collections/community/general/apparmor_module.html)
