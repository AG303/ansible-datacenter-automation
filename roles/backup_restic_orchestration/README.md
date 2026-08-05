# backup_restic_orchestration

**Category:** Disaster Recovery
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Installs `restic` (via distro package or a pinned upstream binary release), initializes an
S3-compatible backup repository on first run (or reuses an existing one), runs backups of a
configurable list of filesystem paths, applies `--keep-daily/--keep-weekly/--keep-monthly/
--keep-yearly` retention with `forget --prune`, and verifies repository integrity with
`restic check`. The repository encryption password is never hardcoded — it is resolved at
runtime from HashiCorp Vault or AWS Secrets Manager.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `restic_backup_paths` | `[/etc, /var/lib, /home]` | Filesystem paths to back up |
| `restic_backup_excludes` | see defaults | Glob/path exclude patterns |
| `restic_repository` | S3 URL templated with `aws_region`/`inventory_hostname` | Restic repository target (S3-compatible) |
| `restic_s3_access_key` / `restic_s3_secret_key` | from environment | S3 credentials for the repository backend |
| `restic_password_source` | `vault` | `vault` or `aws_secrets_manager` — selects the secret lookup path |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |
| `restic_keep_daily`/`weekly`/`monthly`/`yearly` | `7`/`4`/`6`/`2` | Retention policy passed to `restic forget` |
| `restic_prune_after_forget` | `true` | Adds `--prune` to reclaim space immediately after forget |
| `restic_backup_action` | `backup` | `backup`, `forget`, `check`, or `unlock` |
| `restic_run_check_after_backup` | `true` | Runs `restic check` after a successful backup |
| `restic_check_read_data_subset` | `"5%"` | Partial data-read verification percentage for routine checks |
| `restic_install_method` | `package` | `package` (distro repo) or `binary` (pinned `restic_version`) |
| `restic_notify_webhook_url` | `""` | Optional webhook (Slack/Teams) notified after backup completion |

## Example play

```yaml
- hosts: backup_targets
  become: true
  roles:
    - role: backup_restic_orchestration
      vars:
        restic_backup_action: backup
        restic_repository: "s3:https://s3.us-east-1.amazonaws.com/dc-backups-bucket/{{ inventory_hostname }}"
        restic_password_source: vault
```

## Secrets manager pattern

```yaml
# HashiCorp Vault (community.hashi_vault)
restic_repository_password: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                                 engine_mount_point=vault_kv_mount, url=vault_addr).secret.repository_password }}"

# AWS Secrets Manager (amazon.aws)
restic_repository_password: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- `restic_backup_action: unlock` should only be run manually after confirming no other backup
  process is genuinely holding the lock — stale locks are the common case after a killed job.
- Pair this role with `backup_restore_verification` on a schedule to prove restorability, not
  just backup completion.

## References

- [restic official documentation](https://restic.readthedocs.io/en/stable/)
- [restic forget command reference](https://restic.readthedocs.io/en/stable/060_forget.html)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
- [amazon.aws.aws_secret lookup docs](https://docs.ansible.com/ansible/7/collections/amazon/aws/aws_secret_lookup.html)
