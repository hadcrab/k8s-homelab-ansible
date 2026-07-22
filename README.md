# Automated Kubernetes Cluster Deployment

Ansible playbook for automated k8s cluster provisioning on virtual machines.

Single control-plane, multi-worker topology. Debian 13 target hosts. containerd runtime, Flannel CNI.

All application-level components (MetalLB, Traefik, cert-manager, Longhorn, monitoring, workloads) are managed via GitOps in a separate repository, not by this playbook.

## Requirements

- Ansible 2.12+
- Python 3.10+ on control node
- Target hosts: Debian 13, reachable via SSH
- Vault-encrypted credentials in `group_vars/all/vault.yml`

```bash
ansible-galaxy collection install -r requirements.yml
```

Collections: `kubernetes.core >=6.5.0`, `ansible.posix`

## Inventory

Example `hosts.ini`:

```ini
[control_plane]
cp-1 ansible_host=192.168.1.10

[workers]
worker-1 ansible_host=192.168.1.11
worker-2 ansible_host=192.168.1.12

[all:vars]
ansible_user={{ vault_ansible_user }}
ansible_password={{ vault_ansible_password }}
ansible_become_method=sudo
ansible_become_user=root
ansible_become_password={{ vault_ansible_become_password }}
```

## Usage

Run the full playbook:

```bash
ansible-playbook site.yml --ask-vault-pass
```

Run a specific stage by tag:

```bash
ansible-playbook site.yml --tags install
ansible-playbook site.yml --tags init
ansible-playbook site.yml --tags workers
ansible-playbook site.yml --tags sysapps
ansible-playbook site.yml --tags longhorn-prereqs
```

## What It Does

1. **Prepare OS** - disables swap, loads kernel modules (`overlay`, `br_netfilter`, `vxlan`), applies sysctl for bridge networking, installs containerd with `SystemdCgroup = true`, configures sandbox image to `registry.k8s.io/pause:3.10`.

2. **Install k8s packages** - adds Kubernetes apt repository, installs `kubeadm`, `kubelet`, `kubectl` at pinned version.

3. **Init control plane** - runs `kubeadm init` with pod CIDR `10.244.0.0/16`, copies admin config to `~control/.kube/config`, deploys Flannel CNI v0.25.5, generates worker join command.

4. **Join workers** - executes the join command (skips if node already joined).

5. **Deploy apps** - installs ArgoCD via Helm, bootstraps a root Application that watches `argocd/` directory in the manifests repo.

6. **Install Longhorn prerequisites** - installs `open-iscsi`, `nfs-common`, `curl` on all nodes. Longhorn itself is deployed via ArgoCD.
