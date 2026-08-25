# patch_rollback_snapshot

**Category:** Patch Management
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Provides a two-mode safety net around patching. In **`snapshot`** mode (run
before patching), it records every currently-installed package and its exact
version to a JSON fact file on the managed host, then archives a durable copy
to a persistent Linux host (`patch_rollback_archive_host`) via
`ansible.builtin.fetch` + `ansible.builtin.copy` — **not** to `localhost`,
since Ansible Automation Platform 2.6 execution nodes are ephemeral pods with
no durable storage of their own. It optionally also takes an LVM snapshot of
the root/var logical volume if the host is LVM-backed. In
**`rollback`** mode (run after patching, typically gated on a failed health
check), it reads the recorded fact file back and downgrades the affected
packages to their previously-installed versions using `allow_downgrade` on
`ansible.builtin.dnf` / `ansible.builtin.apt`, then optionally removes the
now-unneeded LVM snapshot once health is confirmed restored.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `patch_rollback_mode` | `"snapshot"` | `"snapshot"` (pre-patch recording) or `"rollback"` (post-patch downgrade) |
| `patch_rollback_state_dir` | `/var/lib/ansible-patch-rollback` | On-host directory for the recorded fact file |
| `patch_rollback_state_file` | derived path | Per-host JSON fact file of installed package versions |
| `patch_rollback_archive_host` | `"{{ report_archive_host }}"` | Persistent Linux host that receives the durable snapshot copy (AAP 2.6 execution nodes are ephemeral — never `localhost`) |
| `patch_rollback_archive_dir` | `{{ report_archive_base_dir }}/rollback_state` | Directory on `patch_rollback_archive_host` that stores the durable per-host copy |
| `patch_rollback_local_fetch_stage` | `/tmp/ansible_rollback_fetch_stage` | Transient staging path on the execution node used only during the `fetch` → `copy` hop; cleaned up at the end of the same play |
| `patch_rollback_lvm_enabled` | `false` | Enable LVM snapshot/restore (only for LVM-backed hosts) |
| `patch_rollback_lvm_vg` / `_lv` / `_snapshot_name` / `_snapshot_size` | see defaults | LVM snapshot coordinates |
| `patch_rollback_health_check_service` | `"sshd"` | Service checked to decide whether rollback is triggered |
| `patch_rollback_packages_scope` | `[]` | Restrict downgrade to specific packages; empty = downgrade everything recorded |

## Example play

```yaml
# Before patching
- hosts: linux_fleet
  become: true
  roles:
    - role: patch_rollback_snapshot
      vars:
        patch_rollback_mode: snapshot
        patch_rollback_lvm_enabled: true

# After patching, only if health checks failed
- hosts: linux_fleet
  become: true
  roles:
    - role: patch_rollback_snapshot
      vars:
        patch_rollback_mode: rollback
```

## Secrets manager pattern

This role does not require any secrets — it records/reads local package
facts and manages local LVM volumes. No Vault/AWS Secrets Manager lookups
are used.

## Notes

- On Ansible Automation Platform 2.6+, execution nodes are ephemeral pods, so
  the durable copy of the snapshot can never land on `localhost`/the control
  node — it is fetched to a transient local staging path for the duration of
  the play only, then pushed via `ansible.builtin.copy` (`delegate_to:
  "{{ patch_rollback_archive_host }}"`) to a real, persistent host, and the
  staging copy is removed immediately after. Repoint every role in this
  collection at a different archive host by overriding the single
  `report_archive_host` variable in `inventory/group_vars/all.yml`.
- Always run this role in `snapshot` mode immediately before
  `os_patch_rolling_update`, `security_only_patching`, or
  `kernel_update_reboot` so a rollback path is available.
- LVM snapshot mode is opt-in (`patch_rollback_lvm_enabled: false` by
  default) because not every host in a mixed fleet is LVM-backed — verify
  volume layout with `ansible_facts['lvm']` or `pvs`/`vgs` before enabling.
- The rollback task path only executes package downgrades when the
  configured health check is failing, preventing accidental downgrades on
  a healthy host.

## References

- [ansible.builtin.dnf module docs (allow_downgrade)](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/dnf_module.html)
- [ansible.builtin.apt module docs (allow_downgrade)](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)
- [community.general.lvol module docs](https://docs.ansible.com/ansible/latest/collections/community/general/lvol_module.html)
- [ansible.builtin.fetch module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/fetch_module.html)
- [ansible.builtin.copy module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html)
- [Ansible delegate_to documentation](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_delegation.html)
