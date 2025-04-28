# 🖧 Proxmox Cluster over Tailscale VPN

## Project Overview

This project demonstrates how to build a fully functional **Proxmox VE Cluster across two physically separated locations** using **Tailscale** as a secure networking layer, bypassing the requirement for a traditional local network.

Further, it shows how to deploy a **multi-node Kubernetes cluster** within this virtual environment, spanning both Proxmox nodes, proving that **self-contained, secure, and scalable infrastructure** is achievable even in a homelab setup.

---

## 📦 Tech Stack

- **Proxmox VE 8.x**
- **Tailscale (WireGuard-based VPN)**
- **Corosync** (Proxmox Cluster Communication)
- **Ubuntu Server 22.04 LTS**
- **Kubernetes (via kubeadm)**

---

## 🌐 Network Topology

```text
+-------------------------+           +--------------------------+
|    Proxmox Node 1       |           |     Proxmox Node 2       |
|  Location A (Home)      |           |   Location B (Remote)     |
|  - Tailscale IP: 100.x  |<--------->|   - Tailscale IP: 100.y   |
|  - VM1: kube-master     |           |   - VM3: kube-worker-2    |
|  - VM2: kube-worker-1   |           +--------------------------+
+-------------------------+
```
---

## 🛠️ Step-by-Step Setup

### 1. Install Proxmox VE on Two Physical Nodes

Install Proxmox VE on two separate machines in different physical locations.

- Node 1: Primary cluster initiator
- Node 2: Secondary node to join cluster

### 2. Set Up Tailscale

Install and authenticate Tailscale on **both Proxmox nodes**:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --authkey <your-auth-key> --hostname proxmox-nodeX
```
Verify connectivity:

```bash
tailscale status
Ensure both Proxmox nodes can reach each other via Tailscale IPs.
```
### 3. Create the Proxmox Cluster (on Node 1)
On the first node:

```bash
pvecm create my-cluster
```
### 4. Modify corosync.conf to Use Tailscale IPs
Edit the Corosync config (on Node 1):

```bash
nano /etc/pve/corosync.conf
```
Replace LAN IP with Tailscale IP:

```ini
logging {
  debug: off
  to_syslog: yes
}

nodelist {
  node {
    name: maximous
    nodeid: 1
    quorum_votes: 1
    ring0_addr: 100.x.x.x Replace with tailscale IP
  }
}

quorum {
  provider: corosync_votequorum
}

totem {
  cluster_name: cluster2
  config_version: 1
  interface {
    linknumber: 0
    bindnetaddr: 100.x.x.x Replace with Tailscale Network IP
    mcastport: 5405
    tty: 1
  }
  ip_version: ipv4-6
  link_mode: passive
  secauth: on
  version: 2
}

```
Save and sync.

### 5. Join the Cluster from Node 2
On Node 2:

```bash
pvecm add 100.x.x.x  # Tailscale IP of Node 1
```
Make sure ports 22, 5405 (Corosync), and 8006 (Web UI) are reachable via Tailscale.

### 6. Create Virtual Machines (Ubuntu 22.04)

VM Name	Host Node	Role
kube-master	Proxmox Node 1	K8s control
kube-worker-1	Proxmox Node 1	Worker
kube-worker-2	Proxmox Node 2	Worker
Install Ubuntu Server 22.04 on all VMs.

### 7. Connect All VMs to Tailscale
Install Tailscale on each VM:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --authkey <your-auth-key> --hostname kube-nodeX
```
Verify all nodes are on the Tailscale network.

### 8. Initialize Kubernetes Cluster
On kube-master:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
Set up kubectl for the user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Install CNI (e.g., Flannel):

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
Join worker nodes using the kubeadm join command output from kubeadm init.

---
## ✅ Outcome
You now have:

A Proxmox cluster spread across physical locations

Cluster communication routed securely through Tailscale

A Kubernetes cluster with nodes hosted on separate Proxmox servers

A robust, remotely manageable infrastructure setup

---
## 🔒 Security & Reliability Notes
Tailscale ensures zero-trust encrypted networking between nodes

No need for port forwarding or public IPs

Adds resilience and scalability to homelab projects

---
## 💡 Lessons & Use Cases
Ideal for homelab enthusiasts, developers, and edge deployments

Validates Proxmox and Kubernetes integration over secure VPN

Enables simulation of production-grade setups in a home environment

