# Ansible Datacenter Automation

A fully-coded collection of **50 production-quality Ansible roles and wrapper playbooks** for
managing the configuration of a large datacenter of Linux servers across the RHEL family
(RHEL/CentOS/Rocky/Alma) and the Debian family (Ubuntu/Debian). It covers four domains:
**Security & Compliance**, **Patch Management**, **Application Deployment**, and
**Disaster Recovery**.

- **Compliance posture:** general security best practices (hardening, least-privilege, audit
  logging). This collection is **not** mapped to CIS Benchmark or DISA STIG control IDs — treat it
  as a strong baseline, not a certified compliance artifact.
- **Secrets management:** every credential (database passwords, TLS keys, DNS API tokens, etc.)
  is resolved at runtime from **HashiCorp Vault** (`community.hashi_vault`) or **AWS Secrets
  Manager** (`amazon.aws`) — nothing is ever hardcoded in a playbook, role default, or committed
  inventory file.
- **Inventory:** static, standard Ansible inventory (YAML), grouped by OS family
  (`rhel_family` / `debian_family`) plus role-specific functional groups (e.g. `dr_site`,
  `web_blue`/`web_green`).

## Repository structure

```
ansible.cfg                     # project-local Ansible configuration
requirements.yml                # ansible-galaxy collection dependencies (install before use)
site.yml                        # master playbook — imports all 4 category site-*.yml files
inventory/
  hosts.example.yml              # example static inventory — copy and adapt for your fleet
  group_vars/
    all.yml                      # fleet-wide defaults (secrets backend selection, etc.)
    rhel_family.yml               # RHEL-family group vars
    debian_family.yml             # Debian-family group vars
  host_vars/                     # per-host variable overrides (empty placeholder)
roles/
  <50 role directories>/          # each with defaults/, vars/, tasks/, handlers/, templates/,
                                   # meta/, and its own README.md
playbooks/
  security_compliance/01-13_*.yml # one wrapper playbook per Security & Compliance role
  patch_management/14-25_*.yml    # one wrapper playbook per Patch Management role
  app_deployment/26-38_*.yml      # one wrapper playbook per Application Deployment role
  disaster_recovery/39-50_*.yml   # one wrapper playbook per Disaster Recovery role
  site-security-compliance.yml    # category-level orchestration playbook
  site-patch-management.yml       # category-level orchestration playbook
  site-app-deployment.yml         # category-level orchestration playbook
  site-disaster-recovery.yml      # category-level orchestration playbook
```

Every individual role directory contains its own `README.md` with a full variable reference,
an example play, the secrets-manager lookup pattern it uses (where applicable), and links to the
official module documentation it relies on.

## Quickstart

### 1. Install collection dependencies

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Set up your inventory

Copy `inventory/hosts.example.yml` to `inventory/hosts.yml` (or point `ansible.cfg` /
`-i` at your own inventory source) and fill in your real hosts under the `rhel_family` and
`debian_family` groups, plus any functional groups a given role expects (see that role's
README — e.g. `dr_site`, `web_blue`/`web_green`, `backend_group`).

### 3. Configure your secrets manager

Set `secrets_backend: vault` or `secrets_backend: aws_secrets_manager` in
`inventory/group_vars/all.yml`, then provide the corresponding connection details:

**HashiCorp Vault** (`community.hashi_vault`):
```yaml
vault_addr: "https://vault.internal.example.com:8200"
vault_kv_mount: "kv"
# Authenticate via VAULT_TOKEN env var, AppRole, or another community.hashi_vault auth method —
# never hardcode a Vault token in inventory or group_vars.
```

**AWS Secrets Manager** (`amazon.aws`):
```yaml
aws_region: "us-east-1"
# Authenticate via the standard AWS credential chain (env vars, instance profile, or
# ~/.aws/credentials on the control node) — never hardcode AWS keys in inventory or group_vars.
```

Each role's README documents the exact secret path variable(s) it expects (e.g.
`vault_secret_path`, `mysql_root_password_secret_path`).

### 4. Run playbooks

