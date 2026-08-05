# package_version_pinning

**Category:** Patch Management
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Locks specific packages at a fixed, known-good version so they are excluded
from any subsequent `dnf upgrade`/`apt upgrade` run, driven entirely by a
single `pinned_packages` list var. On RHEL family this installs
`python3-dnf-plugin-versionlock` and manages entries with `dnf versionlock
add`/`delete`. On Debian family it uses `ansible.builtin.dpkg_selections` to
apply/release `apt-mark hold`. If a `version` is specified for an entry, the
role first ensures that exact version is installed (via `allow_downgrade`)
before locking it, so pinning can also be used to pin-and-downgrade in one
run. Set `package_version_pinning_state: absent` to release all pins/holds
managed by this role.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `pinned_packages` | sample list (`kernel`, `docker-ce`) | List of `{name, version}` dicts; blank `version` locks whatever is currently installed |
| `package_version_pinning_state` | `"present"` | `"present"` to apply pins/holds, `"absent"` to release them |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: package_version_pinning
      vars:
        pinned_packages:
          - name: docker-ce
            version: "5:24.0.9-1"
          - name: kubelet
            version: ""
```

## Secrets manager pattern

This role does not require any secrets — it only manages local package
manager pin/hold state. No Vault/AWS Secrets Manager lookups are used.

## Notes

- Pin the exact package name as it appears to the package manager (e.g. the
  full `dpkg` binary package name, or the `dnf`/`yum` package name) —
  wildcards are supported by `dnf versionlock` (e.g. `kernel-*`) but not by
  `apt-mark hold`.
- Remember to run this role with `package_version_pinning_state: absent`
  before a planned major-version upgrade that intentionally needs to move
  a pinned package forward.
- Combine with `os_patch_rolling_update`'s `os_patch_exclude_packages` for
  belt-and-suspenders protection: versionlock/hold prevents ANY upgrade
  path (including manual `dnf install newer-version`) from touching the
  package, while `os_patch_exclude_packages` only skips it during that one
  role's run.

## References

- [DNF versionlock plugin documentation](https://dnf-plugins-core.readthedocs.io/en/latest/versionlock.html)
- [ansible.builtin.dpkg_selections module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/dpkg_selections_module.html)
- [ansible.builtin.apt module docs (allow_downgrade)](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)
