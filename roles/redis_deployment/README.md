# redis_deployment

**Category:** Application Deployment
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Installs Redis and templates `redis.conf` (bind address, memory limit and eviction
policy, RDB/AOF/both persistence mode). The `requirepass` authentication secret is always
sourced from an external secrets manager — never inline. Optional Sentinel configuration
(guarded by `redis_sentinel_enabled`) adds a `sentinel.conf` for HA monitoring/failover of
a monitored primary, including replica-of wiring on non-primary nodes.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `redis_bind_address` | `"127.0.0.1"` | Listener address; widen only with firewall controls in place |
| `redis_maxmemory` | `"512mb"` | Max memory before eviction |
| `redis_maxmemory_policy` | `"allkeys-lru"` | Eviction policy |
| `redis_persistence_mode` | `"rdb"` | `"rdb"`, `"aof"`, or `"both"` |
| `redis_secrets_backend` | `"hashi_vault"` | `"hashi_vault"` or `"aws_secrets_manager"` — selects which lookup pattern populates `redis_requirepass` |
| `redis_sentinel_enabled` | `false` | Opt-in guard for Sentinel HA configuration |
| `redis_sentinel_quorum` | `2` | Sentinel quorum for failover decisions |
| `redis_master_host` | `"127.0.0.1"` | Primary node address referenced by replicas/sentinels |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |

## Example play

```yaml
- hosts: redis_fleet
  become: true
  roles:
    - role: redis_deployment
      vars:
        redis_maxmemory: "1gb"
        redis_persistence_mode: both
        redis_sentinel_enabled: true
        redis_master_host: "10.0.1.10"
```

## Secrets manager pattern (requirepass — never hardcoded)

```yaml
# HashiCorp Vault (community.hashi_vault)
redis_requirepass: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                         engine_mount_point=vault_kv_mount, url=vault_addr).secret.requirepass }}"

# AWS Secrets Manager (amazon.aws)
redis_requirepass: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- The rendered `redis.conf` never contains a literal password value in source control —
  `redis_requirepass` is always resolved at runtime from the secrets manager lookup.
- When Sentinel is enabled, any host whose `redis_bind_address` differs from
  `redis_master_host` is templated as a replica of the primary automatically.

## References

- [Redis configuration reference](https://redis.io/docs/latest/operate/oss_and_stack/management/config/)
- [Redis Sentinel documentation](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
