# nginx_deployment

**Category:** Application Deployment
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Installs nginx and manages it as a data-driven collection of virtual hosts. A single
`nginx_vhosts` list variable drives one templated server-block file per site: RedHat-family
hosts render straight into `conf.d/` (served automatically, no enable step), while
Debian-family hosts follow the classic `sites-available` -> `sites-enabled` symlink pattern,
with disabled vhosts having their symlink (Debian) or file (RedHat) removed. Each site may be
a static file root or a reverse proxy (`proxy_pass`), and may optionally terminate TLS using
certificate/key paths produced by the `certificate_management` role. All config changes notify
a handler that reloads nginx (never an unconditional restart), and the rendered `nginx.conf` is
validated with `nginx -t` before it is allowed to replace the live file.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `nginx_worker_processes` | `"auto"` | Worker process count |
| `nginx_worker_connections` | `1024` | Max connections per worker |
| `nginx_client_max_body_size` | `"10m"` | Max request body size |
| `nginx_gzip_enabled` | `true` | Enables gzip compression |
| `nginx_server_tokens_off` | `true` | Hides the nginx version in the `Server` response header |
| `nginx_vhosts` | `[]` | List of site definitions (name, server_name, listen_port, root/proxy_pass, tls_enabled, tls_cert_path, tls_key_path, extra_locations, enabled) |
| `nginx_certificate_management_output_dir` | `/etc/pki/nginx/certs` | Default cert/key directory produced by the `certificate_management` role |
| `nginx_tls_protocols` / `nginx_tls_ciphers` | modern baseline | TLS protocol/cipher restriction for vhosts with `tls_enabled: true` |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates (for basic-auth/API-key use cases) |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |

## Example play

```yaml
- hosts: web_fleet
  become: true
  roles:
    - role: nginx_deployment
      vars:
        nginx_vhosts:
          - name: app.example.com
            server_name: app.example.com
            listen_port: 80
            tls_enabled: true
            tls_cert_path: /etc/pki/nginx/certs/app.example.com.crt
            tls_key_path: /etc/pki/nginx/certs/app.example.com.key
            proxy_pass: "http://127.0.0.1:3000"
          - name: static-site.example.com
            server_name: static-site.example.com
            root: /var/www/static-site
```

## Secrets manager pattern (optional, for vhost extra_locations needing auth)

```yaml
# HashiCorp Vault (community.hashi_vault)
nginx_basic_auth_password: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                                 engine_mount_point=vault_kv_mount, url=vault_addr).secret.password }}"

# AWS Secrets Manager (amazon.aws)
nginx_basic_auth_password: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- TLS cert/key paths are expected to come from the `certificate_management` role's output
  directory — this role does not issue certificates itself.
- Removing the distro default vhost only happens when `nginx_vhosts` is non-empty, avoiding
  accidental loss of the default catch-all page on a bare install.

## References

- [nginx core module docs](https://nginx.org/en/docs/)
- [ansible.builtin.template module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html)
- [ansible.posix.firewalld module docs](https://docs.ansible.com/ansible/latest/collections/ansible/posix/firewalld_module.html)
- [community.general.ufw module docs](https://docs.ansible.com/ansible/latest/collections/community/general/ufw_module.html)
