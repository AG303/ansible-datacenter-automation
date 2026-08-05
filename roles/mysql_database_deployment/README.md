# mysql_database_deployment

**Category:** Application Deployment
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Installs MySQL or MariaDB (selectable via `mysql_flavor`), templates a `my.cnf` tuning
override file (InnoDB buffer pool/log sizing, slow query log, character set), bootstraps
the root password from an external secrets manager (never inline), removes anonymous
accounts, and creates application databases/users from `mysql_databases`/`mysql_users`
list variables using `community.mysql.mysql_db` and `community.mysql.mysql_user`. Optional
GTID-based streaming replication (source or replica role) is available behind
`mysql_replication_enabled` and uses `community.mysql.mysql_replication`.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `mysql_flavor` | `"mariadb"` | `"mariadb"` or `"mysql"` |
| `mysql_bind_address` | `"127.0.0.1"` | Listener address; widen only with firewall controls in place |
| `mysql_innodb_buffer_pool_size` | `"1G"` | InnoDB buffer pool size |
| `mysql_secrets_backend` | `"hashi_vault"` | `"hashi_vault"` or `"aws_secrets_manager"` — selects which lookup pattern populates `mysql_root_password` |
| `mysql_databases` | `[]` | List of `{name, encoding, collation}` databases to create |
| `mysql_users` | `[]` | List of `{name, host, priv, password_secret_field}` users to create, least-privilege grants |
| `mysql_replication_enabled` | `false` | Opt-in guard for streaming replication config |
| `mysql_replication_role` | `"source"` | `"source"` or `"replica"` |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |

## Example play

```yaml
- hosts: mysql_fleet
  become: true
  roles:
    - role: mysql_database_deployment
      vars:
        mysql_flavor: mariadb
        mysql_databases:
          - name: app_orders
        mysql_users:
          - name: orders_svc
            host: "10.0.%.%"
            priv: "app_orders.*:ALL"
            password_secret_field: orders_svc_password
```

## Secrets manager pattern (root and per-user passwords — never hardcoded)

```yaml
# HashiCorp Vault (community.hashi_vault)
mysql_root_password: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                           engine_mount_point=vault_kv_mount, url=vault_addr).secret.root_password }}"

# AWS Secrets Manager (amazon.aws)
mysql_root_password: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

Per-user passwords follow the same pattern, keyed by each user's `password_secret_field`
under the same secret path.

## Notes

- `no_log: true` is set on every task that handles a password value, so credentials never
  appear in play output or logs.
- The root password task is idempotent against a fresh install (root has no password yet)
  and safe to re-run — it will not clobber a password that has since been rotated
  out-of-band unless the stored Vault/Secrets-Manager value is intentionally updated too.

## References

- [community.mysql collection docs](https://docs.ansible.com/ansible/latest/collections/community/mysql/index.html)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
- [amazon.aws.aws_secret lookup docs](https://docs.ansible.com/ansible/7/collections/amazon/aws/aws_secret_lookup.html)
