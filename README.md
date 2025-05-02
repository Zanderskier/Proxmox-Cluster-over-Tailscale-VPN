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
┌──────────────────────┐     Tailscale VPN    ┌──────────────────────┐
│   Proxmox Node A     │◄────────────────────►│   Proxmox Node B     │
│  (e.g., 192.168.1.x) │                      │  (e.g., 192.168.2.x) │
│      100.x.x.x       │                      │       100.y.y.y      │
|    (Tailscale IP)    |                      |     (Tailscale IP)   |
│ ┌──────────────────┐ │                      │ ┌──────────────────┐ │
│ │  Ubuntu VM #1    │ │                      │ │  Ubuntu VM #2    │ │
│ │ (k8s controlplane│ │                      │ │ (k8s worker node)│ │
│ │  + Tailscale IP) │ │                      │ │  + Tailscale IP  │ │
│ └──────────────────┘ │                      │ └──────────────────┘ │
└──────────────────────┘                      └──────────────────────┘

                ⇅                                ⇅
            Kubernetes Cluster Communication (via Tailscale)

                    🛡️ All VMs in same Tailscale Tailnet
                    🔁 Internal Pod/Service traffic routes via Tailscale IPs
                    📡 No need for port-forwarding or public exposure
```
---

## 🛠️ Step-by-Step Setup

## Part 1 Proxmox Cluster

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

### 3. Edit Hosts file to Ensure Hostname Resolution for Cluster Communication
**File should be edited on all nodes in cluster**

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
### 5. Modify corosync.conf to Use Tailscale IPs
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

Restart the corosync service on the node, allowing it to read the updated configuration
```bash
systemctl restart corosync
```

### 6. Join the Cluster from Node 2
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

## Part 2 Create The Kubernetes Cluster
### 1. Create Virtual Machines (Ubuntu 22.04)

VM Name	Host Node	Role

kube-master	Proxmox Node 1	K8s control

kube-worker-1	Proxmox Node 2	Worker

Install Ubuntu Server 22.04 on all VMs.

Each Ubuntu server was given 2 CPU's 2GB Ram, 32GB storage

### 2. Connect All VMs to Tailscale

Add Tailscale's package signing key and repository:

```bash
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null

curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.tailscale-keyring.list | sudo tee /etc/apt/sources.list.d/tailscale.list

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

### 3. Configure hostnames (all nodes)
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
### 4. Disable swap space (all nodes)

Kubernetes requires swap to be disabled because it relies on precise memory management using Linux cgroups to enforce resource limits. When swap is enabled, the operating system can move memory pages to disk, which interferes with Kubernetes' ability to track and enforce memory usage, potentially leading to performance issues and inaccurate scheduling decisions. Additionally, swap can prevent proper out-of-memory (OOM) behavior, as processes might be swapped out instead of terminated. To ensure consistent and reliable operation.

Disable swap
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

### 5. Load Containerd modules (all nodes)

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

### 6. Configure Kubernetes IPv4 networking (all nodes)

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

### 7. Install Docker (on all nodes)
```bash
sudo apt update
```
```bash
sudo apt install docker.io -y
```
Enable the Docker daemon to autostart on system startup or upon a reboot
```bash
sudo systemctl enable docker
```
Check Docker status:
```bash
sudo systemctl status docker
```

The installation of Docker also comes with containerd.
Configure containerd to ensure it runs reliably in the Kubernetes cluster

```bash
sudo mkdir /etc/containerd
```
Next, create a default configuration file for containerd.
```bash
sudo sh -c "containerd config default > /etc/containerd/config.toml"
```

Update the SystemdCgroup directive by setting it to true as shown
```bash
sudo sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml
```
Restart the containerd service to apply the changes made.
```bash
sudo systemctl restart containerd.service
```
verify that the containerd service is running as expected
```bash
sudo systemctl status containerd.service
```

### 8. Install Kubernetes components (on all nodes)

The next step is to install Kubernetes
```bash
sudo apt-get install curl ca-certificates apt-transport-https  -y
```