```bash
# Everything, category by category (recommended over running site.yml wholesale in production):
ansible-playbook -i inventory/hosts.yml playbooks/site-security-compliance.yml
ansible-playbook -i inventory/hosts.yml playbooks/site-patch-management.yml
ansible-playbook -i inventory/hosts.yml playbooks/site-app-deployment.yml
ansible-playbook -i inventory/hosts.yml playbooks/site-disaster-recovery.yml

# A single role's wrapper playbook:
ansible-playbook -i inventory/hosts.yml playbooks/security_compliance/01_ssh_hardening.yml

# Everything via the master entry point (mainly useful for CI syntax-checking; --tags/--limit
# recommended for any real production run):
ansible-playbook -i inventory/hosts.yml site.yml --check --diff
```

**Deliberately excluded from the category `site-*.yml` auto-chains** — these are standalone,
often destructive, explicit-invocation-only tools that should never run unattended as part of a
routine sweep: `19_patch_rollback_snapshot.yml`, `42_dr_failover_orchestration.yml`, and
`48_dr_runbook_execution.yml`. Run these individually, with the explicit confirmation variables
each one's README documents (e.g. `dr_failover_confirm=true`).

### 5. Always dry-run first

```bash
ansible-playbook -i inventory/hosts.yml <playbook> --check --diff --limit <a_test_group>
```

## Report/artifact output on Ansible Automation Platform 2.6+

AAP 2.6 runs playbooks on **ephemeral execution-node pods**. Anything written
to `localhost`/the control node (via `delegate_to: localhost`,
`connection: local`, or `hosts: localhost`) disappears the moment the pod is
recycled and is never reachable outside that single run — it is never a valid
destination for a report, snapshot, or any other artifact that needs to
persist.

Every role in this collection that generates a durable file
(`patch_inventory_facts`, `patch_compliance_report`, `patch_rollback_snapshot`)
delegates its final write to a real, persistent Linux host instead, resolved
from one collection-wide variable:

```yaml
# inventory/group_vars/all.yml
report_archive_host: "report-archive.dc1.example.com"
report_archive_base_dir: "/srv/ansible-reports"
```

Override `report_archive_host` / `report_archive_base_dir` once (fleet-wide,
in `inventory/group_vars/all.yml`) to repoint **every** report-writing role at
a different archive host without touching any role code. Each affected role
also exposes its own `<role>_report_host` / `<role>_archive_host` default
(defaulting to `report_archive_host`) if that role's output needs a different
home than the rest of the collection — see each role's README for the exact
variable names. The `report_archive` inventory group in
`inventory/hosts.example.yml`, paired with `inventory/group_vars/report_archive.yml`,
is where you define connection details (SSH user/key, jump host, etc.) for
whatever host(s) you point `report_archive_host` at.

For content that is fully rendered in memory (JSON/CSV built from `hostvars`
via Jinja), the conversion is simply pointing `ansible.builtin.copy`'s
`delegate_to` at the archive host instead of `localhost` — there is no
remote source file to fetch, so nothing else changes. For content that must
first be pulled off a managed host (e.g. `patch_rollback_snapshot`'s
pre-patch package-version fact file), the pattern is `ansible.builtin.fetch`
to a transient local staging path on the execution node, immediately followed
by `ansible.builtin.copy` (`delegate_to` the archive host) to push it off the
pod before it is destroyed, with the staging copy removed at the end of the
same play.

## The 50 roles

### Security & Compliance (13)

