# dr_test_automation

**Category:** Disaster Recovery
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Runs a non-destructive DR readiness checklist meant to be scheduled regularly (e.g. daily via
cron/AWX) to validate disaster-recovery posture without performing an actual failover. It checks:
DR site reachability (HTTP health endpoints), latest backup age against `dr_max_backup_age_hours`
(via a manifest endpoint, S3 listing, or NFS file age), database/storage replication lag against
`dr_replication_lag_warning_seconds`, and the failover DNS record's TTL. Results are compiled into
a JSON pass/fail report written to disk and optionally posted to a notification webhook.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `dr_health_check_urls` | see defaults | Endpoints checked for DR-site reachability |
| `dr_max_backup_age_hours` | `26` | Maximum acceptable age of the latest backup |
| `dr_backup_manifest_urls` | see defaults | HTTP endpoint(s) returning `{"last_backup_iso8601": ...}` |
| `dr_backup_check_destination_type` | `s3` | `s3` or `nfs` — alternate backup-age check source |
| `dr_replication_lag_warning_seconds` | `60` | Threshold for replication lag pass/fail |
| `dr_replication_status_url` | `""` | Optional HTTP endpoint returning `{"lag_seconds": N}` |
| `dr_dns_record_name` / `dr_dns_expected_ttl_seconds` | see defaults | DNS TTL check parameters |
| `dr_test_report_path` | templated with timestamp | Where the JSON readiness report is written |
| `dr_test_notify_webhook_url` | `""` | Optional webhook notified with the overall pass/fail result |
| `dr_test_fail_on_any_check_failure` | `true` | Fails the Ansible play if any check fails (for CI/alerting integration) |

## Example play

```yaml
- hosts: dr_site
  become: true
  roles:
    - role: dr_test_automation
      vars:
        dr_max_backup_age_hours: 26
        dr_replication_lag_warning_seconds: 30
```

## Notes

- This role never mutates state — it is safe to run on a schedule against production and DR
  hosts alike with no risk of unintended side effects.
- Pair with `replication_health_check` for deeper per-engine replication diagnostics beyond the
  simple pass/fail threshold check performed here.
- Set `dr_test_fail_on_any_check_failure: false` if you want the report generated but do not
  want the Ansible run itself to fail (e.g. when a downstream alerting system consumes the report).

## References

- [ansible.builtin.uri module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html)
- [amazon.aws.s3_object module docs](https://docs.ansible.com/ansible/latest/collections/amazon/aws/s3_object_module.html)
- [ansible.builtin.find module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/find_module.html)
