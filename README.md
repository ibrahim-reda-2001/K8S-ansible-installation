### Overview ###

This guide explains how to install and configure a **highly available (HA) Kubernetes cluster** using Ansible, following the [kubeadm HA topology](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/) with an **external etcd cluster**.

Topology:

- **3 control-plane (master) nodes** — kube-apiserver, controller-manager, scheduler
- **etcd runs as a systemd service** on the 3 master nodes (NOT as static pods) — kubeadm is configured with `etcd.external`
- **1 haproxy load balancer node** — provides the stable `controlPlaneEndpoint` (TCP :6443) in front of the 3 API servers
- **3 worker nodes**

Using this setup, Ansible will:

1. Prepare the OS, container runtime (containerd) and Kubernetes packages on all masters and workers
2. Install and configure haproxy on the load balancer node
3. Generate a dedicated etcd CA + TLS certificates (server/peer per node + `apiserver-etcd-client`) and deploy etcd as a **systemd service** cluster on the 3 masters
4. Run `kubeadm init` on the first master with a config file pointing at the external etcd endpoints and the load balancer endpoint
5. Join the remaining 2 masters with `--control-plane --certificate-key`
6. Join the worker nodes
7. Deploy Calico CNI

## Requirements ##

- Ubuntu hosts, SSH access with the user configured in `ansible.cfg` (default `ubuntu`)
- The `community.crypto` collection on the Ansible control node (used for etcd TLS certs):

```
ansible-galaxy collection install community.crypto
```

## 1- Install ansible ##

```
$ sudo apt update
$ sudo apt install software-properties-common
$ sudo add-apt-repository --yes --update ppa:ansible/ansible
$ sudo apt install ansible
```

## 2- Private key ##

Put your private key path in `ansible.cfg`:
```
private_key_file = ./key.pem
chmod 400 key.pem
```

## 3- Edit inventory.ini with your IPs ##

```ini
[masters]
master1 ansible_host=192.168.1.11
master2 ansible_host=192.168.1.12
master3 ansible_host=192.168.1.13

[workers]
worker1 ansible_host=192.168.1.21
worker2 ansible_host=192.168.1.22
worker3 ansible_host=192.168.1.23

[loadbalancer]
lb1 ansible_host=192.168.1.10
```

## 4- Verify inventory and reachability ##

```
ansible-inventory --graph
ansible all -m ping
```

## 5- Run the playbook ##

```
ansible-playbook playbook.yml
```

## 6- Verify the cluster ##

On any master:

```
kubectl get nodes -o wide
```

Check the etcd cluster (systemd service):

```
systemctl status etcd

sudo etcdctl endpoint status --cluster -w table \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/pki/ca.crt \
  --cert=/etc/etcd/pki/server.crt \
  --key=/etc/etcd/pki/server.key
```

## Notes ##

- Shared settings live in `group_vars/all.yml` — edit `k8s_version` (apt repo minor version for kubelet/kubeadm/kubectl), `calico_version`, `etcd_version`, `pod_network_cidr` there.
- The API server endpoint used by all kubeconfigs is the haproxy node (`<lb-ip>:6443`), so any single master can fail without losing API access.
- etcd data lives in `/var/lib/etcd`, TLS material in `/etc/etcd/pki`. kubeadm's external-etcd certs are placed at `/etc/kubernetes/pki/etcd/ca.crt` and `/etc/kubernetes/pki/apiserver-etcd-client.{crt,key}` on every master.
