# certificate_management

**Category:** Security & Compliance
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Deploys and rotates TLS certificates and private keys pulled from an external secrets
manager — HashiCorp Vault (PKI secrets engine or KV) or AWS Secrets Manager, selectable per
certificate via `secret_backend` — installs them to configurable paths with correct
ownership/permissions, backs up prior material before rotation, reloads dependent services
(nginx/httpd/apache2/haproxy) via handler notify, and monitors expiry with a
`community.crypto.x509_certificate_info` check that warns if `not_after` falls within
`cert_expiry_warning_days`.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `certificate_deployments` | `[]` | List of certs to manage: `name, secret_backend, secret_path, cert_dest, key_dest, chain_dest, owner, group, key_mode, cert_mode, notify_service` |
| `cert_expiry_warning_days` | `30` | Days before expiry to raise a warning |
| `cert_backup_before_rotation` | `true` | Back up existing cert/key before overwriting |
| `cert_backup_dir` | `/var/backups/tls_certs` | Backup destination directory |
| `vault_addr`, `vault_kv_mount` | see defaults | HashiCorp Vault lookup coordinates |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |

## Example play

```yaml
- hosts: web_tier
  become: true
  roles:
    - role: certificate_management
      vars:
        certificate_deployments:
          - name: www-example-com
            secret_backend: vault
            secret_path: "datacenter/pki/certs/www.example.com"
            cert_dest: /etc/pki/tls/certs/www.example.com.crt
            key_dest: /etc/pki/tls/private/www.example.com.key
            chain_dest: /etc/pki/tls/certs/www.example.com-chain.crt
            owner: root
            group: nginx
            notify_service: nginx
```

## Secrets manager pattern

```yaml
# HashiCorp Vault (community.hashi_vault) — secret expected to expose
# .secret.certificate / .secret.private_key / .secret.chain fields
cert_bundle: "{{ lookup('community.hashi_vault.vault_kv2_get', cert_item.secret_path,
                 engine_mount_point=vault_kv_mount, url=vault_addr).secret }}"

# AWS Secrets Manager (amazon.aws) — secret expected to be a JSON string with
# the same certificate/private_key/chain fields
cert_bundle: "{{ lookup('amazon.aws.aws_secret', cert_item.secret_path, region=aws_region) | from_json }}"
```

## Notes

- Private key material is retrieved and written with `no_log: true` on every task that
  touches it, and the transient fact is cleared at the end of each certificate's deploy loop.
- Add a matching named handler in `handlers/main.yml` for any service beyond
  nginx/httpd/apache2/haproxy that needs a reload on certificate rotation.

## References

- [community.crypto.x509_certificate_info module docs](https://docs.ansible.com/ansible/latest/collections/community/crypto/x509_certificate_info_module.html)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
- [amazon.aws.aws_secret lookup docs](https://docs.ansible.com/ansible/7/collections/amazon/aws/aws_secret_lookup.html)
