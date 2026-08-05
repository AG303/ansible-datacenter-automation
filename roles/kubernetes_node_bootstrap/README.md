# kubernetes_node_bootstrap

**Category:** Application Deployment
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Prepares a Linux host to join a Kubernetes cluster as a worker or control-plane node:
disables swap (runtime `swapoff -a` plus a commented-out fstab entry so it stays off across
reboots), loads the `br_netfilter` and `overlay` kernel modules, applies the sysctl settings
required for bridged network traffic to reach iptables, adds the official Kubernetes and
Docker (for `containerd.io`) package repositories, installs `containerd` with the systemd
cgroup driver enabled, and installs `kubelet`/`kubeadm`/`kubectl` pinned to an exact version
(held against unattended upgrades on Debian via `apt-mark hold`). The actual `kubeadm join`
step is out of scope by default and only runs when `kubernetes_join_cluster_enabled: true`
and a `kubernetes_join_command` is supplied — the role leaves the node "join-ready" for a
separate control-plane-driven step otherwise.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `kubernetes_version` | `"1.29.5"` | Pinned Kubernetes minor/patch version (drives the repo URL) |
| `kubernetes_disable_swap` | `true` | Disables swap at runtime and comments it out of fstab |
| `kubernetes_kernel_modules` | `[br_netfilter, overlay]` | Kernel modules loaded and persisted |
| `kubernetes_sysctl_settings` | bridge-nf-call + ip_forward | sysctl keys applied for container networking |
| `kubernetes_containerd_use_systemd_cgroup` | `true` | Sets `SystemdCgroup = true` in containerd's config.toml (required by kubelet cgroupDriver=systemd) |
| `kubernetes_join_cluster_enabled` | `false` | Opt-in gate for running `kubeadm join` in this same play |
| `kubernetes_join_command` | `""` | Full `kubeadm join ...` string (sourced from the secrets manager or a control-plane fact) |
| `vault_addr`, `vault_kv_mount`, `vault_secret_path` | see defaults | HashiCorp Vault KV v2 lookup coordinates for the join command |
| `aws_region` | `us-east-1` | Region for AWS Secrets Manager lookup pattern |

## Example play

```yaml
- hosts: k8s_worker_nodes
  become: true
  roles:
    - role: kubernetes_node_bootstrap
      vars:
        kubernetes_version: "1.29.5"
        kubernetes_join_cluster_enabled: true
        kubernetes_join_command: "{{ lookup('community.hashi_vault.vault_kv2_get',
                                        'datacenter/kubernetes/join/' + inventory_hostname,
                                        engine_mount_point='kv', url=vault_addr).secret.join_command }}"
```

## Secrets manager pattern (join token retrieval)

```yaml
# HashiCorp Vault (community.hashi_vault)
kubernetes_join_command: "{{ lookup('community.hashi_vault.vault_kv2_get', vault_secret_path,
                                engine_mount_point=vault_kv_mount, url=vault_addr).secret.join_command }}"

# AWS Secrets Manager (amazon.aws)
kubernetes_join_command: "{{ lookup('amazon.aws.aws_secret', vault_secret_path, region=aws_region) }}"
```

## Notes

- Join tokens are short-lived by default in kubeadm; regenerate the stored secret
  (`kubeadm token create --print-join-command`) before rolling this role out with
  `kubernetes_join_cluster_enabled: true` against a fleet.
- SELinux is set to permissive on RedHat-family nodes per current upstream kubeadm
  guidance for kubelet/CNI compatibility.

## References

- [Kubernetes kubeadm install docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [containerd getting-started docs](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)
- [community.general.modprobe module docs](https://docs.ansible.com/ansible/latest/collections/community/general/modprobe_module.html)
- [ansible.posix.sysctl module docs](https://docs.ansible.com/ansible/latest/collections/ansible/posix/sysctl_module.html)
