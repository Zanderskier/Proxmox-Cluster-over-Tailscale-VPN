# 🖧 Proxmox & Kubernetes Cluster over Tailscale VPN

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

## Part 1 Procmox Cluster

### 1. Install Proxmox VE on Two Physical Nodes

Install Proxmox VE on two separate machines in different physical locations.

- Node 1: Primary cluster initiator
- Node 2: Secondary node to join cluster

### 2. Set Up Tailscale
**Repeat the following steps on both nodes**

Add Tailscale's package signing key and repository:

```bash
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.noarmor.gpg | tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null

curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.tailscale-keyring.list | tee /etc/apt/sources.list.d/tailscale.list

```
Install Tailscale:

```bash
sudo apt-get update
sudo apt-get install tailscale
```
Connect machines to Tailscale network and authenticate via browser:
```bash
sudo tailscale up
```
Verify connection:
```bash
tailscale status
```
Ensure both Proxmox nodes can reach each other via Tailscale IPs.

### 3. Edit Hosts file to Ensuring Hostname Resolution for Cluster Communication
**File should be editted on all nodes in cluster**

```bash
nano /etc/hosts
```
file should look similar to the following:
```bash

127.0.0.1 localhost.localdomain localhost
x.x.x.x <node 1 hostname>.<FQD> <alt hostname for LAN>
(LAN IP Address)

100.x.x.x <Tailscale Node 1 Hostname>
100.x.x.x <Tailscale Node 2 Hostname>

# The following lines are desirable for IPv6 capable hosts

::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
```

### 4. Create the Proxmox Cluster (on Node 1)
On the first node:

```bash
pvecm create <my-cluster-name>
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
    name: <node 1 hostname>
    nodeid: 1
    quorum_votes: 1
    ring0_addr: 100.x.x.x Replace with tailscale IP
  }
}

quorum {
  provider: corosync_votequorum
}

totem {
  cluster_name: <cluster-name>
  config_version: 1
  interface {
    linknumber: 0
    bindnetaddr: 100.x.x.x Replace with Tailscale Network IP (typically ends in 100.x.x.0)
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
**If the Above step fails, try the following**
```bash
pvecm add <node 1 hostname>
```
Make sure ports 22, 5405 (Corosync), and 8006 (Web UI) are reachable via Tailscale.

### ✅ A Proxmox Cluster should be established now

---

## Create The Kubernetes Cluster
### 6. Create Virtual Machines (Ubuntu 22.04)

VM Name	Host Node	Role
kube-master	Proxmox Node 1	K8s control
kube-worker-1	Proxmox Node 1	Worker
kube-worker-2	Proxmox Node 2	Worker
Install Ubuntu Server 22.04 on all VMs.

Each Ubuntu server in was given 2 CPU's 2GB Ram, 32GB storage

### 7. Connect All VMs to Tailscale

Add Tailscale's package signing key and repository:

```bash
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.noarmor.gpg | tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null

curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.tailscale-keyring.list | tee /etc/apt/sources.list.d/tailscale.list

```
Install Tailscale:

```bash
sudo apt-get update
sudo apt-get install tailscale
```
Connect machines to Tailscale network and authenticate via browser:
```bash
sudo tailscale up
```
Verify connection:
```bash
tailscale status
```

---
## Initialize Kubernetes Cluster

The following steps should be completed on all nodes in cluster:

### 1. Configure hostnames (all nodes)
```bash
sudo  nano  /etc/hosts
```
The file should look similar to this:
(node1 file example)
```bash
127.0.0.1 localhost
127.0.1.1 <node1 hostname>

<tailscale IP> <node1 hostname>
<tailscale IP> <node2 hostname>

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```
### 2. Disable swap space (all nodes)

``` bash
sudo swapoff -a
```

This will only temporarily disable swap and it will turn on on reboot, to fix this edit the **fstab** file and comment out the **swap** line

```bash
sudo nano /etc/fstab
```
![swap](https://github.com/user-attachments/assets/08592719-dbe2-40b3-b81c-bf9d9ba7ce2f)

check that it is disabled by running this command. There will be no output if swap is off.
```bash
swapon --show
```

### 3. Load Containerd modules (all nodes)

```bash
sudo modprobe overlay
```
```bash
sudo modprobe br_netfilter
```

Create a configuration file as shown and specify the modules to load them permanently.

```bash
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
```

### 4. Configure Kubernetes IPv4 networking (all nodes)

```bash
sudo nano   /etc/sysctl.d/k8s.conf
```

Add the following lines:
```bash
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward   = 1
```

Then apply the settings by running the following command:
```bash
sudo sysctl --system
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

