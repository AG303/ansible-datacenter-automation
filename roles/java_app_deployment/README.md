# java_app_deployment

**Category:** Application Deployment
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Installs a pinned OpenJDK version and deploys a Java application in one of three modes
selected by `app_type`: a standalone `jar` or `war` run under a custom systemd unit
(the default paths), or a full Apache Tomcat installation (`app_type: tomcat`) with the
WAR artifact dropped into `webapps/`. Both paths follow a drain-then-restart rolling
pattern — an `ansible.builtin.pause` grace period lets in-flight requests complete before
the service is restarted — and both finish with an HTTP health check against
`app_health_check_url`. The matching wrapper playbook applies
`serial: "{{ rolling_batch_size }}"` for gradual rollout across the fleet.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `java_version` | `"17"` | OpenJDK major version installed |
| `app_type` | `"jar"` | `"jar"`, `"war"`, or `"tomcat"` |
| `app_name` | `"sample-java-app"` | Application/service name |
| `app_artifact_url` / `app_artifact_checksum` | `""` | Artifact download coordinates (jar/war/Tomcat WAR) |
| `java_heap_min` / `java_heap_max` | `512m` / `1024m` | JVM heap sizing |
| `app_port` | `8080` | Port the jar/war app listens on |
| `tomcat_version` | `"10.1.24"` | Apache Tomcat version installed when `app_type: tomcat` |
| `tomcat_http_port` | `8080` | Tomcat HTTP connector port |
| `rolling_batch_size` | `"30%"` | Default `serial` batch size for the wrapper playbook |
| `app_drain_wait_seconds` | `15` | Grace period before restarting the service |
| `app_health_check_url` | `http://127.0.0.1:8080/actuator/health` | Post-deploy HTTP health-check endpoint |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates for app runtime secrets |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |

## Example play

```yaml
- hosts: java_app_fleet
  become: true
  serial: "30%"
  roles:
    - role: java_app_deployment
      vars:
        app_name: orders-service
        app_type: jar
        app_artifact_url: "https://artifacts.example.com/orders-service/3.1.0.jar"
        app_artifact_checksum: "sha256:7f2c...ab90"
        app_port: 8081
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

- When `app_type: tomcat`, the standalone jar/war systemd tasks are skipped entirely and
  `tasks/tomcat.yml` handles installation, WAR deployment, and its own systemd unit.
- Always verify `app_artifact_checksum` in production — it is optional only to ease local
  testing, and the role emits a warning-free but less-safe path when it is left empty.

## References

- [Apache Tomcat documentation](https://tomcat.apache.org/tomcat-10.1-doc/index.html)
- [ansible.builtin.get_url module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html)
- [ansible.builtin.systemd_service module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/systemd_service_module.html)
