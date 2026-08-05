# lvm_snapshot_management

**Category:** Disaster Recovery
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Creates, removes, and reverts LVM snapshots of specified volume groups/logical volumes —
typically run immediately before a risky patch or deployment so a fast rollback point exists.
`snapshot_action: create` runs a free-space pre-check assertion (`lvm_min_free_percent_required`)
before creating each snapshot via `community.general.lvol` and records retention metadata.
`snapshot_action: remove` purges an explicitly named snapshot or all snapshots past
`lvm_snapshot_retention_days`. `snapshot_action: revert` merges a snapshot back into its origin
volume — a destructive operation gated by an explicit confirmation flag and an interactive pause.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `snapshot_action` | `create` | `create`, `remove`, or `revert` |
| `lvm_volumes` | see defaults | List of `{vg, lv}` pairs to manage snapshots for |
| `lvm_snapshot_suffix` | `-snap-<timestamp>` | Suffix appended to the origin LV name for the snapshot LV |
| `lvm_snapshot_size` | `"20%ORIGIN"` | Snapshot size/percentage of origin extents |
| `lvm_min_free_percent_required` | `25` | Pre-check: VG must have at least this % free extents before create |
| `lvm_snapshot_retention_days` | `7` | Age threshold for automatic snapshot purge |
| `lvm_revert_confirm` | `false` | Must be explicitly `true` to allow a revert (merge) to proceed |
| `lvm_revert_snapshot_name` | `""` | Explicit snapshot LV name to revert; empty reverts the latest per origin |
| `lvm_remove_snapshot_name` | `""` | Explicit snapshot LV name to remove; empty purges all expired snapshots |

## Example play

```yaml
# Before a risky patch/deploy
- hosts: db_servers
  become: true
  roles:
    - role: lvm_snapshot_management
      vars:
        snapshot_action: create
        lvm_volumes:
          - { vg: vg_data, lv: lv_db }

# Guarded revert after a failed change
- hosts: db_servers
  become: true
  roles:
    - role: lvm_snapshot_management
      vars:
        snapshot_action: revert
        lvm_revert_confirm: true
        lvm_volumes:
          - { vg: vg_data, lv: lv_db }
```

## Notes

- `revert` is destructive: it discards all writes made since the snapshot was taken. The role
  asserts `lvm_revert_confirm: true` and then pauses for interactive operator confirmation.
- Always confirm `lvm_min_free_percent_required` matches your storage team's guidance — a
  snapshot that fills up its allocated extents will be dropped by the kernel automatically.
- Pair with `os_patch_rolling_update` or `patch_rollback_snapshot` as a pre-change safety net.

## References

- [community.general.lvol module docs](https://docs.ansible.com/ansible/latest/collections/community/general/lvol_module.html)
- [LVM snapshot documentation (Red Hat)](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_and_managing_logical_volumes/creating-and-managing-thin-provisioned-snapshots_configuring-and-managing-logical-volumes)
- [ansible.posix.mount module docs](https://docs.ansible.com/ansible/latest/collections/ansible/posix/mount_module.html)
