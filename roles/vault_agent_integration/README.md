# vault_agent_integration

**Category:** Security & Compliance
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Installs and configures the HashiCorp Vault Agent as a systemd service on each managed
node, authenticating via AppRole with an auto-renewing token and rendering one or more
secrets to local files (via Consul Template syntax) for consumption by other roles/
applications. For shops standardized on AWS Secrets Manager instead of Vault, an alternate
lightweight mode (`vault_agent_integration_mode: aws_lightweight`) skips the Vault Agent
daemon entirely and instead installs a systemd timer that periodically refreshes a secret
using the `amazon.aws.aws_secret` lookup pattern.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `vault_agent_integration_mode` | `vault_agent` | `vault_agent` or `aws_lightweight` |
| `vault_addr` | see defaults | Vault server address |
| `vault_agent_version` | `1.16.3` | Vault Agent binary release version |
| `vault_approle_role_id` | `""` | AppRole role_id (not secret; may live in group_vars) |
| `vault_approle_secret_id_path` | see defaults | Vault KV path where the AppRole secret_id is stored |
| `vault_agent_templates` | `[]` | List of `{secret_path, dest, mode, owner, group}` secret-to-file renders |
| `vault_agent_token_renew_enabled` | `true` | Enable Vault Agent's token auto-renew cache |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |
| `aws_lightweight_secret_path` | see defaults | Secret path/name refreshed in lightweight mode |
| `aws_lightweight_dest_file` | `/etc/vault-agent/aws_secret_lightweight.env` | Destination file for the refreshed secret |
| `aws_lightweight_refresh_interval` | `15min` | systemd timer `OnUnitActiveSec=` refresh interval |

## Example play

```yaml
- hosts: app_tier
  become: true
  roles:
    - role: vault_agent_integration
      vars:
        vault_agent_integration_mode: vault_agent
        vault_approle_role_id: "db02de05-fa39-4855-059b-67221c5c2f63"
        vault_agent_templates:
          - secret_path: "kv/data/datacenter/myapp/db_creds"
            dest: /etc/myapp/db_creds.env
            mode: "0600"
```

## Secrets manager pattern

```yaml
# HashiCorp Vault (community.hashi_vault) — used to bootstrap the AppRole secret_id itself
vault_approle_secret_id: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_approle_secret_id_path,
                            engine_mount_point=vault_kv_mount, url=vault_addr).secret.secret_id }}"

# AWS Secrets Manager (amazon.aws) — lightweight mode pre-seed and periodic refresh
aws_secret_value: "{{ lookup('amazon.aws.aws_secret', aws_lightweight_secret_path, region=aws_region) }}"
```

## Notes

- The AppRole `secret_id` and rendered Vault Agent token are always written with
  `no_log: true` and restrictive file modes (`0600`), owned by the dedicated `vault-agent`
  system user.
- In `aws_lightweight` mode, the AWS CLI itself must already be authenticated on the host
  (instance profile, IRSA, or configured credentials) — Ansible only manages the refresh
  script, systemd units, and the initial pre-seed via the `amazon.aws.aws_secret` lookup.

## References

- [Vault Agent documentation](https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent)
- [Vault AppRole auth method docs](https://developer.hashicorp.com/vault/docs/auth/approle)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
- [amazon.aws.aws_secret lookup docs](https://docs.ansible.com/ansible/7/collections/amazon/aws/aws_secret_lookup.html)
