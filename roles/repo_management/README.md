# repo_management

**Category:** Patch Management
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Manages package repository definitions fleet-wide from a single unified
`repo_definitions` list var, covering both RedHat family (`.repo` files via
`ansible.builtin.yum_repository`, GPG import via `ansible.builtin.rpm_key`)
and Debian family (apt `sources.list.d` entries, preferring the modern
`ansible.builtin.deb822_repository` format on Ubuntu 22.04+/Debian 12+, with
a legacy one-line `apt_repository` + `apt_key` fallback path for older
releases). Internal mirror URLs are fully supported since `baseurl` is just a
plain var. GPG keys are always imported before the repo file references them,
and metadata caches are refreshed via handler after any change.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `repo_definitions` | sample internal-mirror list | List of repo dicts: `name`, `description`, `baseurl`, `deb_suite`, `deb_components`, `gpgkey`, `gpgcheck`, `enabled`, `priority` |
| `repo_management_clean_metadata_after_change` | `true` | Refresh dnf/apt metadata cache via handler after any repo file changes |
| `repo_management_use_deb822` | `true` | Use `ansible.builtin.deb822_repository` (modern) instead of legacy one-line `apt_repository` + `apt_key` |

### `repo_definitions` item schema

| Field | Applies to | Description |
|---|---|---|
| `name` | both | Identifier, also used as filename stem |
| `description` | RedHat | Human-readable `name=` field in the `.repo` file |
| `baseurl` | both | yum/dnf baseurl, or the apt repo URL |
| `deb_suite` | Debian only (marks entry as a Debian repo) | Suite/codename, e.g. `{{ ansible_facts['distribution_release'] }}` |
| `deb_components` | Debian | List of components, e.g. `["main"]` |
| `deb_line` | Debian (legacy mode only) | Full override one-line `deb [...] url suite components` string |
| `gpgkey` | both | URL to the ASCII-armored GPG key |
| `gpgcheck` | RedHat | Whether to enforce gpgcheck (default `true`) |
| `enabled` | both | Whether the repo is enabled |
| `priority` | both | yum-priorities value (RedHat) / apt pin priority (Debian) |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: repo_management
      vars:
        repo_definitions:
          - name: internal-baseos
            baseurl: "https://repo.internal.example.com/rhel/$releasever/baseos/$basearch"
            gpgkey: "https://repo.internal.example.com/keys/RPM-GPG-KEY-internal"
            enabled: true
```

## Secrets manager pattern

Internal mirror URLs in this role are not secrets by default, but if your
mirror requires an authenticated URL/token, pull it from the secrets manager
rather than hardcoding it in `repo_definitions`:

```yaml
# HashiCorp Vault (community.hashi_vault)
repo_mirror_token: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                          engine_mount_point=vault_kv_mount, url=vault_addr).secret.token }}"

# AWS Secrets Manager (amazon.aws)
repo_mirror_token: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

Then reference `"https://{{ repo_mirror_token }}@repo.internal.example.com/rhel/..."` in `baseurl`
and mark that task `no_log: true`.

## Notes

- On Debian 11/Ubuntu 20.04, `deb822_repository` is not supported by the OS
  apt tooling itself — set `repo_management_use_deb822: false` for those
  releases to use the legacy `apt_repository`/`apt_key` path.
- `repo_definitions` entries are distinguished as Debian-targeted by the
  presence of `deb_suite`; RedHat-targeted entries are matched by `baseurl`
  patterns (`$releasever`, `/rhel/`, `/centos/`, `/rocky/`, `/alma/`).
- Pair with `third_party_repo_hardening` for EPEL/PPA-style repos that need
  restrictive defaults rather than full trust.

## References

- [ansible.builtin.yum_repository module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_repository_module.html)
- [ansible.builtin.rpm_key module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/rpm_key_module.html)
- [ansible.builtin.deb822_repository module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/deb822_repository_module.html)
- [ansible.builtin.apt_repository module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_repository_module.html)
- [ansible.builtin.apt_key module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_key_module.html)
