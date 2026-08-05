# haproxy_load_balancer

**Category:** Application Deployment
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Installs HAProxy and templates `haproxy.cfg` from a `haproxy_backends` list variable —
each entry defines a frontend bind address and a backend whose server list is generated
automatically from an Ansible inventory group (`backend_group`), so adding/removing a host
from that group changes the load-balanced pool on the next run. The stats socket and an
authenticated stats dashboard are enabled by default. Every rendered config is validated
with `haproxy -c -f` against a staged file before it replaces the live config, and a
timestamped backup of the previous config is kept. Config changes notify a handler that
reloads (never restarts) haproxy.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `haproxy_maxconn` | `20000` | Global max connections |
| `haproxy_mode` | `"http"` | Default proxy mode (`http` or `tcp`) |
| `haproxy_backends` | `[]` | List of `{name, frontend_bind, backend_port, backend_group, balance_algorithm, health_check_path}` |
| `haproxy_stats_enabled` | `true` | Enables the stats socket + dashboard |
| `haproxy_stats_bind_address` / `haproxy_stats_port` | `127.0.0.1:8404` | Stats dashboard bind coordinates |
| `haproxy_stats_user` | `"admin"` | Stats dashboard basic-auth username |
| `haproxy_validate_config_before_reload` | `true` | Runs `haproxy -c -f` against the staged file before activating it |
| `haproxy_backup_existing_config` | `true` | Keeps a timestamped backup of the previous config |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates for the stats password |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |

## Example play

```yaml
- hosts: load_balancers
  become: true
  roles:
    - role: haproxy_load_balancer
      vars:
        haproxy_backends:
          - name: web_backend
            frontend_bind: "*:80"
            backend_port: 8080
            backend_group: web_fleet
            balance_algorithm: roundrobin
            health_check_path: /health
```

## Secrets manager pattern (stats dashboard password)

```yaml
# HashiCorp Vault (community.hashi_vault)
haproxy_stats_password: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                              engine_mount_point=vault_kv_mount, url=vault_addr).secret.stats_password }}"

# AWS Secrets Manager (amazon.aws)
haproxy_stats_password: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- `backend_group` must be a valid Ansible inventory group; each member's
  `ansible_default_ipv4.address` fact is used as the backend server address, so `gather_facts`
  must run against those hosts (via `hostvars`) before this role templates the config.
- Always keep the timestamped `haproxy.cfg.bak-*` backups if you need to manually roll back
  a bad backend definition outside of Ansible.

## References

- [HAProxy configuration manual](https://docs.haproxy.org/2.8/configuration.html)
- [ansible.builtin.template module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html)
- [ansible.posix.firewalld module docs](https://docs.ansible.com/ansible/latest/collections/ansible/posix/firewalld_module.html)
