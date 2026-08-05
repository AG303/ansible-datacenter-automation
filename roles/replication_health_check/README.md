# replication_health_check

**Category:** Disaster Recovery
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Checks database replication lag — MySQL via `SHOW REPLICA STATUS` (`Seconds_Behind_Source`) or
PostgreSQL via `pg_last_xact_replay_timestamp()` — against `replication_lag_warning_seconds`,
and filesystem/storage replication freshness by comparing the most recently modified file
timestamp on the primary path against the same path on a DR-site mirror host against
`replication_fs_max_staleness_seconds`. Compiles a JSON report, alerts via webhook when a
threshold is exceeded, and optionally fails the play for CI/alerting integration.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `replication_check_database` / `replication_check_filesystem` | `true` | Toggle each check independently |
| `replication_db_engine` | `postgresql` | `mysql` or `postgresql` |
| `replication_db_host` / `_port` / `_name` / `_user` | see defaults | Database connection details |
| `replication_credential_source` | `vault` | `vault` or `aws_secrets_manager` |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |
| `replication_lag_warning_seconds` | `60` | Threshold for database replication lag pass/fail |
| `replication_fs_primary_path` / `_mirror_path` | see defaults | Paths compared for filesystem mirror freshness |
| `replication_fs_mirror_hosts_group` | `dr_site` | Inventory group the mirror path check delegates to |
| `replication_fs_max_staleness_seconds` | `900` | Threshold for filesystem mirror staleness pass/fail |
| `replication_report_path` | templated with timestamp | Where the JSON report is written |
| `replication_notify_webhook_url` | `""` | Optional webhook notified when a threshold is exceeded |
| `replication_fail_on_lag_exceeded` | `true` | Fails the Ansible play if any threshold is exceeded |

## Example play

```yaml
- hosts: db_servers
  become: true
  roles:
    - role: replication_health_check
      vars:
        replication_db_engine: postgresql
        replication_lag_warning_seconds: 30
```

## Secrets manager pattern

```yaml
# HashiCorp Vault (community.hashi_vault)
replication_db_password: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                              engine_mount_point=vault_kv_mount, url=vault_addr).secret.password }}"

# AWS Secrets Manager (amazon.aws)
replication_db_password: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- This role only reads state; it never mutates the database or filesystem.
- Run on a tight schedule (every 1-5 minutes) against primary/standby pairs for early lag
  detection ahead of a real failover decision made by `dr_failover_orchestration`.

## References

- [MySQL SHOW REPLICA STATUS reference](https://dev.mysql.com/doc/refman/8.0/en/show-replica-status.html)
- [PostgreSQL replication monitoring functions](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-RECOVERY-INFO-TABLE)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
