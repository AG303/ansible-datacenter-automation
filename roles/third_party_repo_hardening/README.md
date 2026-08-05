# third_party_repo_hardening

**Category:** Patch Management
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Safely manages third-party package repositories so they extend the OS
package set without ever silently shadowing (and thereby replacing) trusted
OS base packages. On RHEL family, this installs `epel-release` from the
official Fedora Project URL, imports its GPG key, and — critically —
**disables EPEL by default** (`enabled=0` in each of `epel`,
`epel-debuginfo`, `epel-source`) unless the repo ID is explicitly present in
`epel_allowlisted_repo_ids`; when enabled, it is also given a lower
`priority` via the yum/dnf priorities mechanism so it can never outrank the
OS base repos for a package that exists in both. On Debian family, each
entry in `third_party_repos` (e.g. Docker CE, a vendor PPA-equivalent) is
deployed via the modern `ansible.builtin.deb822_repository` module with an
explicit signing key, and an apt preferences pin file is generated per repo
with a `pin_priority` deliberately set below the default `500` so it never
wins over the OS archive for overlapping package names — with an optional
stricter allow-list pin (`allowed_packages`) that scopes a repo to only the
specific packages it should ever provide.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `epel_install` | `true` | Install EPEL on RedHat family |
| `epel_enabled_by_default` | `false` | Documented intent flag; actual enable/disable is driven by `epel_allowlisted_repo_ids` |
| `epel_allowlisted_repo_ids` | `[]` | Repo IDs (`epel`, `epel-debuginfo`, `epel-source`) explicitly permitted to be enabled |
| `epel_priority` | `10` | yum-priorities value for EPEL — kept higher (lower-priority) than OS base repos (priority 1-2) |
| `yum_priorities_plugin_install` | `true` | Ensure the priorities-capable plugin package is present |
| `third_party_repos` | sample Docker CE entry | List of `{name, baseurl, deb_suite, deb_components, gpgkey, enabled, pin_priority, allowed_packages}` |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: third_party_repo_hardening
      vars:
        epel_allowlisted_repo_ids: ["epel"]
        third_party_repos:
          - name: docker-ce
            baseurl: "https://download.docker.com/linux/ubuntu"
            deb_suite: "jammy"
            deb_components: ["stable"]
            gpgkey: "https://download.docker.com/linux/ubuntu/gpg"
            enabled: true
            pin_priority: 100
            allowed_packages: ["docker-ce", "docker-ce-cli", "containerd.io"]
```

## Secrets manager pattern

This role does not require any secrets for the sample public repos (EPEL,
Docker CE) — GPG keys and baseurls are public. If an internal third-party
mirror requires an authenticated URL, pull the token from the secrets
manager rather than hardcoding it:

```yaml
# HashiCorp Vault (community.hashi_vault)
third_party_repo_token: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                              engine_mount_point=vault_kv_mount, url=vault_addr).secret.token }}"

# AWS Secrets Manager (amazon.aws)
third_party_repo_token: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- Never enable EPEL fleet-wide without an allowlist — its package set can
  silently replace OS base packages that share a name but differ in
  build/patch level, breaking vendor support contracts on RHEL/Rocky/Alma.
- On Debian family, `pin_priority` values below `1000` never trigger a
  downgrade of an already-installed package, and values below the OS
  archive's implicit `500` ensure the third-party repo is only consulted
  when the OS archive doesn't provide the package at all — set even lower
  (`100`) as done here to be conservative.
- Pair with `repo_management` for internal-mirror OS base repos and
  `package_version_pinning` to lock a specific third-party package version
  once validated.

## References

- [EPEL Fedora Project documentation](https://docs.fedoraproject.org/en-US/epel/)
- [DNF/YUM priorities plugin documentation](https://dnf-plugins-core.readthedocs.io/en/latest/priorities.html)
- [ansible.builtin.deb822_repository module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/deb822_repository_module.html)
- [APT preferences (pinning) manual](https://manpages.debian.org/bookworm/apt/apt_preferences.5.en.html)
