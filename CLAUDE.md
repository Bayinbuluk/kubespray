# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Kubespray is an Ansible-based Kubernetes cluster lifecycle management tool — part of `kubernetes-sigs`. It deploys production-ready clusters (Kubernetes 1.36.0) on bare metal and cloud (AWS, GCE, Azure, OpenStack, vSphere, etc.).

## Quick reference

```bash
# Deploy a cluster
ansible-playbook -i inventory/mycluster/hosts.yml cluster.yml

# Scale up / remove nodes
ansible-playbook -i inventory/mycluster/hosts.yml scale.yml
ansible-playbook -i inventory/mycluster/hosts.yml remove-node.yml

# Upgrade / reset
ansible-playbook -i inventory/mycluster/hosts.yml upgrade-cluster.yml
ansible-playbook -i inventory/mycluster/hosts.yml reset.yml

# Lint everything (requires pre-commit)
pre-commit run --all-files

# Run molecule tests for a single role
cd roles/container-engine/containerd && molecule test

# Run all molecule tests
tests/scripts/molecule_run.sh

# Local dev with Vagrant
vagrant up
```

Requirements: Ansible >=2.18.0,<2.19.0 (`meta/runtime.yml`), Python packages from `requirements.txt` (ansible, cryptography, jmespath, netaddr), plus `tests/requirements.txt` for testing (molecule, pytest-testinfra).

## Architecture

### Playbook flow

Top-level `.yml` files are thin wrappers that import playbooks from `playbooks/`. The main deployment flow (`playbooks/cluster.yml`):

1. `boilerplate.yml` — dynamic inventory groups, bastion SSH config (run for every operation)
2. `internal_facts.yml` — bootstrap OS, gather hardware/network facts
3. **etcd phase** — `kubespray_defaults` → `kubernetes/preinstall` → `container-engine` → `download` → `install_etcd.yml`
4. **node phase** — `kubespray_defaults` → `kubernetes/node` (kubelet)
5. **control-plane phase** — `kubespray_defaults` → `kubernetes/control-plane` → `kubernetes/client` → `kubernetes-apps/cluster_roles`
6. **kubeadm + CNI** — `kubespray_defaults` → `kubernetes/kubeadm` → `network_plugin` (calico/cilium/flannel/etc.)
7. **apps phase** — `kubernetes-apps` (coredns, argocd, helm, metallb, etc.)

The `kubespray_defaults` role is called at the start of **every play** — it's the central variable authority, not a task executor.

### Variable architecture

- **`roles/kubespray_defaults/defaults/main/main.yml`** — global defaults: Ansible SSH config, Kubernetes version (`kube_version`), proxy mode, swap, etc.
- **`roles/kubespray_defaults/defaults/main/download.yml`** — ALL component checksums and versions (kubelet, etcd, calico, cilium, containerd, coredns, helm, etc.). Component versions are derived from checksum dicts (e.g., `kube_version: "{{ (kubelet_checksums['amd64'] | dict2items)[0].key }}"` maps to the newest version in the checksums dict).
- **`inventory/sample/group_vars/`** — user-facing overrides:
  - `all/all.yml` — bin_dir, cloud_provider, proxy, NTP
  - `all/aws.yml`, `all/azure.yml`, etc. — cloud-specific
  - `k8s_cluster/k8s-cluster.yml` — network plugin, DNS, container_manager, kubeadm patches, resource reservations
  - `k8s_cluster/k8s-net-*.yml` — per-CNI settings
  - `k8s_cluster/addons.yml` — addon versions and configuration

Each role also has its own `defaults/main.yml` for role-specific defaults.

### Key roles and responsibilities

| Role | Purpose |
|---|---|
| `kubespray_defaults` | Central default variables (called in every play) |
| `bootstrap_os` | OS preparation, package installs |
| `container-engine/` | Container runtime (containerd default; also docker, cri-o, gvisor, kata, youki) |
| `download` | Pre-download container images and binaries |
| `etcd` | etcd cluster deployment |
| `kubernetes/control-plane` | kube-apiserver, scheduler, controller-manager |
| `kubernetes/node` | kubelet installation |
| `kubernetes/kubeadm` | kubeadm init/join |
| `network_plugin/` | CNI deployment (calico default, cilium, flannel, kube-ovn, kube-router, macvlan, multus, custom_cni) |
| `kubernetes-apps/` | coredns, argocd, helm, metallb, cert-manager, ingress, CSI drivers, etc. |
| `validate_inventory` | Pre-flight inventory validation |
| `upgrade/` | Pre-upgrade and post-upgrade tasks |
| `reset/` | Cluster teardown |
| `remove_node/` | Node removal + cleanup |

### Collection structure

This is an Ansible collection (`kubernetes_sigs.kubespray`, version in `galaxy.yml`). Custom modules live in `plugins/modules/` (with `library/` symlinked for legacy). Collection dependencies in `galaxy.yml`: `ansible.utils`, `community.crypto`, `community.general`, `ansible.netcommon`, `ansible.posix`, `community.docker`, `kubernetes.core`.

### Directory quick-reference

- `playbooks/` — Core Ansible playbooks (cluster, scale, upgrade, reset, remove_node, recover_control_plane)
- `roles/` — 27+ Ansible roles
- `inventory/sample/` — Reference inventory with group_vars
- `contrib/` — Terraform modules, offline/air-gap tools, cloud inventory scripts
- `tests/` — Molecule configs, test cases, helper scripts, Makefile
- `scripts/` — Utility scripts (CI matrix, galaxy version check, docs sidebar)
- `docs/` — Documentation (20+ topic directories)
- `extra_playbooks/` — Supplementary playbooks

## Conventions

- YAML: 2-space indent, no trailing whitespace, no line-length limits (`.editorconfig`, `.yamllint`)
- Role names use underscores (`bootstrap_os`, `network_plugin`, `kubespray_defaults`); a few legacy symlinks with hyphens exist
- Pre-commit runs: yamllint (strict), ansible-lint, shellcheck, markdownlint, misspell, jinja syntax check, plus custom hooks (collection build, docs sidebar, CI matrix, galaxy version, propagate variables, sorted checksums)
- CI uses GitLab CI (not GitHub Actions for testing) — stages: build → test → deploy-part1 → deploy-extended
- GitHub Actions only for issue labeling and automated patch version updates
- Component updates: edit checksums in `roles/kubespray_defaults/defaults/main/download.yml`, run `pre-commit run propagate-ansible-variables` to sync variables into documentation/inventory files
