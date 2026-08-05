# docker_engine_setup

**Category:** Application Deployment
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Installs Docker Engine from Docker's official upstream repository (GPG-key import +
repo-add, RedHat via `ansible.builtin.yum_repository`, Debian via
`ansible.builtin.apt_repository` with a keyring-based `signed-by` pin), templates
`/etc/docker/daemon.json` (log driver/size limits, storage driver, registry mirrors,
insecure registries, address pools), adds specified users to the `docker` group for
passwordless CLI access, and enables the service. `daemon.json` changes notify a handler
that restarts docker (never an unconditional restart on every run).

## Key variables

| Variable | Default | Description |
|---|---|---|
| `docker_log_driver` | `"json-file"` | Container log driver |
| `docker_log_max_size` / `docker_log_max_file` | `"10m"` / `"3"` | Log rotation limits |
| `docker_storage_driver` | `"overlay2"` | Storage driver |
| `docker_registry_mirrors` | `[]` | List of registry mirror URLs |
| `docker_insecure_registries` | `[]` | List of insecure (HTTP or self-signed) registries |
| `docker_default_address_pools` | `[]` | Custom bridge network address pools |
| `docker_live_restore` | `true` | Keep containers running across dockerd restarts |
| `docker_group_members` | `[]` | Users added to the `docker` group |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates (private registry auth) |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |

## Example play

```yaml
- hosts: container_hosts
  become: true
  roles:
    - role: docker_engine_setup
      vars:
        docker_group_members: ["deploy", "ansible_svc"]
        docker_registry_mirrors: ["https://mirror.gcr.io"]
        docker_insecure_registries: ["registry.internal.example.com:5000"]
```

## Secrets manager pattern (private registry authentication)

```yaml
# HashiCorp Vault (community.hashi_vault)
docker_registry_password: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                                 engine_mount_point=vault_kv_mount, url=vault_addr).secret.password }}"

# AWS Secrets Manager (amazon.aws)
docker_registry_password: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- Users added to the `docker` group receive root-equivalent access on the host via the
  Docker socket — restrict `docker_group_members` to trusted service/deploy accounts only.
- On RedHat family hosts with firewalld active, the `docker0` bridge interface is placed
  in the `trusted` zone so Docker's own iptables chains are not double-filtered.

## References

- [Docker Engine install docs](https://docs.docker.com/engine/install/)
- [Docker daemon.json reference](https://docs.docker.com/reference/cli/dockerd/#daemon-configuration-file)
- [ansible.builtin.apt_repository module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_repository_module.html)
- [ansible.builtin.yum_repository module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_repository_module.html)