| # | Role | Purpose | Playbook | Role docs |
|---|---|---|---|---|
| 01 | `ssh_hardening` | Hardens the OpenSSH server on every managed host: enforces key-only authentication, disables root login, restricts ciphers/MACs/KEX to a modern baseline, applies access-control lists (`AllowGroups`/`AllowUsers`), deploys a legal warning banner, and validates the new `sshd_config` with `sshd -t` before it replaces the live file (with an automatic timestamped backup). | [`01_ssh_hardening.yml`](playbooks/security_compliance/01_ssh_hardening.yml) | [README](roles/ssh_hardening/README.md) |
| 02 | `firewall_management` | Provides a single, distro-agnostic interface for default-deny inbound firewall management: `firewalld` on RHEL-family hosts and `ufw` on Debian-family hosts. | [`02_firewall_management.yml`](playbooks/security_compliance/02_firewall_management.yml) | [README](roles/firewall_management/README.md) |
| 03 | `user_account_management` | Manages the complete local user/group lifecycle from a single `user_accounts` list variable: creates and updates accounts (`state: present`), locks accounts that should retain data but lose login access (`state: disabled`), and fully decommissions accounts with an optional home directory archive before removal (`state: absent`). | [`03_user_account_management.yml`](playbooks/security_compliance/03_user_account_management.yml) | [README](roles/user_account_management/README.md) |
| 04 | `auditd_configuration` | Installs and configures the Linux Audit Daemon (`auditd`) with a curated, general best-practice rule set covering identity/account database changes, privilege escalation, network configuration changes, scheduled task (cron) modification, sudoers changes, and key log file watches (wtmp/btmp/lastlog/auth log). | [`04_auditd_configuration.yml`](playbooks/security_compliance/04_auditd_configuration.yml) | [README](roles/auditd_configuration/README.md) |
| 05 | `selinux_apparmor_management` | Provides a single interface for managing the mandatory access control (MAC) subsystem across both OS families. | [`05_selinux_apparmor_management.yml`](playbooks/security_compliance/05_selinux_apparmor_management.yml) | [README](roles/selinux_apparmor_management/README.md) |
| 06 | `fail2ban_deployment` | Installs and configures `fail2ban` to automatically block brute-force login attempts. | [`06_fail2ban_deployment.yml`](playbooks/security_compliance/06_fail2ban_deployment.yml) | [README](roles/fail2ban_deployment/README.md) |
| 07 | `password_policy_enforcement` | Enforces password complexity via PAM `pwquality` (minimum length, character-class requirements, max repeated characters, difference-from-previous requirement), account aging via `login.defs` (`PASS_MAX_DAYS`, `PASS_MIN_DAYS`, `PASS_WARN_AGE`), password history reuse prevention via `pam_pwhistory`, and account lockout after repeated failed logins via `pam_faillock` (preferred) or `pam_tally2` (fallback for older Debian releases). | [`07_password_policy_enforcement.yml`](playbooks/security_compliance/07_password_policy_enforcement.yml) | [README](roles/password_policy_enforcement/README.md) |
| 08 | `ntp_time_sync` | Installs and configures `chronyd` as the default NTP time-synchronization provider across the fleet, sets the system timezone, and verifies sync status as a health-check task after deployment. | [`08_ntp_time_sync.yml`](playbooks/security_compliance/08_ntp_time_sync.yml) | [README](roles/ntp_time_sync/README.md) |
| 09 | `baseline_os_hardening` | Applies general security best-practice OS hardening that is not mapped to any named compliance framework (no CIS/STIG control IDs anywhere in this role): blacklists unused filesystem kernel modules (cramfs, freevxfs, jffs2, hfs/hfsplus, squashfs, udf — `usb-storage` is opt-in only), applies sysctl network/kernel hardening (reverse-path filtering, disabling source routing/ICMP redirects, ASLR, dmesg/kptr restriction, ptrace scope), disables IP forwarding unless the host is explicitly marked as a router, restricts core dump generation, removes insecure legacy services (telnet, rsh, tftp, ypserv), and hardens `/tmp` mount options when `/tmp` is already a separate filesystem. | [`09_baseline_os_hardening.yml`](playbooks/security_compliance/09_baseline_os_hardening.yml) | [README](roles/baseline_os_hardening/README.md) |
| 10 | `certificate_management` | Deploys and rotates TLS certificates and private keys pulled from an external secrets manager — HashiCorp Vault (PKI secrets engine or KV) or AWS Secrets Manager, selectable per certificate via `secret_backend` — installs them to configurable paths with correct ownership/permissions, backs up prior material before rotation, reloads dependent services (nginx/httpd/apache2/haproxy) via handler notify, and monitors expiry with a `community.crypto.x509_certificate_info` check that warns if `not_after` falls within `cert_expiry_warning_days`. | [`10_certificate_management.yml`](playbooks/security_compliance/10_certificate_management.yml) | [README](roles/certificate_management/README.md) |
| 11 | `vault_agent_integration` | Installs and configures the HashiCorp Vault Agent as a systemd service on each managed node, authenticating via AppRole with an auto-renewing token and rendering one or more secrets to local files (via Consul Template syntax) for consumption by other roles/ applications. | [`11_vault_agent_integration.yml`](playbooks/security_compliance/11_vault_agent_integration.yml) | [README](roles/vault_agent_integration/README.md) |
| 12 | `compliance_scan_reporting` | Installs and runs `lynis` (default, cross-distro) or `openscap-scanner` (RedHat-family, best-effort on Debian) as a general system-hygiene audit tool, captures the raw scan report and log locally under `compliance_scan_report_dir`, parses a simple pass/warn/fail-style summary from the lynis log, and pushes that summary to a centralized webhook or S3 bucket for fleet-wide tracking. | [`12_compliance_scan_reporting.yml`](playbooks/security_compliance/12_compliance_scan_reporting.yml) | [README](roles/compliance_scan_reporting/README.md) |
| 13 | `sudo_privilege_management` | Generates `/etc/sudoers.d/*` drop-in files from a single `sudo_rules` list variable, so privileged command grants are version-controlled and auditable. | [`13_sudo_privilege_management.yml`](playbooks/security_compliance/13_sudo_privilege_management.yml) | [README](roles/sudo_privilege_management/README.md) |

