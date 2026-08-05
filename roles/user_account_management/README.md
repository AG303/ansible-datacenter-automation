# user_account_management

**Category:** Security & Compliance
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Manages the complete local user/group lifecycle from a single `user_accounts` list variable:
creates and updates accounts (`state: present`), locks accounts that should retain data but
lose login access (`state: disabled`), and fully decommissions accounts with an optional home
directory archive before removal (`state: absent`). Deploys SSH `authorized_keys` from a
var-driven list per account, manages sudo/admin group membership, and enforces home directory
ownership/permissions.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `user_accounts` | `[]` | List of account definitions: `name`, `state`, `comment`, `groups`, `shell`, `create_home`, `ssh_authorized_keys`, `sudo`, `password_from_vault`, `remove_home` |
| `user_sudo_group` | `wheel`/`sudo` | OS-family sudo/admin group name |
| `user_home_base_dir` | `/home` | Base path for home directories |
| `user_home_dir_mode` | `"0750"` | Permission mode applied to managed home directories |
| `user_default_shell` | `/bin/bash` | Default login shell when not set per-account |
| `user_ssh_authorized_keys_exclusive` | `true` | Replace `authorized_keys` entirely with the declared list (prevents drift) |
| `user_archive_home_on_removal` | `true` | tar the home dir before deleting a decommissioned account |
| `user_home_archive_dir` | `/var/backups/decommissioned_users` | Archive destination for decommissioned home directories |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates for initial account passwords |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: user_account_management
      vars:
        user_accounts:
          - name: deploy
            state: present
            comment: "CI/CD deploy service account"
            groups: ["wheel"]
            sudo: true
            ssh_authorized_keys:
              - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... deploy-key"
          - name: contractor_jsmith
            state: disabled
          - name: old_service_acct
            state: absent
            remove_home: true
```

## Secrets manager pattern (per-account `password_from_vault: true`)

```yaml
# HashiCorp Vault (community.hashi_vault)
user_initial_password: "{{ lookup('community.hashi_vault.vault_kv2_get', 'datacenter/local_accounts/' + acct.name,
                             engine_mount_point=vault_kv_mount, url=vault_addr).secret.password }}"

# AWS Secrets Manager (amazon.aws)
user_initial_password: "{{ lookup('amazon.aws.aws_secret', 'datacenter/local_accounts/' + acct.name, region=aws_region) }}"
```

## Notes

- All password lookups use `no_log: true` to prevent secret material from appearing in
  Ansible output/logs.
- Decommissioned accounts are archived to `user_home_archive_dir` before deletion unless
  `user_archive_home_on_removal` is disabled — review retention policy for that directory.

## References

- [ansible.builtin.user module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/user_module.html)
- [ansible.posix.authorized_key module docs](https://docs.ansible.com/ansible/latest/collections/ansible/posix/authorized_key_module.html)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
- [amazon.aws.aws_secret lookup docs](https://docs.ansible.com/ansible/7/collections/amazon/aws/aws_secret_lookup.html)
