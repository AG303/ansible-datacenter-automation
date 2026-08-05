# incident_snapshot_capture

**Category:** Disaster Recovery
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

On-demand diagnostic capture role for incident postmortems. Gathers top CPU/memory processes,
disk and memory usage, recent `journalctl` lines, active network connections/listening sockets
(`ss -tunapl`), and the status of a configurable list of services of interest — all into a
single timestamped `tar.gz` archive per host. The archive can optionally be uploaded to the
backup destination (S3 or NFS) for centralized incident-response review. The role is strictly
read-only: it never restarts a service or mutates any system state.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `incident_capture_top_processes` / `_disk_memory` / `_journal_lines` / `_network_connections` / `_service_statuses` | `true` | Toggle each capture category independently |
| `incident_capture_top_process_count` | `25` | Number of top processes captured by CPU/memory |
| `incident_capture_journal_line_count` / `_since` | `500` / `"1 hour ago"` | Journal capture window |
| `incident_capture_services_of_interest` | see defaults | Services whose `systemctl status` is captured |
| `incident_capture_local_dir` | `/var/tmp/incident-snapshot-capture` | Local staging/archive directory |
| `incident_capture_upload_enabled` | `true` | Whether to upload the archive to the backup destination |
| `incident_capture_backup_destination_type` | `s3` | `s3` or `nfs` |
| `incident_capture_s3_bucket` / `_prefix` | see defaults | S3 destination coordinates |
| `incident_capture_nfs_path` | see defaults | NFS destination path |
| `incident_capture_retain_local_archive` | `true` | Whether to keep a local copy after upload |
| `incident_capture_local_retention_days` | `30` | Local archive retention before purge |

## Example play

```yaml
- hosts: "{{ incident_hosts | default('all') }}"
  become: true
  roles:
    - role: incident_snapshot_capture
      vars:
        incident_capture_services_of_interest:
          - nginx
          - postgresql
          - docker
```

## Notes

- Designed to be run ad hoc against a specific host or group during an active incident
  (`ansible-playbook ... -l incident_hosts`) rather than on a fixed schedule.
- Captures are read-only diagnostics; this role is always safe to run against a production host
  under active investigation without risk of side effects.
- Pair the resulting archive with the reports from `dr_test_automation` and
  `replication_health_check` for a complete incident timeline.

## References

- [community.general.archive module docs](https://docs.ansible.com/ansible/latest/collections/community/general/archive_module.html)
- [amazon.aws.s3_object module docs](https://docs.ansible.com/ansible/latest/collections/amazon/aws/s3_object_module.html)
- [journalctl manual](https://man7.org/linux/man-pages/man1/journalctl.1.html)