### Patch Management (12)

| # | Role | Purpose | Playbook | Role docs |
|---|---|---|---|---|
| 14 | `patch_inventory_facts` | Read-only fleet-wide reporting role. | [`14_patch_inventory_facts.yml`](playbooks/patch_management/14_patch_inventory_facts.yml) | [README](roles/patch_inventory_facts/README.md) |
| 15 | `os_patch_rolling_update` | Performs a full OS package upgrade fleet-wide (`dnf upgrade` on RedHat family, `apt full-upgrade` on Debian family) while protecting production availability: a pre-update health check confirms the host is healthy before touching packages, the upgrade runs, connectivity is re-established with `ansible.builtin.wait_for_connection`, and a post-update health check must pass before the host is considered successfully patched. | [`15_os_patch_rolling_update.yml`](playbooks/patch_management/15_os_patch_rolling_update.yml) | [README](roles/os_patch_rolling_update/README.md) |
| 16 | `kernel_update_reboot` | Updates the kernel package to the latest available version, determines whether a reboot is actually required by comparing the currently-running kernel (`uname -r`) against the newest kernel installed on disk, and — if a reboot is needed — reboots the host using the **`ansible.builtin.reboot`** module (never a raw shell `reboot` command), with `reboot_timeout` and `post_reboot_delay` tuned for datacenter hardware boot times. | [`16_kernel_update_reboot.yml`](playbooks/patch_management/16_kernel_update_reboot.yml) | [README](roles/kernel_update_reboot/README.md) |
| 17 | `patch_compliance_report` | Cross-references currently installed package versions (via `ansible.builtin.package_facts`) against a `required_patch_baseline` dict of `package -> minimum version`, using Ansible's `version()` Jinja test for robust version comparison across both RPM and dpkg version schemes. | [`17_patch_compliance_report.yml`](playbooks/patch_management/17_patch_compliance_report.yml) | [README](roles/patch_compliance_report/README.md) |
| 18 | `security_only_patching` | Applies only security-classified updates instead of a full OS upgrade, for teams that want a lighter-touch, faster-cadence patch cycle between full maintenance windows. | [`18_security_only_patching.yml`](playbooks/patch_management/18_security_only_patching.yml) | [README](roles/security_only_patching/README.md) |
| 19 | `patch_rollback_snapshot` | Provides a two-mode safety net around patching. | [`19_patch_rollback_snapshot.yml`](playbooks/patch_management/19_patch_rollback_snapshot.yml) | [README](roles/patch_rollback_snapshot/README.md) |
| 20 | `repo_management` | Manages package repository definitions fleet-wide from a single unified `repo_definitions` list var, covering both RedHat family (`.repo` files via `ansible.builtin.yum_repository`, GPG import via `ansible.builtin.rpm_key`) and Debian family (apt `sources.list.d` entries, preferring the modern `ansible.builtin.deb822_repository` format on Ubuntu 22.04+/Debian 12+, with a legacy one-line `apt_repository` + `apt_key` fallback path for older releases). | [`20_repo_management.yml`](playbooks/patch_management/20_repo_management.yml) | [README](roles/repo_management/README.md) |
| 21 | `unattended_upgrades_config` | Configures a fully automated patching cadence so hosts self-patch on a schedule without a human running an Ansible play every time. | [`21_unattended_upgrades_config.yml`](playbooks/patch_management/21_unattended_upgrades_config.yml) | [README](roles/unattended_upgrades_config/README.md) |
| 22 | `patch_maintenance_window` | Orchestration guard role meant to be included at the **top** of other patch playbooks. | [`22_patch_maintenance_window.yml`](playbooks/patch_management/22_patch_maintenance_window.yml) | [README](roles/patch_maintenance_window/README.md) |
| 23 | `package_version_pinning` | Locks specific packages at a fixed, known-good version so they are excluded from any subsequent `dnf upgrade`/`apt upgrade` run, driven entirely by a single `pinned_packages` list var. | [`23_package_version_pinning.yml`](playbooks/patch_management/23_package_version_pinning.yml) | [README](roles/package_version_pinning/README.md) |
| 24 | `third_party_repo_hardening` | Safely manages third-party package repositories so they extend the OS package set without ever silently shadowing (and thereby replacing) trusted OS base packages. | [`24_third_party_repo_hardening.yml`](playbooks/patch_management/24_third_party_repo_hardening.yml) | [README](roles/third_party_repo_hardening/README.md) |
| 25 | `patch_notification_alerting` | Closes the loop on a patch run by aggregating per-host results — packages updated, reboot-required status, success/fail — across every host in the play (`ansible_play_hosts_all` + `hostvars`) and POSTing a single JSON summary to a configurable webhook (Slack, Microsoft Teams, or a generic JSON receiver) using `ansible.builtin.uri`, executed once at play end via `delegate_to: localhost` / `run_once: true`. | [`25_patch_notification_alerting.yml`](playbooks/patch_management/25_patch_notification_alerting.yml) | [README](roles/patch_notification_alerting/README.md) |

