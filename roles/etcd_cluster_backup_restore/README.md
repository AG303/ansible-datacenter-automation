# etcd_cluster_backup_restore

**Category:** Disaster Recovery
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Manages etcd cluster state backup and recovery for Kubernetes control-plane nodes. In
`snapshot` mode it runs `etcdctl snapshot save` using cert paths from `vars`, verifies the
snapshot with `etcdctl snapshot status`, and uploads it to the configured backup destination
(S3 or NFS). In `restore` mode it performs a fully guarded, manual-confirmation
`etcdctl snapshot restore` into a new data directory, swaps it into place, and validates
cluster health — intended strictly for DR recovery of a lost/corrupted control plane.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `etcd_action` | `snapshot` | `snapshot` or `restore` |
| `etcd_endpoints` | `https://127.0.0.1:2379` | etcdctl endpoint(s) |
| `etcd_ca_file` / `etcd_cert_file` / `etcd_key_file` | under `etcd_cert_dir` | TLS material for etcdctl |
| `etcd_snapshot_dir` | `/var/backups/etcd` | Local staging directory for snapshots |
| `etcd_snapshot_retention_days` | `14` | Local snapshot retention before purge |
| `etcd_backup_destination_type` | `s3` | `s3` or `nfs` |
| `etcd_backup_s3_bucket` / `etcd_backup_s3_prefix` | see defaults | S3 destination coordinates |
| `etcd_backup_nfs_path` | see defaults | NFS destination path |
| `etcd_restore_confirm` | `false` | Must be explicitly `true` to allow the restore path to run |
| `etcd_restore_source_file` | `""` | Explicit snapshot filename to restore; empty pulls the latest |
| `etcd_restore_new_data_dir` | `/var/lib/etcd-restored` | Staging data dir etcdctl writes the restore into |
| `etcd_stop_service_before_restore` | `true` | Stops the etcd service prior to swapping data dirs |
| `etcd_credential_source` | `vault` | `vault` or `aws_secrets_manager` (for any client-cert secret material) |

## Example play

```yaml
# Routine snapshot (safe, non-destructive)
- hosts: k8s_control_plane
  become: true
  roles:
    - role: etcd_cluster_backup_restore
      vars:
        etcd_action: snapshot

# Guarded DR restore (destructive — requires explicit confirmation)
- hosts: k8s_control_plane
  become: true
  roles:
    - role: etcd_cluster_backup_restore
      vars:
        etcd_action: restore
        etcd_restore_confirm: true
```

## Secrets manager pattern

```yaml
# HashiCorp Vault (community.hashi_vault)
etcd_client_cert_passphrase: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                                  engine_mount_point=vault_kv_mount, url=vault_addr).secret.passphrase }}"

# AWS Secrets Manager (amazon.aws)
etcd_client_cert_passphrase: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- The restore path is guarded by both an `ansible.builtin.assert` on `etcd_restore_confirm`
  and an `ansible.builtin.pause` confirmation gate — this is a full control-plane recovery
  action and should only be run after confirming quorum loss or data corruption.
- Run this role serially, one control-plane node at a time, and validate cluster health with
  `replication_health_check` before proceeding to the next node.

## References

- [etcd disaster recovery documentation](https://etcd.io/docs/v3.5/op-guide/recovery/)
- [etcdctl snapshot command reference](https://etcd.io/docs/v3.5/op-guide/maintenance/#snapshot-backup)
- [amazon.aws.s3_object module docs](https://docs.ansible.com/ansible/latest/collections/amazon/aws/s3_object_module.html)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
