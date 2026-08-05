# ssh_hardening

**Category:** Security & Compliance
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Hardens the OpenSSH server on every managed host: enforces key-only authentication,
disables root login, restricts ciphers/MACs/KEX to a modern baseline, applies
access-control lists (`AllowGroups`/`AllowUsers`), deploys a legal warning banner,
and validates the new `sshd_config` with `sshd -t` before it replaces the live file
(with an automatic timestamped backup).

## Key variables

| Variable | Default | Description |
|---|---|---|
| `ssh_permit_root_login` | `"no"` | Disable direct root SSH login |
| `ssh_password_authentication` | `"no"` | Disable password auth (keys only) |
| `ssh_port` | `22` | SSH listener port; role opens the matching firewalld/ufw exception automatically if changed |
| `ssh_max_auth_tries` | `3` | Max auth attempts before disconnect |
| `ssh_ciphers` / `ssh_macs` / `ssh_kex_algorithms` | modern list | Restricts negotiated crypto to strong, current algorithms |
| `ssh_allow_groups` / `ssh_allow_users` | `[]` | Optional allow-list restriction |
| `ssh_banner_enabled` | `true` | Deploys `/etc/issue.net` legal banner |
| `ssh_validate_config_before_reload` | `true` | Runs `sshd -t` against the rendered file before activating it |
| `ssh_hostkey_from_vault_enabled` | `false` | Opt-in: pull a centrally-issued host key from HashiCorp Vault / AWS Secrets Manager instead of the locally generated one |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: ssh_hardening
      vars:
        ssh_port: 2222
        ssh_allow_groups: ["sysadmins", "sre"]
```

## Secrets manager pattern (if `ssh_hostkey_from_vault_enabled: true`)

```yaml
# HashiCorp Vault (community.hashi_vault)
ssh_host_key_private: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                           engine_mount_point=vault_kv_mount, url=vault_addr).secret.private_key }}"

# AWS Secrets Manager (amazon.aws)
ssh_host_key_private: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- Always keep an out-of-band console/IPMI/iLO access path before rolling this out fleet-wide —
  a misapplied `AllowGroups`/`AllowUsers` filter can lock out SSH access.
- Roll out with `serial: "10%"` on the first batch in production estates.

## References

- [OpenSSH sshd_config manual](https://man.openbsd.org/sshd_config)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
- [amazon.aws.aws_secret lookup docs](https://docs.ansible.com/ansible/7/collections/amazon/aws/aws_secret_lookup.html)
- [ansible.posix.firewalld module docs](https://docs.ansible.com/ansible/latest/collections/ansible/posix/firewalld_module.html)
