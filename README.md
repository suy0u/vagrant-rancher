# Kubernetes RKE2 + Rancher + NFS Lab Environment

## Overview

This project deploys a complete Kubernetes laboratory using **Vagrant** and **VirtualBox**.

The environment consists of:

| VM        | IP           | Purpose                            |
| --------- | ------------ | ---------------------------------- |
| rancher   | 10.10.172.10 | Rancher Management Server          |
| nfsserver | 10.10.172.20 | Shared NFS Storage                 |
| node1     | 10.10.172.31 | RKE2 Server (Control Plane + etcd) |
| node2     | 10.10.172.32 | RKE2 Worker                        |
| node3     | 10.10.172.33 | RKE2 Worker                        |

---

# Requirements

Software:

- VirtualBox
- Vagrant
- Internet connection
- At least:
  - 16 GB RAM recommended
  - 6 CPU cores recommended
  - 30 GB free disk space

---

# Configuration

Current configuration:

```
VM Box:          bento/ubuntu-22.04
Provider:        VirtualBox
Kubernetes:      RKE2
Rancher:         v2.9.3
Private network: 10.10.172.0/24
```

If your wireless adapter is different, modify:

```ruby
BRIDGE_IFACE = "wlp5s0"
```

Example:

```
eth0
enp3s0
wlan0
Wi-Fi
```

You can list available adapters with:

```bash
VBoxManage list bridgedifs
```

---

# Deploy

Start the entire environment:

```bash
vagrant up
```

Provisioning may take 10–20 minutes depending on your Internet connection.

---

# Verify Virtual Machines

```bash
vagrant status
```

Expected output:

```
rancher
nfsserver
node1
node2
node3
```

---

# Access Rancher

Open:

```
https://10.10.172.10 or https://rancher.local
```

Ignore the self-signed certificate warning.

Get the bootstrap password:

```bash
vagrant ssh rancher
docker logs rancher | grep Bootstrap
```

---

# Verify Kubernetes

Connect to node1:

```bash
vagrant ssh node1
```

Check cluster:

```bash
sudo /var/lib/rancher/rke2/bin/kubectl \
--kubeconfig /etc/rancher/rke2/rke2.yaml get nodes
```

Expected:

```
node1 Ready control-plane,etcd
node2 Ready
node3 Ready
```

---

# Verify NFS

Check exported directory:

```bash
showmount -e 10.10.172.20
```

Expected:

```
/nfs
```

Test mounting:

```bash
sudo mkdir /mnt/test

sudo mount -t nfs \
10.10.172.20:/nfs \
/mnt/test
```

---

# Import Cluster into Rancher

Do **NOT** create a new Kubernetes cluster.

Instead:

1. Open Rancher
2. Cluster Management
3. Import Existing Cluster
4. Copy generated registration command
5. Execute it on node1

After a few moments Rancher will detect:

- node1
- node2
- node3

---

# Deploy Applications

Example:

```
Nginx

Tomcat

Jenkins

Redis

PostgreSQL

MySQL

RabbitMQ

Prometheus

Grafana
```

Applications can be deployed:

- using Rancher UI
- using kubectl
- using Helm charts

---

# Persistent Storage

The environment already includes an NFS server.

Server:

```
10.10.172.20
```

Export:

```
/nfs
```

Recommended StorageClass:

- NFS CSI Driver
- NFS Subdir External Provisioner

This allows Kubernetes to automatically create PersistentVolumes.

---

# Useful Commands

Check nodes

```bash
kubectl get nodes
```

Check pods

```bash
kubectl get pods -A
```

Check services

```bash
kubectl get svc -A
```

Check deployments

```bash
kubectl get deployments -A
```

Describe pod

```bash
kubectl describe pod POD_NAME
```

Logs

```bash
kubectl logs POD_NAME
```

---

# Recommendations

## Use Namespaces

Example:

```
nginx

tomcat

monitoring

database

dev

production
```

---

## Use Persistent Volumes

Do not store important data inside containers.

Use:

- PVC
- PV
- StorageClass

---

## Deploy Applications with Helm

Examples:

```
NGINX Ingress

Prometheus

Grafana

Jenkins

Harbor

GitLab

ArgoCD
```

---

## Monitor the Cluster

Install:

- Metrics Server
- Prometheus
- Grafana

---

## Use Ingress

Instead of NodePort, expose applications through an Ingress Controller.

Recommended:

- ingress-nginx

---

# Destroy Environment

Stop VMs:

```bash
vagrant halt
```

Destroy everything:

```bash
vagrant destroy -f
```

---

# Troubleshooting

Check RKE2 server:

```bash
sudo systemctl status rke2-server
```

Check worker:

```bash
sudo systemctl status rke2-agent
```

View logs:

```bash
sudo journalctl -u rke2-server -f
```

or

```bash
sudo journalctl -u rke2-agent -f
```

Verify NFS:

```bash
showmount -e 10.10.172.20
```

Verify Rancher:

```bash
docker ps
```

---

# Project Architecture

```
                    Internet
                        │
                +----------------+
                |    Rancher     |
                | 10.10.172.10   |
                +--------+-------+
                         |
        -------------------------------------
        |                 |                 |
    +--------+       +--------+       +--------+
    | node1  |-------| node2  |-------| node3  |
    | Server |       | Worker |       | Worker |
    +--------+       +--------+       +--------+
           \             |             /
            \            |            /
             +-----------------------+
             |      NFS Server       |
             |    10.10.172.20       |
             |         /nfs          |
             +-----------------------+
```

---

# Notes

- Ubuntu 22.04
- RKE2 Kubernetes
- Rancher v2.9.3
- Docker installed on every VM
- Shared NFS storage
- Ready for deploying Kubernetes workloads
