# config_backup_git

**Category:** Disaster Recovery
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Periodically mirrors a configurable list of `/etc` configuration files/directories into a local
git working tree, commits them only when `git status --porcelain` shows real changes, and pushes
to a bare repository on a remote backup host over SSH. This provides fast change-tracking and a
quick config-only disaster recovery path independent of full-system backups. The SSH private key
used to push is resolved from an external secrets manager, never stored in plaintext in the repo.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `config_backup_paths` | see defaults | List of files/directories under `/etc` to track |
| `config_backup_local_repo` | `/var/lib/config-backup-git` | Local git working directory |
| `config_backup_commit_author_name` / `_email` | see defaults | Commit author identity for the bot |
| `config_backup_remote_enabled` | `true` | Whether to push to the remote bare repository |
| `config_backup_remote_host` / `_user` / `_bare_path` | see defaults | Remote bare repo coordinates |
| `config_backup_ssh_key_source` | `vault` | `vault` or `aws_secrets_manager` |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |
| `config_backup_ssh_key_path` | `/root/.ssh/config_backup_git_ed25519` | Where the resolved private key is installed |
| `config_backup_commit_message` | templated with timestamp/hostname | Commit message for each snapshot |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: config_backup_git
      vars:
        config_backup_paths:
          - /etc/ssh/sshd_config
          - /etc/haproxy/haproxy.cfg
          - /etc/nginx
```

## Secrets manager pattern

```yaml
# HashiCorp Vault (community.hashi_vault)
config_backup_ssh_private_key: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                                    engine_mount_point=vault_kv_mount, url=vault_addr).secret.private_key }}"

# AWS Secrets Manager (amazon.aws)
config_backup_ssh_private_key: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- Commits are idempotent — the role only stages/commits/pushes when `git status --porcelain`
  reports actual changes, so re-running on a schedule with no drift produces no new commits.
- The remote bare repository must already exist on the backup host (`git init --bare`); this
  role only manages the client-side working copy and push.
- Restoring a single config file after an incident is a simple `git checkout <path>` against the
  bare repo clone — much faster than a full restic/database restore for config-only drift.

## References

- [community.general.git_config module docs](https://docs.ansible.com/ansible/latest/collections/community/general/git_config_module.html)
- [ansible.builtin.command module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/command_module.html)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