### Application Deployment (13)

| # | Role | Purpose | Playbook | Role docs |
|---|---|---|---|---|
| 26 | `nginx_deployment` | Installs nginx and manages it as a data-driven collection of virtual hosts. | [`26_nginx_deployment.yml`](playbooks/app_deployment/26_nginx_deployment.yml) | [README](roles/nginx_deployment/README.md) |
| 27 | `apache_deployment` | Installs Apache (`httpd` on RedHat family, `apache2` on Debian family) and manages a data-driven set of virtual hosts from the `apache_vhosts` list variable. | [`27_apache_deployment.yml`](playbooks/app_deployment/27_apache_deployment.yml) | [README](roles/apache_deployment/README.md) |
| 28 | `docker_engine_setup` | Installs Docker Engine from Docker's official upstream repository (GPG-key import + repo-add, RedHat via `ansible.builtin.yum_repository`, Debian via `ansible.builtin.apt_repository` with a keyring-based `signed-by` pin), templates `/etc/docker/daemon.json` (log driver/size limits, storage driver, registry mirrors, insecure registries, address pools), adds specified users to the `docker` group for passwordless CLI access, and enables the service. | [`28_docker_engine_setup.yml`](playbooks/app_deployment/28_docker_engine_setup.yml) | [README](roles/docker_engine_setup/README.md) |
| 29 | `kubernetes_node_bootstrap` | Prepares a Linux host to join a Kubernetes cluster as a worker or control-plane node: disables swap (runtime `swapoff -a` plus a commented-out fstab entry so it stays off across reboots), loads the `br_netfilter` and `overlay` kernel modules, applies the sysctl settings required for bridged network traffic to reach iptables, adds the official Kubernetes and Docker (for `containerd.io`) package repositories, installs `containerd` with the systemd cgroup driver enabled, and installs `kubelet`/`kubeadm`/`kubectl` pinned to an exact version (held against unattended upgrades on Debian via `apt-mark hold`). | [`29_kubernetes_node_bootstrap.yml`](playbooks/app_deployment/29_kubernetes_node_bootstrap.yml) | [README](roles/kubernetes_node_bootstrap/README.md) |
| 30 | `nodejs_app_deployment` | Installs Node.js (NodeSource repository package by default, or per-user `nvm` when `nodejs_install_method: nvm`), deploys an application release into a timestamped `releases/<id>` directory (from a `git` checkout or a tarball artifact), runs `npm ci --production` via `community.general.npm`, symlinks `current -> releases/<id>`, templates and manages a systemd unit, and performs a post-deploy HTTP health check. | [`30_nodejs_app_deployment.yml`](playbooks/app_deployment/30_nodejs_app_deployment.yml) | [README](roles/nodejs_app_deployment/README.md) |
| 31 | `java_app_deployment` | Installs a pinned OpenJDK version and deploys a Java application in one of three modes selected by `app_type`: a standalone `jar` or `war` run under a custom systemd unit (the default paths), or a full Apache Tomcat installation (`app_type: tomcat`) with the WAR artifact dropped into `webapps/`. | [`31_java_app_deployment.yml`](playbooks/app_deployment/31_java_app_deployment.yml) | [README](roles/java_app_deployment/README.md) |
| 32 | `mysql_database_deployment` | Installs MySQL or MariaDB (selectable via `mysql_flavor`), templates a `my.cnf` tuning override file (InnoDB buffer pool/log sizing, slow query log, character set), bootstraps the root password from an external secrets manager (never inline), removes anonymous accounts, and creates application databases/users from `mysql_databases`/`mysql_users` list variables using `community.mysql.mysql_db` and `community.mysql.mysql_user`. | [`32_mysql_database_deployment.yml`](playbooks/app_deployment/32_mysql_database_deployment.yml) | [README](roles/mysql_database_deployment/README.md) |
| 33 | `postgresql_database_deployment` | Installs PostgreSQL from the official PGDG repository (RedHat via `dnf` RPM install + disabling the distro's built-in `postgresql` module to avoid version conflicts, Debian via `apt_repository` with a keyring-based `signed-by` pin), initializes the cluster, templates `postgresql.conf` and `pg_hba.conf` from role variables, and manages databases/roles with `community.postgresql.postgresql_db` / `postgresql_user`. | [`33_postgresql_database_deployment.yml`](playbooks/app_deployment/33_postgresql_database_deployment.yml) | [README](roles/postgresql_database_deployment/README.md) |
| 34 | `redis_deployment` | Installs Redis and templates `redis.conf` (bind address, memory limit and eviction policy, RDB/AOF/both persistence mode). | [`34_redis_deployment.yml`](playbooks/app_deployment/34_redis_deployment.yml) | [README](roles/redis_deployment/README.md) |
| 35 | `haproxy_load_balancer` | Installs HAProxy and templates `haproxy.cfg` from a `haproxy_backends` list variable — each entry defines a frontend bind address and a backend whose server list is generated automatically from an Ansible inventory group (`backend_group`), so adding/removing a host from that group changes the load-balanced pool on the next run. | [`35_haproxy_load_balancer.yml`](playbooks/app_deployment/35_haproxy_load_balancer.yml) | [README](roles/haproxy_load_balancer/README.md) |
| 36 | `blue_green_deployment` | Generic, reusable blue/green deployment orchestrator. | [`36_blue_green_deployment.yml`](playbooks/app_deployment/36_blue_green_deployment.yml) | [README](roles/blue_green_deployment/README.md) |
| 37 | `systemd_service_deployment` | Generic, reusable role for deploying any custom systemd-managed process. | [`37_systemd_service_deployment.yml`](playbooks/app_deployment/37_systemd_service_deployment.yml) | [README](roles/systemd_service_deployment/README.md) |
| 38 | `artifact_registry_deployment` | Deploys a versioned application artifact (tarball/zip) sourced from a Nexus/Artifactory repository over HTTPS (`ansible.builtin.get_url` with a bearer token) or from S3 (`amazon.aws.s3_object`), verifies its checksum, unpacks it into `releases/<app_version>`, atomically re-points the `current` symlink at the new release, prunes releases beyond `artifact_keep_releases` (kept for fast rollback — just re-point `current` at an older `releases/<version>` directory), and restarts the target systemd service via a notify handler. | [`38_artifact_registry_deployment.yml`](playbooks/app_deployment/38_artifact_registry_deployment.yml) | [README](roles/artifact_registry_deployment/README.md) |

### Disaster Recovery (12)

| # | Role | Purpose | Playbook | Role docs |
|---|---|---|---|---|
| 39 | `backup_restic_orchestration` | Installs `restic` (via distro package or a pinned upstream binary release), initializes an S3-compatible backup repository on first run (or reuses an existing one), runs backups of a configurable list of filesystem paths, applies `--keep-daily/--keep-weekly/--keep-monthly/ --keep-yearly` retention with `forget --prune`, and verifies repository integrity with `restic check`. | [`39_backup_restic_orchestration.yml`](playbooks/disaster_recovery/39_backup_restic_orchestration.yml) | [README](roles/backup_restic_orchestration/README.md) |
| 40 | `database_backup_restore` | Two-mode role (`db_backup_action: backup` or `restore`) for MySQL and PostgreSQL. In backup mode it runs `mysqldump`/`pg_dump` to a timestamped, gzip-compressed file and ships it to an S3 bucket or NFS backup destination, then prunes local dumps past `db_dump_retention_days`. | [`40_database_backup_restore.yml`](playbooks/disaster_recovery/40_database_backup_restore.yml) | [README](roles/database_backup_restore/README.md) |
| 41 | `etcd_cluster_backup_restore` | Manages etcd cluster state backup and recovery for Kubernetes control-plane nodes. | [`41_etcd_cluster_backup_restore.yml`](playbooks/disaster_recovery/41_etcd_cluster_backup_restore.yml) | [README](roles/etcd_cluster_backup_restore/README.md) |
| 42 | `dr_failover_orchestration` | Orchestrates failover of production traffic to the `dr_site_group`. | [`42_dr_failover_orchestration.yml`](playbooks/disaster_recovery/42_dr_failover_orchestration.yml) | [README](roles/dr_failover_orchestration/README.md) |
| 43 | `lvm_snapshot_management` | Creates, removes, and reverts LVM snapshots of specified volume groups/logical volumes — typically run immediately before a risky patch or deployment so a fast rollback point exists. | [`43_lvm_snapshot_management.yml`](playbooks/disaster_recovery/43_lvm_snapshot_management.yml) | [README](roles/lvm_snapshot_management/README.md) |
| 44 | `config_backup_git` | Periodically mirrors a configurable list of `/etc` configuration files/directories into a local git working tree, commits them only when `git status --porcelain` shows real changes, and pushes to a bare repository on a remote backup host over SSH. This provides fast change-tracking and a quick config-only disaster recovery path independent of full-system backups. | [`44_config_backup_git.yml`](playbooks/disaster_recovery/44_config_backup_git.yml) | [README](roles/config_backup_git/README.md) |
| 45 | `dr_test_automation` | Runs a non-destructive DR readiness checklist meant to be scheduled regularly (e.g. daily via cron/AWX) to validate disaster-recovery posture without performing an actual failover. | [`45_dr_test_automation.yml`](playbooks/disaster_recovery/45_dr_test_automation.yml) | [README](roles/dr_test_automation/README.md) |
| 46 | `replication_health_check` | Checks database replication lag — MySQL via `SHOW REPLICA STATUS` (`Seconds_Behind_Source`) or PostgreSQL via `pg_last_xact_replay_timestamp()` — against `replication_lag_warning_seconds`, and filesystem/storage replication freshness by comparing the most recently modified file timestamp on the primary path against the same path on a DR-site mirror host against `replication_fs_max_staleness_seconds`. | [`46_replication_health_check.yml`](playbooks/disaster_recovery/46_replication_health_check.yml) | [README](roles/replication_health_check/README.md) |
| 47 | `backup_restore_verification` | Proves backups are actually restorable — not just completed. | [`47_backup_restore_verification.yml`](playbooks/disaster_recovery/47_backup_restore_verification.yml) | [README](roles/backup_restore_verification/README.md) |
| 48 | `dr_runbook_execution` | The single top-level DR orchestration entry point. | [`48_dr_runbook_execution.yml`](playbooks/disaster_recovery/48_dr_runbook_execution.yml) | [README](roles/dr_runbook_execution/README.md) |
| 49 | `cross_region_data_sync` | Periodically syncs application data directories from primary-site hosts to DR-site hosts. | [`49_cross_region_data_sync.yml`](playbooks/disaster_recovery/49_cross_region_data_sync.yml) | [README](roles/cross_region_data_sync/README.md) |
| 50 | `incident_snapshot_capture` | On-demand diagnostic capture role for incident postmortems. | [`50_incident_snapshot_capture.yml`](playbooks/disaster_recovery/50_incident_snapshot_capture.yml) | [README](roles/incident_snapshot_capture/README.md) |

## Design conventions used throughout

- **Multi-distro branching:** every role that differs between families branches on
  `ansible_facts['os_family']` (`RedHat` or `Debian`), never on `ansible_distribution` directly,
  so RHEL/CentOS/Rocky/Alma and Ubuntu/Debian are each handled with one shared code path per family.
- **FQCN everywhere:** all module references use fully-qualified collection names
  (`ansible.builtin.*`, `community.general.*`, etc.) — no short module names.
- **No hardcoded secrets:** every password, token, or key is resolved via a `lookup()` call against
  `community.hashi_vault` or `amazon.aws.aws_secret` at task-execution time, gated by `no_log: true`
  on the resolving task.
- **Idempotency first:** state-based modules (`present`/`absent`/`enabled`) are preferred over
  `command`/`shell`; where a raw command is unavoidable, a `changed_when`/`creates`/`removes` guard
  makes the task idempotent and accurately reported.
- **Handlers, not inline restarts:** service restarts/reloads triggered by config changes go
  through `notify:` + a role handler, never an unconditional task-level restart.
- **A role cannot re-target a different host than its including play.** Any workflow that needs to
  act on a different host set mid-flow (e.g. blue/green cutover, DR failover promoting a database
  on a different host) is structured as a **multi-play wrapper playbook** — a play (often
  `localhost`) that computes facts / uses `add_host`, a play that targets the actual host group for
  the action, and a closing play for any remaining orchestration logic. See
  `playbooks/app_deployment/36_blue_green_deployment.yml` and
  `playbooks/disaster_recovery/42_dr_failover_orchestration.yml` for the reference pattern —
  `ansible.builtin.include_role` combined with `delegate_to` is invalid Ansible and will fail
  `--syntax-check`.

## Validation performed

Every role's task/handler/template/vars/defaults YAML was parsed for syntax validity, and all 50
individual wrapper playbooks plus the 4 category `site-*.yml` files and the master `site.yml` were
validated with a real `ansible-core` install via `ansible-playbook --syntax-check` (not just YAML
parsing — genuine Ansible semantic validation), exercising the major conditional branches in each
role via `-e` overrides. No hardcoded secrets, no unresolved TODO/placeholder markers, and no
`notify:` references without a matching handler were found.

## References

- [Ansible documentation](https://docs.ansible.com/ansible/latest/index.html)
- [Ansible Galaxy collections index](https://galaxy.ansible.com/)
- [community.hashi_vault collection docs](https://docs.ansible.com/projects/ansible/latest/collections/community/hashi_vault/index.html)
- [amazon.aws collection docs](https://docs.ansible.com/projects/ansible/latest/collections/amazon/aws/index.html)
- [Ansible best practices guide](https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html)
