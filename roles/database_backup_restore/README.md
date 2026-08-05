# database_backup_restore

**Category:** Disaster Recovery
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Two-mode role (`db_backup_action: backup|restore`) for MySQL and PostgreSQL. In backup mode it
runs `mysqldump`/`pg_dump` to a timestamped, gzip-compressed file and ships it to an S3 bucket
or NFS backup destination, then prunes local dumps past `db_dump_retention_days`. In restore
mode it pulls the latest (or a named) dump from the backup destination and loads it back with
`mysql`/`pg_restore`, gated behind an explicit confirmation flag and an interactive pause.
Database credentials are always resolved from an external secrets manager, never hardcoded.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `db_backup_action` | `backup` | `backup` or `restore` |
| `db_engine` | `mysql` | `mysql` or `postgresql` |
| `db_host` / `db_port` / `db_name` / `db_admin_user` | see defaults | Connection details |
| `db_credential_source` | `vault` | `vault` or `aws_secrets_manager` |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |
| `db_dump_local_dir` | `/var/backups/db-dumps` | Local staging directory for dumps |
| `db_dump_retention_days` | `14` | Local dump retention window before purge |
| `db_backup_destination_type` | `s3` | `s3` or `nfs` |
| `db_backup_s3_bucket` / `db_backup_s3_prefix` | see defaults | S3 destination coordinates |
| `db_backup_nfs_path` | `/mnt/dr-backups/db/{{ inventory_hostname }}` | NFS destination path |
| `db_restore_target_file` | `""` | Explicit dump filename to restore; empty pulls the latest |
| `db_restore_confirm` | `false` | Must be explicitly set `true` to allow a restore to run |
| `db_restore_drop_existing` | `false` | Drops/recreates the target database before loading the dump |

## Example play

```yaml
# Backup
- hosts: db_servers
  become: true
  roles:
    - role: database_backup_restore
      vars:
        db_backup_action: backup
        db_engine: postgresql
        db_name: app_production

# Restore (requires explicit confirmation)
- hosts: db_servers
  become: true
  roles:
    - role: database_backup_restore
      vars:
        db_backup_action: restore
        db_engine: postgresql
        db_name: app_production
        db_restore_confirm: true
```

## Secrets manager pattern

```yaml
# HashiCorp Vault (community.hashi_vault)
db_admin_password: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                         engine_mount_point=vault_kv_mount, url=vault_addr).secret.password }}"

# AWS Secrets Manager (amazon.aws)
db_admin_password: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- Restore is destructive: the role asserts `db_restore_confirm: true` and then pauses for an
  interactive operator confirmation (`ansible.builtin.pause`) before touching data.
- Combine with `backup_restore_verification` to periodically prove a dump actually restores
  cleanly into a scratch environment.

## References

- [community.mysql.mysql_db module docs](https://docs.ansible.com/ansible/latest/collections/community/mysql/mysql_db_module.html)
- [community.postgresql.postgresql_db module docs](https://docs.ansible.com/ansible/latest/collections/community/postgresql/postgresql_db_module.html)
- [amazon.aws.s3_object module docs](https://docs.ansible.com/ansible/latest/collections/amazon/aws/s3_object_module.html)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
