# systemd_service_deployment

**Category:** Application Deployment
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Generic, reusable role for deploying any custom systemd-managed process. Given
`service_name`, `exec_start`, `working_directory`, `service_user`, and an `env_vars` dict,
it creates the service account and working directory (optional, guarded), templates a
systemd unit file, reloads the systemd daemon, and enables/starts the service. An optional
HTTP health-check hook (`service_health_check_enabled`) waits for the port to accept
connections and then polls the health endpoint before the task run is considered complete —
useful when this role is included by other roles (such as `blue_green_deployment`) or driven
by a wrapper playbook with `serial: "{{ rolling_batch_size }}"` for a gradual rollout. Unlike
the app-specific deployment roles in this collection, this role makes no assumptions about
what the service actually runs.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `service_name` | `"sample-custom-service"` | systemd unit / service name |
| `exec_start` | `/usr/local/bin/sample-custom-service` | `ExecStart` command |
| `working_directory` | `/opt/<service_name>` | `WorkingDirectory` |
| `service_user` / `service_group` | `svc-<service_name>` | Dedicated, no-login service account |
| `env_vars` | `{}` | Environment variables injected into the unit |
| `service_restart_policy` | `"on-failure"` | systemd `Restart=` policy |
| `service_manage_user` / `service_manage_directory` | `true` | Whether this role creates the account/directory, or expects them to pre-exist |
| `rolling_batch_size` | `"30%"` | Default `serial` batch size for the wrapper playbook |
| `service_health_check_enabled` | `false` | Opt-in HTTP health-check hook |
| `service_health_check_url` | `""` | Endpoint polled after (re)start when the hook is enabled |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates for `env_vars` secrets |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |

## Example play

```yaml
- hosts: metrics_exporters
  become: true
  serial: "30%"
  roles:
    - role: systemd_service_deployment
      vars:
        service_name: node-exporter
        exec_start: "/usr/local/bin/node_exporter --web.listen-address=:9100"
        working_directory: /opt/node-exporter
        service_user: node-exporter
        service_group: node-exporter
        service_health_check_enabled: true
        service_health_check_url: "http://127.0.0.1:9100/metrics"
        env_vars:
          LOG_LEVEL: info
```

## Secrets manager pattern (env_vars containing credentials)

```yaml
# HashiCorp Vault (community.hashi_vault)
env_vars:
  API_KEY: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                  engine_mount_point=vault_kv_mount, url=vault_addr).secret.api_key }}"

# AWS Secrets Manager (amazon.aws)
env_vars:
  API_KEY: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- Because this role targets systemd directly rather than a distro-specific package
  manager, it needs no `tasks/redhat.yml` / `tasks/debian.yml` split — its behavior is
  identical across every supported OS family.
- Other roles in this collection (e.g. `blue_green_deployment`) can include this role via
  `app_deploy_role: systemd_service_deployment` to drive a generic service through the same
  orchestration pattern used for the language-specific app roles.

## References

- [systemd.service manual](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)
- [ansible.builtin.systemd_service module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/systemd_service_module.html)
- [ansible.builtin.uri module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html)
