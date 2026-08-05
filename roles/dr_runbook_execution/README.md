# dr_runbook_execution

**Category:** Disaster Recovery
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

The single top-level DR orchestration entry point. Chains the other DR roles in the correct
order — health check (`dr_test_automation`) → snapshot latest state (`lvm_snapshot_management`)
→ failover (`dr_failover_orchestration`) → post-failover verification
(`backup_restore_verification` / `replication_health_check`) → notification — behind one
`dr_runbook_action: test|execute` switch. In `test` mode every step runs in its safe/read-only
form and the failover step is skipped entirely with a clear log message. In `execute` mode the
role requires `dr_failover_confirm: true` and pauses for an explicit interactive operator
confirmation before the failover step (a genuinely destructive DR action) is allowed to run.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `dr_runbook_action` | `test` | `test` (safe dry-run of the checklist) or `execute` (real failover) |
| `dr_site_group` | `dr_site` | Inventory group representing the DR-site hosts |
| `dr_runbook_run_health_check` | `true` | Toggle step 1 |
| `dr_runbook_run_snapshot` | `true` | Toggle step 2 |
| `dr_runbook_snapshot_lvm_volumes` | see defaults | Volumes snapshotted in step 2 |
| `dr_runbook_run_failover` | `true` | Toggle step 3 (only ever executes when `dr_runbook_action == execute`) |
| `dr_runbook_promotion_method` | `replica_command` | Passed through to `dr_failover_orchestration` |
| `dr_runbook_run_verification` | `true` | Toggle step 4 |
| `dr_runbook_notify_webhook_url` | `""` | Optional webhook notified on runbook completion |
| `dr_runbook_report_path` | templated with timestamp | Where the JSON runbook execution report is written |

## Example play

```yaml
# Safe DR test run (no destructive action)
- hosts: dr_site
  become: true
  roles:
    - role: dr_runbook_execution
      vars:
        dr_runbook_action: test

# Real DR execution (destructive — requires explicit confirmation)
- hosts: dr_site
  become: true
  serial: 1
  roles:
    - role: dr_runbook_execution
      vars:
        dr_runbook_action: execute
        dr_failover_confirm: true
```

## Notes

- This role is a thin orchestrator: it delegates to `dr_test_automation`, `lvm_snapshot_management`,
  `dr_failover_orchestration`, `backup_restore_verification`, and `replication_health_check` via
  `ansible.builtin.include_role` — see each role's own README for its full variable surface.
- The `execute` path is guarded twice: an `ansible.builtin.assert` on `dr_failover_confirm` and an
  `ansible.builtin.pause` confirmation gate, matching the same pattern used by
  `dr_failover_orchestration` and `etcd_cluster_backup_restore`.
- Always run `dr_runbook_action: test` on a schedule to keep DR readiness continuously validated;
  reserve `execute` for real incidents or planned DR drills with a rollback plan in hand.

## References

- [ansible.builtin.include_role module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/include_role_module.html)
- [ansible.builtin.pause module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/pause_module.html)
- [ansible.builtin.assert module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html)
