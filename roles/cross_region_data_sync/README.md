# cross_region_data_sync

**Category:** Disaster Recovery
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Periodically syncs application data directories from primary-site hosts to DR-site hosts.
Supports two transport tools via `cross_region_sync_tool`: `rsync` (via
`ansible.posix.synchronize` over SSH, honoring bandwidth limits and exclude patterns) for
filesystem-to-filesystem replication, or `rclone` for object-storage-backed cross-region sync
between S3-compatible buckets. Both modes produce a sync-lag report comparing source and
destination freshness. The SSH key used for rsync transport is resolved from an external
secrets manager, never hardcoded.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `cross_region_sync_tool` | `rsync` | `rsync` or `rclone` |
| `cross_region_sync_paths` | see defaults | List of `{src, dest}` pairs synced (rsync mode) |
| `cross_region_dest_host_group` | `dr_site` | Inventory group of DR-site destination hosts |
| `cross_region_rsync_bwlimit_kbps` | `20000` | rsync bandwidth limit in KB/s; `0` = unlimited |
| `cross_region_rsync_excludes` | see defaults | Glob exclude patterns applied to both tools |
| `cross_region_rsync_delete` | `false` | Mirror deletions from source to destination |
| `cross_region_rclone_remote_source` / `_dest` | see defaults | rclone remote:bucket coordinates |
| `cross_region_rclone_bwlimit` | `"20M"` | rclone bandwidth limit |
| `cross_region_credential_source` | `vault` | `vault` or `aws_secrets_manager` |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |
| `cross_region_max_sync_lag_seconds` | `1800` | Threshold for the sync-lag report pass/fail and alerting |
| `cross_region_notify_webhook_url` | `""` | Optional webhook notified when sync lag exceeds threshold |

## Example play

```yaml
- hosts: primary_site
  become: true
  roles:
    - role: cross_region_data_sync
      vars:
        cross_region_sync_tool: rsync
        cross_region_sync_paths:
          - { src: /data/app/, dest: /data/app/ }
        cross_region_dest_host_group: dr_site
        cross_region_rsync_bwlimit_kbps: 15000
```

## Secrets manager pattern

```yaml
# HashiCorp Vault (community.hashi_vault)
cross_region_ssh_private_key: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                                   engine_mount_point=vault_kv_mount, url=vault_addr).secret.private_key }}"

# AWS Secrets Manager (amazon.aws)
cross_region_ssh_private_key: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- Always set a sane `cross_region_rsync_bwlimit_kbps` on WAN links between regions to avoid
  saturating the inter-site link during business hours.
- `cross_region_rsync_delete: true` mirrors deletions — verify this is genuinely desired before
  enabling it, since it can propagate accidental source deletions to the DR copy.
- Run on a tight schedule; the sync-lag report is only as fresh as the last successful sync run.

## References

- [ansible.posix.synchronize module docs](https://docs.ansible.com/ansible/latest/collections/ansible/posix/synchronize_module.html)
- [rclone sync command reference](https://rclone.org/commands/rclone_sync/)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
