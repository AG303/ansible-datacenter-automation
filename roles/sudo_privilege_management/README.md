# sudo_privilege_management

**Category:** Security & Compliance
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Generates `/etc/sudoers.d/*` drop-in files from a single `sudo_rules` list variable, so
privileged command grants are version-controlled and auditable. Each rule is rendered to a
staging file, validated with `visudo -c` before activation (invalid syntax never reaches the
live `sudoers.d` directory), and any prior version of the same file is backed up first.
Optionally prunes previously-deployed rule files that have since been removed from
`sudo_rules`.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `sudo_rules` | `[]` | List of rule definitions: `name, principal_type (user\|group), principal, commands, nopasswd, hosts, runas` |
| `sudo_sudoers_d_path` | `/etc/sudoers.d` | Destination directory for generated drop-in files |
| `sudo_file_mode` / `_owner` / `_group` | `0440` / `root` / `root` | Ownership/permissions applied to generated files |
| `sudo_validate_before_activation` | `true` | Run `visudo -c` against the staged file before activating it |
| `sudo_backup_existing_files` | `true` | Back up the prior version of a rule file before overwriting |
| `sudo_backup_dir` | `/var/backups/sudoers_d` | Backup destination directory |
| `sudo_prune_unmanaged_files` | `false` | Remove previously-deployed rule files no longer declared in `sudo_rules` |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: sudo_privilege_management
      vars:
        sudo_rules:
          - name: sre-team-full-access
            principal_type: group
            principal: sre
            commands: ["ALL"]
            nopasswd: false
            hosts: "ALL"
            runas: "ALL"
          - name: deploy-svc-restart-only
            principal_type: user
            principal: deploy
            commands:
              - /usr/bin/systemctl restart myapp
              - /usr/bin/systemctl status myapp
            nopasswd: true
```

## Notes

- Every generated file is rendered to a temporary staging path first and validated with
  `visudo -c -f <staged file>`; the task fails before touching the live `sudoers.d` directory
  if validation fails, preventing a broken sudo configuration from ever being activated.
- `sudo_prune_unmanaged_files` only removes files that carry this role's marker comment,
  so hand-authored or other-tool-managed `sudoers.d` files are never touched.

## References

- [sudoers manual](https://man7.org/linux/man-pages/man5/sudoers.5.html)
- [visudo manual](https://man7.org/linux/man-pages/man8/visudo.8.html)
- [ansible.builtin.template module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html)
