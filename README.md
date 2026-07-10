# Kubernetes Cluster Installation with Ansible (single master)

This project installs and configures a Kubernetes cluster — **one master node and multiple worker nodes** — using Ansible, in a consistent, repeatable, and idempotent way.

What the playbook does:

1. Prepares the OS on every node — containerd runtime, CNI plugins, swap off, kernel modules, sysctl
2. Installs `kubelet`, `kubeadm`, `kubectl` (version-pinned, packages held) with bash completion for all users
3. Initializes the control plane with `kubeadm init` on the master
4. Deploys the Calico CNI
5. Joins the worker nodes automatically
6. Validates each stage — API server `/readyz`, Calico rollout, CoreDNS available, every node `Ready`

## Choosing versions

All versions live in [`group_vars/all.yml`](group_vars/all.yml) — edit them there, nothing is hardcoded in the roles:

| Variable | Default | Meaning |
| --- | --- | --- |
| `k8s_version` | `"1.31"` | Kubernetes **minor** version — selects the pkgs.k8s.io apt repo for kubelet/kubeadm/kubectl (latest patch of that minor is installed) |
| `calico_version` | `"v3.30.2"` | Calico release tag (keep the `v` prefix) |
| `cni_plugins_version` | `"v1.4.0"` | containernetworking plugins release tag |
| `pod_network_cidr` | `"192.168.0.0/16"` | Pod network passed to `kubeadm init` (Calico's default) |

The playbook also **verifies** the installed kubeadm and the running control plane actually match `k8s_version`, and fails loudly if they don't.

## Requirements

- Ubuntu hosts reachable over SSH as the user set in `ansible.cfg` (default `ubuntu`)
- An Ansible control node (can be any machine with SSH access to all nodes)

## 1 - Install Ansible on the control node

```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

## 2 - SSH private key

Put your private key path in `ansible.cfg`:

```ini
private_key_file = ./key.pem
```

```bash
# on AWS, restrict permissions first
chmod 400 key.pem
```

## 3 - Map hostnames to IPs

Edit `/etc/hosts` on the control node with your nodes' IPs, matching the names in `inventory.ini`:

```text
127.0.0.1     localhost
18.205.246.4  master1
54.167.62.72  worker1
54.152.204.6  worker2
3.88.104.5    worker3
```

## 4 - Verify inventory and reachability

```bash
ansible-inventory --graph
```

```text
@all:
  |--@ungrouped:
  |--@master:
  |  |--master1
  |--@workers:
  |  |--worker1
  |  |--worker2
  |  |--worker3
```

```bash
ansible all -m ping
```

Every host should answer `SUCCESS => "ping": "pong"`.

## 5 - Run the playbook

```bash
ansible-playbook playbook.yml
```

The playbook is **idempotent**: running it a second time on a healthy cluster should finish with `changed=0` on every host. That also makes it safe to re-run after a failure — completed steps are skipped, only the missing ones execute.

## 6 - Verify the cluster

On the master:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

All nodes should be `Ready` and all `kube-system` pods `Running`. Tab completion for `kubectl` works for every user out of the box (new login shell required after first install).

## Project layout

```text
playbook.yml            # two plays: master (setup, init, CNI, join-token), workers (setup, join)
group_vars/all.yml      # versions and pod CIDR - the only file you normally edit
roles/
  k8s-node-setup/       # OS prep, containerd, kubelet/kubeadm/kubectl, bash completion
  kubeadm/              # kubeadm init + kubeconfig for the SSH user + API validation
  cni/                  # Calico download/apply + rollout validation
  join/                 # join-token generation on master, worker join + Ready validation
```

## Notes

- Kubernetes packages are apt-held after install; changing `k8s_version` affects fresh nodes only — upgrading a live cluster is a separate `kubeadm upgrade` procedure.
- For a **highly available** setup (3 masters, external etcd as a systemd service, haproxy load balancer), see the `master` branch of this repository.
