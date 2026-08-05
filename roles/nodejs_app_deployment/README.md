# nodejs_app_deployment

**Category:** Application Deployment
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Installs Node.js (NodeSource repository package by default, or per-user `nvm` when
`nodejs_install_method: nvm`), deploys an application release into a timestamped
`releases/<id>` directory (from a `git` checkout or a tarball artifact), runs
`npm ci --production` via `community.general.npm`, symlinks `current -> releases/<id>`,
templates and manages a systemd unit, and performs a post-deploy HTTP health check.
The matching wrapper playbook applies `serial: "{{ rolling_batch_size }}"` so the fleet
rolls out gradually rather than restarting every host's app process simultaneously.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `nodejs_major_version` | `"20"` | NodeSource major version line (or nvm target version) |
| `nodejs_install_method` | `"nodesource"` | `"nodesource"` (system package) or `"nvm"` (per-user) |
| `app_name` | `"sample-node-app"` | Application/service name |
| `app_service_user` / `app_service_group` | `nodeapp` | Dedicated, no-login service account |
| `app_source_type` | `"git"` | `"git"` or `"tarball"` artifact source |
| `app_git_repo` / `app_git_version` | `""` / `"main"` | Git checkout coordinates |
| `app_artifact_url` / `app_artifact_checksum` | `""` | Tarball artifact coordinates |
| `app_exec_start` | `node .../current/server.js` | systemd `ExecStart` command |
| `app_port` | `3000` | Port the app listens on |
| `app_env_vars` | `{NODE_ENV, PORT}` | Environment variables injected into the systemd unit |
| `rolling_batch_size` | `"30%"` | Default `serial` batch size for the wrapper playbook |
| `app_health_check_url` | `http://127.0.0.1:3000/health` | Post-deploy HTTP health-check endpoint |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates for app runtime secrets |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |

## Example play

```yaml
- hosts: node_app_fleet
  become: true
  serial: "30%"
  roles:
    - role: nodejs_app_deployment
      vars:
        app_name: checkout-api
        app_git_repo: "https://github.com/example-org/checkout-api.git"
        app_git_version: "v2.4.1"
        app_port: 3001
        app_env_vars:
          NODE_ENV: production
          PORT: 3001
          DATABASE_PASSWORD: "{{ lookup('community.hashi_vault.vault_kv2_get',
                                   'datacenter/apps/checkout-api/' + inventory_hostname,
                                   engine_mount_point='kv', url=vault_addr).secret.db_password }}"
```

## Secrets manager pattern (runtime environment secrets)

```yaml
# HashiCorp Vault (community.hashi_vault)
DATABASE_PASSWORD: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                         engine_mount_point=vault_kv_mount, url=vault_addr).secret.db_password }}"

# AWS Secrets Manager (amazon.aws)
DATABASE_PASSWORD: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- Each deploy creates a new timestamped release directory; pair this role with the
  `artifact_registry_deployment` role's retention pattern if you need automatic pruning
  of old releases on this role's release path.
- The health check retries against `app_health_check_url` before the play is considered
  successful for that batch, preventing a bad rolling deploy from silently reaching 100%.

## References

- [community.general.npm module docs](https://docs.ansible.com/ansible/latest/collections/community/general/npm_module.html)
- [NodeSource distributions](https://github.com/nodesource/distributions)
- [ansible.builtin.uri module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html)
- [ansible.builtin.systemd_service module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/systemd_service_module.html)
