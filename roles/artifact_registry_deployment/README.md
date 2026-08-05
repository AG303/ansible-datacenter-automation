# artifact_registry_deployment

**Category:** Application Deployment
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Deploys a versioned application artifact (tarball/zip) sourced from a Nexus/Artifactory
repository over HTTPS (`ansible.builtin.get_url` with a bearer token) or from S3
(`amazon.aws.s3_object`), verifies its checksum, unpacks it into
`releases/<app_version>`, atomically re-points the `current` symlink at the new release,
prunes releases beyond `artifact_keep_releases` (kept for fast rollback — just re-point
`current` at an older `releases/<version>` directory), and restarts the target systemd
service via a notify handler. Pair this role with `systemd_service_deployment` (pointing
its `working_directory` at `{{ app_base_dir }}/current`) for a complete release pipeline.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `app_name` / `app_version` | `sample-artifact-app` / `1.0.0` | Application identity and the version being deployed this run |
| `app_base_dir` | `/opt/<app_name>` | Base directory holding `releases/` and the `current` symlink |
| `artifact_source_type` | `"nexus_artifactory"` | `"nexus_artifactory"` or `"s3"` |
| `artifact_url` | `""` | Full download URL (Nexus/Artifactory path) |
| `artifact_s3_bucket` / `artifact_s3_key` | `""` | S3 source coordinates |
| `artifact_checksum` | `""` | e.g. `"sha256:abcdef..."`, verified before/after download |
| `artifact_keep_releases` | `5` | Number of past releases retained on disk for rollback |
| `rolling_batch_size` | `"30%"` | Default `serial` batch size for the wrapper playbook |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates for the registry auth token |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern and S3 downloads |

## Example play

```yaml
- hosts: app_servers
  become: true
  serial: "30%"
  roles:
    - role: artifact_registry_deployment
      vars:
        app_name: checkout-service
        app_version: "2.4.1"
        artifact_source_type: nexus_artifactory
        artifact_url: "https://nexus.example.com/repository/releases/checkout-service/2.4.1/checkout-service-2.4.1.tar.gz"
        artifact_checksum: "sha256:3f9c1a...c8de"
        app_service_name: checkout-service
        artifact_keep_releases: 5
```

## Secrets manager pattern (registry auth token)

```yaml
# HashiCorp Vault (community.hashi_vault)
artifact_registry_auth_token: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                                    engine_mount_point=vault_kv_mount, url=vault_addr).secret.auth_token }}"

# AWS Secrets Manager (amazon.aws)
artifact_registry_auth_token: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- For S3 sources, prefer an IAM instance role over static credentials; the
  `artifact_registry_auth_token` secrets-manager pattern above still applies if a static
  access key/secret pair must be injected as environment variables for `amazon.aws.s3_object`.
- All tasks that pass `artifact_registry_auth_token` in a header or use it for S3 auth are
  marked `no_log: true` to keep the token out of play output and logs.
- Rollback is a two-step operation: re-point the `current` symlink at an older
  `releases/<version>` directory with `ansible.builtin.file` (`state: link`) and restart the
  service — no re-download needed as long as the old release directory has not been pruned.

## References

- [ansible.builtin.get_url module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html)
- [amazon.aws.s3_object module docs](https://docs.ansible.com/ansible/latest/collections/amazon/aws/s3_object_module.html)
- [ansible.builtin.unarchive module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html)
- [community.hashi_vault collection docs](https://docs.ansible.com/ansible/latest/collections/community/hashi_vault/index.html)