Add the Kubernetes GPG signing key.
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
Thereafter, add the official Kubernetes repository to your system.
```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Once added, update the sources list for the system to recognize the newly added repository.
```bash
sudo apt update
```
kubeadm: This is a command-line utility for setting up Kubernetes clusters. It automates setting up a cluster, streamlining container deployments, and abstracting any complexities within the cluster. With kubeadm you can initialize the control-plane, configure networking, and join a remote node to a cluster.

kubelet: This is a component that actively runs on each node in a cluster to oversee container management. It takes instructions from the master node and ensures containers run as expected.

kubectl: This is a CLI tool for managing various cluster components including pods, nodes, and the cluster. You can use it to deploy applications and inspect, monitor, and view logs.

To install these salient Kubernetes components, run the following command:

```bash
sudo apt install kubelet kubeadm kubectl -y
```
### 9. Initialize Kubernetes cluster (on master node)

This configures the master node as the control plane. To initialize the cluster, run the command shown. The --pod-network-cidr indicates a unique pod network for the cluster, in this case, the 10.10.0.0 network with a CIDR of /16

Here we will ensure that Kubernetes API server advertises the Tailscale IP to other nodes. By default, it might pick your primary LAN interface by default unless specified.

```bash
sudo kubeadm init --pod-network-cidr=10.10.0.0/16 --apiserver-advertise-address=<node1 Tailscale IP>
```
Towards the end of the output, you will be notified that your cluster was initialized successfully. You will then be required to run the highlighted commands as a regular user. The command for joining the nodes to the cluster will be displayed at the tail end of the output.

Follow the direction of commands to run displayed in the terminal:
```bash
mkdir -p $HOME/.kube
```
```bash
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```
```bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 10: Install Calico network add-on plugin (on master node)

Calico provides a network security solution in a Kubernetes cluster. It secures communication between individual pods as well as pods and external services. By auto-assigning IP addresses to pods, it ensures smooth communication between them.

Deploy the Calico operator using the kubectl CLI tool.
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
```

Download Calico’s custom-resources file
```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml -O
```
Update the CIDR defined in the custom-resources file to match the pod's network
```bash
sed -i 's/cidr: 192\.168\.0\.0\/16/cidr: 10.10.0.0\/16/g' custom-resources.yaml
```

Then create resources defined in the YAML file
```bash
kubectl create -f custom-resources.yaml
```
At this point, the master node, which acts as the control plane, is the only node that is available in the cluster. To verify this, run the command:
```bash
kubectl get nodes
```
### 11: Add worker nodes to the cluster (on worker nodes)
With the master node configured, the remaining step is to add the worker nodes to the cluster.

Run the kubeadm join command below in each worker node
```bash
kubeadm join <node1 Tailscale IP>:6443 --token <generated token> \
        --discovery-token-ca-cert-hash sha256:<generated Hash>
```

Check the nodes in the cluster once again. This time around, you will see the worker nodes have joined the cluster. Run this command on the master node
```bash
kubectl get nodes
```

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

Tailscale IPs are part of a private virtual network (Tailnet): When you connect devices via Tailscale, they are assigned IP addresses in a private IP range (typically in the 100.x.x.x block). This is a part of the WireGuard-based VPN network and is used for communication within the Tailscale network.

Private IP Range: The 100.x.x.x addresses that Tailscale assigns to devices are part of the reserved private address space for Tailscale networks, not public IP addresses. While "100" in the first octet might seem like a public IP address at first glance, it’s specifically reserved for private networks like Tailnet. These addresses are not routable on the global internet, and external entities cannot access these IPs directly.

Zero-trust encrypted networking: Tailscale uses WireGuard to create a secure mesh network, ensuring that all communication between devices is encrypted and cannot be easily intercepted by third parties. The Tailscale IPs are strictly for communication within your private Tailscale network, meaning they cannot be accessed publicly or through traditional routing mechanisms on the internet.

---
## 💡 Lessons & Use Cases
Ideal for homelab enthusiasts, developers, and edge deployments

Validates Proxmox and Kubernetes integration over secure VPN

Enables simulation of production-grade setups in a home environment

Proxmox clustering requires very low-latency communication (<10ms) between nodes, especially for corosync

---
## Sources:

**[Aaron Smallberg, Proxmox vs. Kubernetes: Choosing the Right Platform](https://www.plural.sh/blog/kubernetes-proxmox-guide/)**

**[Brandon Lee, Setting Up Kubernetes on Proxmox: A Comprehensive Guide](https://theitbros.com/set-up-kubernetes-on-proxmox/)**

**[Sam Weaver, Kubernetes on Proxmox: A Practical Guide for DevOps](https://www.plural.sh/blog/kubernetes-on-proxmox-guide/)**

**[Shahalamol R, Kubernetes Cluster Deployment on Proxmox 8 | How To?](https://bobcares.com/blog/kubernetes-cluster-deployment-on-proxmox-8/)**

**[Tailscale Documentation, IP pool](https://tailscale.com/kb/1304/ip-pool)**

**[Winnie Ondara, How to Install Kubernetes on Ubuntu 24.04: Step-by-Step](https://www.cherryservers.com/blog/install-kubernetes-ubuntu)**


