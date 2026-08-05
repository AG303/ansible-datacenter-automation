# backup_restore_verification

**Category:** Disaster Recovery
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Proves backups are actually restorable — not just completed. Restores the most recent backup
(from either `database_backup_restore` or `backup_restic_orchestration`) into an isolated
scratch container/directory that never touches production data, then runs a basic integrity
check: a row-count sanity check for database restores, or a SHA-256 checksum comparison against
the live source files for restic restores. Compiles a pass/fail report and can optionally alert
via webhook. Any secret needed to spin up the scratch environment (e.g. a throwaway DB password)
is resolved from an external secrets manager.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `backup_verify_source` | `database` | `database` or `restic` |
| `backup_verify_container_image` | `postgres:16-alpine` | Scratch container image used for database verification |
| `backup_verify_scratch_data_dir` | see defaults | Host-side scratch working directory |
| `backup_verify_db_engine` | `postgresql` | `mysql` or `postgresql` (database mode) |
| `backup_verify_expected_min_row_count` | `1` | Minimum acceptable row count for the integrity check to pass |
| `backup_verify_restic_repository` | `""` | Restic repository to restore from; falls back to `restic_repository` |
| `backup_verify_restic_checksum_paths` | `[/etc/hosts]` | Files checksummed to prove restic restore fidelity |
| `backup_verify_credential_source` | `vault` | `vault` or `aws_secrets_manager` |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |
| `backup_verify_report_path` | templated with timestamp | Where the JSON verification report is written |
| `backup_verify_notify_webhook_url` | `""` | Optional webhook notified with the pass/fail result |
| `backup_verify_cleanup_after_run` | `true` | Tears down the scratch container/directory after verification |

## Example play

```yaml
- hosts: backup_verification_runners
  become: true
  roles:
    - role: backup_restore_verification
      vars:
        backup_verify_source: database
        backup_verify_db_engine: postgresql
```

## Secrets manager pattern

```yaml
# HashiCorp Vault (community.hashi_vault)
backup_verify_secret_value: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                                 engine_mount_point=vault_kv_mount, url=vault_addr).secret.value }}"

# AWS Secrets Manager (amazon.aws)
backup_verify_secret_value: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- This role depends on the `database_backup_restore` role (included via `ansible.builtin.include_role`
  when `backup_verify_source: database`) to fetch and load the latest dump into the scratch container.
- Runs entirely against a disposable container/scratch directory — production systems are never
  touched, making this safe to schedule frequently (e.g. nightly).
- A failing verification here is a stronger signal than a failing `dr_test_automation` backup-age
  check: it means the backup exists and is fresh, but does not actually restore cleanly.

## References

- [community.docker.docker_container module docs](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_container_module.html)
- [community.postgresql.postgresql_query module docs](https://docs.ansible.com/ansible/latest/collections/community/postgresql/postgresql_query_module.html)
- [restic restore command reference](https://restic.readthedocs.io/en/stable/050_restore.html)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
