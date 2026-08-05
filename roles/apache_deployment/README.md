# apache_deployment

**Category:** Application Deployment
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Installs Apache (`httpd` on RedHat family, `apache2` on Debian family) and manages a
data-driven set of virtual hosts from the `apache_vhosts` list variable. RedHat serves
vhost files directly through `conf.d/*.conf` includes (no enable step needed), while
Debian uses the classic `sites-available`/`sites-enabled` layout: modules are enabled with
`community.general.apache2_module` and sites are enabled/disabled with an `a2ensite`/
`a2dissite` command wrapper (there is no dedicated Ansible module for site enablement).
Each vhost may serve static content or reverse-proxy to a backend (`ProxyPass`), and can
optionally terminate TLS using certificate/key paths produced by the `certificate_management`
role. Every config-changing task notifies a handler that reloads the service.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `apache_server_admin` | `webmaster@example.com` | `ServerAdmin` contact address |
| `apache_server_tokens` | `"Prod"` | Minimizes information disclosed in the `Server` header |
| `apache_debian_modules_enabled` | `[rewrite, ssl, headers, proxy, proxy_http]` | Modules enabled via `apache2_module` on Debian family |
| `apache_vhosts` | `[]` | List of site definitions (name, server_name, listen_port, document_root/proxy_pass, tls_enabled, tls_cert_path, tls_key_path, enabled) |
| `apache_certificate_management_output_dir` | `/etc/pki/tls/apache` | Default cert/key directory produced by the `certificate_management` role |
| `apache_tls_protocols` / `apache_tls_ciphers` | modern baseline | TLS protocol/cipher restriction for vhosts with `tls_enabled: true` |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates (basic-auth/API-key use cases) |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |

## Example play

```yaml
- hosts: web_fleet
  become: true
  roles:
    - role: apache_deployment
      vars:
        apache_vhosts:
          - name: legacy-app.example.com
            server_name: legacy-app.example.com
            listen_port: 80
            document_root: /var/www/legacy-app
          - name: api.example.com
            server_name: api.example.com
            tls_enabled: true
            tls_cert_path: /etc/pki/tls/apache/api.example.com.crt
            tls_key_path: /etc/pki/tls/apache/api.example.com.key
            proxy_pass: "http://127.0.0.1:8080/"
```

## Secrets manager pattern (optional, for vhosts needing basic-auth)

```yaml
# HashiCorp Vault (community.hashi_vault)
apache_basic_auth_password: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                                  engine_mount_point=vault_kv_mount, url=vault_addr).secret.password }}"

# AWS Secrets Manager (amazon.aws)
apache_basic_auth_password: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- On Debian family, the distro default site is disabled with `a2dissite` only when
  `apache_vhosts` is non-empty, so a bare install still serves its default page.
- SELinux's `httpd_can_network_connect` boolean is enabled automatically on RedHat hosts
  only when at least one vhost defines `proxy_pass`.

## References

- [Apache httpd documentation](https://httpd.apache.org/docs/2.4/)
- [community.general.apache2_module module docs](https://docs.ansible.com/ansible/latest/collections/community/general/apache2_module_module.html)
- [ansible.posix.firewalld module docs](https://docs.ansible.com/ansible/latest/collections/ansible/posix/firewalld_module.html)
