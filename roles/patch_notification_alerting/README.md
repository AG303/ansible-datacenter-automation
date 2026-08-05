# patch_notification_alerting

**Category:** Patch Management
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12
(OS-agnostic aggregation/notification role — no package/service management)

## Purpose

Closes the loop on a patch run by aggregating per-host results — packages
updated, reboot-required status, success/fail — across every host in the
play (`ansible_play_hosts_all` + `hostvars`) and POSTing a single JSON
summary to a configurable webhook (Slack, Microsoft Teams, or a generic JSON
receiver) using `ansible.builtin.uri`, executed once at play end via
`delegate_to: localhost` / `run_once: true`. The webhook URL is treated as a
secret (it typically embeds a bearer token) and is resolved from HashiCorp
Vault or AWS Secrets Manager by default rather than hardcoded, with
`no_log: true` on every task that touches the URL or payload.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `patch_notify_webhook_enabled` | `true` | Master on/off switch for the webhook POST |
| `patch_notify_webhook_format` | `"slack"` | `"slack"`, `"teams"`, or `"generic"` — controls payload shape |
| `patch_notify_webhook_timeout` | `15` | HTTP timeout (seconds) for the webhook POST |
| `patch_notify_webhook_from_vault` | `true` | Resolve the webhook URL from the configured secrets backend instead of a literal var |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates |
| `aws_region` | `us-east-1` | Region for the AWS Secrets Manager lookup pattern |
| `patch_notify_webhook_url_override` | `""` | Direct URL fallback, only used when `patch_notify_webhook_from_vault: false` — never commit a real value here |
| `patch_notify_run_label` | `"ad-hoc patch run"` | Human-readable label included in the notification (set via `patch_run_label` extra-var from the calling playbook) |
| `patch_notify_include_package_diff` | `true` | Reserved flag for including a detailed per-package diff in the payload |

## Example play

```yaml
- name: Patch fleet and notify Slack of the outcome
  hosts: "{{ target_hosts | default('all') }}"
  become: true
  gather_facts: true
  serial: "{{ rolling_batch_size | default('30%') }}"
  roles:
    - role: os_patch_rolling_update
    - role: patch_notification_alerting
      vars:
        patch_notify_run_label: "os_patch_rolling_update - {{ target_hosts | default('all') }}"
        patch_notify_webhook_format: slack
```

## Secrets manager pattern (webhook URL)

```yaml
# HashiCorp Vault (community.hashi_vault)
patch_notify_webhook_url: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                               engine_mount_point=vault_kv_mount, url=vault_addr).secret.url }}"

# AWS Secrets Manager (amazon.aws)
patch_notify_webhook_url: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- Always place this role **last** in a patch wrapper playbook, after the
  actual patch role(s), so `hostvars` contains the completed per-host
  results to aggregate.
- Every task that resolves or uses the webhook URL/payload is marked
  `no_log: true` to prevent a bearer-token-bearing URL from leaking into
  Ansible's stdout/log output.
- The debug summary at the end always prints locally, independent of
  webhook delivery, so operators still see a run summary even if
  `patch_notify_webhook_enabled: false` or the secrets lookup fails.

## References

- [ansible.builtin.uri module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
- [amazon.aws.aws_secret lookup docs](https://docs.ansible.com/ansible/7/collections/amazon/aws/aws_secret_lookup.html)
- [Slack incoming webhooks documentation](https://api.slack.com/messaging/webhooks)
- [Microsoft Teams connector card reference](https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/connectors-using)
