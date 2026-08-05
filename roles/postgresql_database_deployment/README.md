# postgresql_database_deployment

**Category:** Application Deployment
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Installs PostgreSQL from the official PGDG repository (RedHat via `dnf` RPM install +
disabling the distro's built-in `postgresql` module to avoid version conflicts, Debian via
`apt_repository` with a keyring-based `signed-by` pin), initializes the cluster, templates
`postgresql.conf` and `pg_hba.conf` from role variables, and manages databases/roles with
`community.postgresql.postgresql_db` / `postgresql_user`. The superuser password and every
per-role password are sourced from an external secrets manager — never hardcoded. Optional
physical streaming replication (primary or standby) is available behind
`postgresql_replication_enabled` using replication slots and `pg_basebackup`.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `postgresql_version` | `"16"` | PostgreSQL major version installed from PGDG |
| `postgresql_listen_addresses` | `"localhost"` | Listener address; widen only with `pg_hba.conf` + firewall controls in place |
| `postgresql_shared_buffers` / `effective_cache_size` | `256MB` / `1GB` | Core memory tuning |
| `postgresql_hba_entries` | localhost-only defaults | List of `{type, database, user, address, method}` access rules |
| `postgresql_secrets_backend` | `"hashi_vault"` | `"hashi_vault"` or `"aws_secrets_manager"` |
| `postgresql_databases` | `[]` | List of `{name, owner, encoding}` databases to create |
| `postgresql_roles` | `[]` | List of `{name, password_secret_field, role_attr_flags}` roles to create |
| `postgresql_replication_enabled` | `false` | Opt-in guard for streaming replication |
| `postgresql_replication_role` | `"primary"` | `"primary"` or `"standby"` |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |

## Example play

```yaml
- hosts: postgresql_fleet
  become: true
  roles:
    - role: postgresql_database_deployment
      vars:
        postgresql_version: "16"
        postgresql_databases:
          - name: app_orders
            owner: orders_svc
        postgresql_roles:
          - name: orders_svc
            password_secret_field: orders_svc_password
            role_attr_flags: "LOGIN"
```

## Secrets manager pattern (superuser and per-role passwords — never hardcoded)

```yaml
# HashiCorp Vault (community.hashi_vault)
postgresql_superuser_password: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                                     engine_mount_point=vault_kv_mount, url=vault_addr).secret.superuser_password }}"

# AWS Secrets Manager (amazon.aws)
postgresql_superuser_password: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- `no_log: true` is set on every task that handles a password value.
- The distro-shipped `postgresql` dnf module is disabled on RedHat family hosts specifically
  to avoid version conflicts with the PGDG-provided packages this role installs.
- Streaming replication uses a physical replication slot; verify WAL retention/disk headroom
  on the primary before enabling it against a standby that may lag or disconnect.

## References

- [community.postgresql collection docs](https://docs.ansible.com/ansible/latest/collections/community/postgresql/index.html)
- [PostgreSQL streaming replication docs](https://www.postgresql.org/docs/current/warm-standby.html#STREAMING-REPLICATION)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
