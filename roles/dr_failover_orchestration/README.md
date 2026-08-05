# dr_failover_orchestration

**Category:** Disaster Recovery
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Orchestrates failover of production traffic to the `dr_site_group`. The role verifies DR site
health checks pass, promotes a standby database, repoints the DNS/load-balancer configuration
(HAProxy template update + reload, plus an optional DNS API record update), and sends a
completion notification. This is a genuinely destructive DR action, so it is gated behind both
an explicit `dr_failover_confirm: true` variable and an interactive `ansible.builtin.pause`
confirmation before anything is changed.

**Important — two ways to use this role, depending on `dr_promotion_method`:**

- `dr_promotion_method: replica_command` — a single lightweight command promotes the standby.
  Safe to invoke this role directly, all-in-one, via its `tasks/main.yml` (see example play below).
- `dr_promotion_method: restore` — promotion is a full `database_backup_restore` run, which MUST
  execute directly against the DR database host rather than merely `delegate_to` from within this
  role (a role cannot re-target a different host than the play that includes it). Use
  `playbooks/disaster_recovery/42_dr_failover_orchestration.yml` instead, which runs a 3-play
  sequence: `pre_promote.yml` (checks/health-check) -> `database_backup_restore` on the DR DB host
  -> `post_promote.yml` (LB/DNS/notify). Do not call `tasks/main.yml` directly for this mode — it
  will fail its own assertion and tell you to use the wrapper playbook.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `dr_site_group` | `dr_site` | Inventory group representing the DR-site hosts |
| `dr_failover_confirm` | `false` | Must be explicitly `true` to allow the play to proceed past the guard |
| `dr_health_check_urls` | see defaults | Endpoints checked for DR-site readiness before failover |
| `dr_promote_database` | `true` | Whether to run the database promotion step |
| `dr_promotion_method` | `replica_command` | `replica_command` or `restore` (delegates to `database_backup_restore`) |
| `dr_replica_promote_command` | see defaults | Command run on the standby to promote it to primary |
| `dr_lb_config_template` / `dr_lb_config_dest` | see defaults | HAProxy config template and destination path |
| `dr_dns_repoint_enabled` | `true` | Whether to call the DNS provider API to repoint the record |
| `dr_dns_record_name` / `dr_dns_target_ip` / `dr_dns_ttl_seconds` | see defaults | DNS record repoint parameters |
| `dr_credential_source` | `vault` | `vault` or `aws_secrets_manager` — resolves the DNS API token |
| `dr_notify_webhook_url` | `""` | Optional webhook notified on failover completion |

## Example play

```yaml
- hosts: dr_site
  become: true
  serial: 1
  roles:
    - role: dr_failover_orchestration
      vars:
        dr_failover_confirm: true
        dr_site_group: dr_site
        dr_promotion_method: replica_command
```

## Secrets manager pattern

```yaml
# HashiCorp Vault (community.hashi_vault)
dns_api_token: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                    engine_mount_point=vault_kv_mount, url=vault_addr).secret.dns_api_token }}"

# AWS Secrets Manager (amazon.aws)
dns_api_token: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- Never run this role without `dr_failover_confirm: true` and a real incident ticket — the
  assertion and pause gate exist specifically to prevent accidental production traffic cutover.
- Run with `serial: 1` and validate health checks between batches when the DR site has multiple
  hosts fronting the load balancer.
- Follow up with `backup_restore_verification` and `replication_health_check` immediately after
  a failover to confirm the new primary is healthy.

## References

- [ansible.builtin.pause module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/pause_module.html)
- [ansible.builtin.uri module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
- [HAProxy configuration manual](https://www.haproxy.org/download/2.8/doc/configuration.txt)
