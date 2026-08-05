# firewall_management

**Category:** Security & Compliance
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Provides a single, distro-agnostic interface for default-deny inbound firewall management:
`firewalld` on RHEL-family hosts and `ufw` on Debian-family hosts. All inbound access is
denied by default; explicit exceptions are declared through the `firewall_allowed_rules`
list variable so the allow-list is auditable in version control. Supports zone/interface
assignment on firewalld and default-policy configuration on ufw, and performs an idempotent
sync of the declared rules on every run.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `firewall_default_zone` | `public` | firewalld zone new rules are applied to |
| `firewall_default_incoming_policy` | `deny` | ufw default inbound policy |
| `firewall_default_outgoing_policy` | `allow` | ufw default outbound policy |
| `firewall_logging` | `"on"` | firewalld/ufw logging level |
| `firewall_allowed_rules` | `[]` | List of `{name, port, proto, service, source}` allow-list entries |
| `firewall_zone_interfaces` | `[]` | List of `{zone, interface}` firewalld zone assignments |
| `firewall_manage_package` | `true` | Whether this role installs the firewall package |
| `firewall_prune_unmanaged_rules` | `false` | Remove previously-applied rules no longer declared in `firewall_allowed_rules` |
| `firewall_source_cidrs_from_vault_enabled` | `false` | Opt-in: pull trusted CIDR list from HashiCorp Vault / AWS Secrets Manager |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: firewall_management
      vars:
        firewall_allowed_rules:
          - name: "SSH management access"
            port: 22
            proto: tcp
            source: "10.0.0.0/8"
          - name: "HTTPS"
            service: https
            port: 443
            proto: tcp
```

## Secrets manager pattern (if `firewall_source_cidrs_from_vault_enabled: true`)

```yaml
# HashiCorp Vault (community.hashi_vault)
firewall_trusted_cidrs: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                             engine_mount_point=vault_kv_mount, url=vault_addr).secret.cidrs }}"

# AWS Secrets Manager (amazon.aws)
firewall_trusted_cidrs: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- Verify out-of-band console access before applying a restrictive `firewall_allowed_rules`
  set fleet-wide; an incomplete allow-list can lock out management access.
- `firewall_prune_unmanaged_rules` is disabled by default to avoid accidentally removing
  rules created outside this role; enable only once the allow-list is the single source of truth.

## References

- [ansible.posix.firewalld module docs](https://docs.ansible.com/ansible/latest/collections/ansible/posix/firewalld_module.html)
- [community.general.ufw module docs](https://docs.ansible.com/ansible/latest/collections/community/general/ufw_module.html)
- [community.general.ini_file module docs](https://docs.ansible.com/ansible/latest/collections/community/general/ini_file_module.html)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
- [amazon.aws.aws_secret lookup docs](https://docs.ansible.com/ansible/7/collections/amazon/aws/aws_secret_lookup.html)
