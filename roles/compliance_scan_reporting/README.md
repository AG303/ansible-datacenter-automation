# compliance_scan_reporting

**Category:** Security & Compliance
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Installs and runs `lynis` (default, cross-distro) or `openscap-scanner` (RedHat-family,
best-effort on Debian) as a general system-hygiene audit tool, captures the raw scan report
and log locally under `compliance_scan_report_dir`, parses a simple pass/warn/fail-style
summary from the lynis log, and pushes that summary to a centralized webhook or S3 bucket
for fleet-wide tracking. This is general hygiene scanning — it is **not** tied to any named
certification and does not label findings with CIS/STIG control IDs.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `compliance_scan_tool` | `lynis` | `lynis` or `openscap_scanner` |
| `compliance_scan_report_dir` | `/var/log/compliance-scans` | Local report/log storage directory |
| `compliance_scan_profile` | `default` | oscap XCCDF profile id (ignored for lynis) |
| `compliance_report_delivery` | `webhook` | `webhook` or `s3` |
| `compliance_report_webhook_url` | `""` | Webhook endpoint to POST the JSON summary to |
| `compliance_report_s3_bucket` / `_prefix` | `""` / `lynis-reports` | S3 destination for the raw report file |
| `aws_region` | `us-east-1` | Region for S3 push / AWS Secrets Manager lookup |
| `compliance_webhook_auth_from_vault_enabled` | `false` | Pull a bearer token from Vault/AWS Secrets Manager for webhook auth |
| `compliance_scan_retain_reports` | `10` | Number of local lynis report files retained before pruning |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: compliance_scan_reporting
      vars:
        compliance_report_delivery: webhook
        compliance_report_webhook_url: "https://siem.internal.example.com/hooks/compliance-scan"
        compliance_webhook_auth_from_vault_enabled: true
```

## Secrets manager pattern (if `compliance_webhook_auth_from_vault_enabled: true`)

```yaml
# HashiCorp Vault (community.hashi_vault)
compliance_webhook_bearer_token: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                                     engine_mount_point=vault_kv_mount, url=vault_addr).secret.token }}"

# AWS Secrets Manager (amazon.aws)
compliance_webhook_bearer_token: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- The pass/warn/fail summary derived from lynis is a simple heuristic (counts of `Warning:`
  and `Suggestion:` lines in the run log) intended for fleet-wide trend tracking, not a
  certified compliance score.
- `openscap_scanner` SCAP content (`scap-security-guide`) is primarily published for
  RHEL-family distributions; on Debian family only the `libopenscap8` library/CLI is
  installed and compatible SCAP content must be supplied separately if this tool is selected.

## References

- [Lynis documentation](https://cisofy.com/documentation/lynis/)
- [OpenSCAP documentation](https://www.open-scap.org/tools/openscap-base/)
- [ansible.builtin.uri module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html)
- [amazon.aws.s3_object module docs](https://docs.ansible.com/ansible/latest/collections/amazon/aws/s3_object_module.html)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
