# Installing K3s on Ubuntu Server — My First Kubernetes Cluster

> **Author:** perseeland  
> **Date:** May 28, 2026  
> **Series:** Kubernetes Homelab Journey — Part 1

---

## Introduction

This is the first entry in my Kubernetes homelab series. My goal is to learn Kubernetes from scratch, document everything I do, and build a portfolio that demonstrates real hands-on experience — not just theory.

For this setup I'm using **K3s**, a lightweight, production-ready Kubernetes distribution made by Rancher. Instead of installing full Kubernetes (which is complex and resource-heavy), K3s gives you the full Kubernetes experience in a single binary that installs in minutes. It's perfect for learning, homelabs, and edge deployments.

**My environment:**
- OS: Ubuntu 26.04 LTS (running as a VM)
- RAM: 3.3GB
- Role: Single-node cluster (control-plane)

---

## What is Kubernetes?

Kubernetes (also written as **K8s**) is an open-source system for automating the deployment, scaling, and management of containerised applications. Instead of manually running Docker containers, Kubernetes lets you describe what you want running, and it makes sure it stays that way — even if something crashes or a server goes down.

### Key concepts you should know before starting:

| Term | What it means |
|------|--------------|
| **Pod** | The smallest deployable unit in Kubernetes. Usually wraps one container. |
| **Node** | A machine (physical or virtual) that runs pods. |
| **Cluster** | A group of nodes managed by Kubernetes. |
| **Control Plane** | The brain of the cluster — schedules workloads and manages state. |
| **kubectl** | The command-line tool used to interact with your cluster. |
| **Namespace** | A way to logically separate workloads inside one cluster. |
| **Deployment** | Tells Kubernetes to run X copies of a pod and keep them running. |
| **Service** | Exposes a pod or deployment to the network via a stable IP/DNS name. |

---

## What is K3s?

K3s is a fully compliant Kubernetes distribution with a much smaller footprint. It:

- Ships as a **single binary** (~70MB)
- Uses **containerd** instead of Docker as the container runtime
- Bundles everything you need: CoreDNS, Traefik ingress, metrics-server, local storage provisioner
- Installs with a **single curl command**
- Is maintained by Rancher (now part of SUSE)

Think of it as Kubernetes with the batteries already included and the unnecessary parts removed.

---

## Prerequisites

### System Requirements

| Resource | Minimum | My Setup |
|----------|---------|----------|
| CPU | 1 core | 2 cores |
| RAM | 512MB | 3.3GB |
| Disk | 2GB | 20GB |
| OS | Ubuntu 20.04+ | Ubuntu 26.04 LTS |

### Skills you should have before starting:
- Basic Linux command line (navigating directories, editing files, running commands)
- Understanding of what containers are (Docker knowledge helps)
- Basic networking concepts (IP addresses, ports, DNS)
- Familiarity with `systemctl` for managing services

---

## Step 1 — Update the System

Before installing anything, always update your package lists and upgrade existing packages. This ensures you have the latest security patches and dependencies.

```bash
sudo apt update && sudo apt upgrade -y
```

- `apt update` — refreshes the list of available packages from repositories
- `apt upgrade -y` — upgrades all installed packages, `-y` auto-confirms

---

## Step 2 — Disable Swap

### What is swap?

Swap is disk space that the OS uses as overflow RAM. When physical RAM fills up, the kernel moves less-used memory pages to the swap partition on disk. It's essentially slow, fake RAM.

### Why disable it for Kubernetes?

Kubernetes was designed around **predictable, guaranteed memory**. Swap creates three problems:

1. **Performance unpredictability** — a pod might suddenly slow down because its memory was swapped to disk without Kubernetes knowing
2. **Resource guarantees break** — when you assign a pod 512MB RAM, Kubernetes can't truly guarantee that if swap is involved
3. **Scheduling decisions go wrong** — the scheduler places pods based on available RAM; swap makes those numbers misleading

Kubernetes wants full control of memory management, and swap interferes with that.

```bash
# Disable swap immediately (takes effect now)
sudo swapoff -a

# Disable swap permanently (survives reboots)
# This comments out the swap line in /etc/fstab
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify swap is off
free -h
```

**Expected output:**
```
               total        used        free      shared  buff/cache   available
Mem:           3.3Gi       591Mi       2.0Gi       1.5Mi       962Mi       2.7Gi
Swap:             0B          0B          0B
```

The `Swap:` row showing `0B` across all columns confirms swap is completely disabled. ✅

---

## Step 3 — Set the Hostname

### What is a hostname?

A hostname is the name of your machine on a network — like a label that identifies it. In Kubernetes, hostnames are used to identify nodes in the cluster. When you run `kubectl get nodes`, you'll see your hostname listed there.

### Why `k3s-master`?

Naming the machine `k3s-master` makes it immediately clear what this node's role is. When you eventually add worker nodes (e.g., `k3s-worker-01`), your cluster topology is obvious at a glance. This is good practice and shows intentional infrastructure design.

```bash
# Set the hostname
sudo hostnamectl set-hostname k3s-master

# Verify the change
hostnamectl
```

After this your terminal prompt will change from `username@linux` to `username@k3s-master` when you open a new session.

---

## Step 4 — Install K3s

This is the main event. K3s installs with a single command:

```bash
curl -sfL https://get.k3s.io | sh -
```

**What this command does:**
- `curl -sfL` — downloads the install script silently (`-s`), fails on errors (`-f`), follows redirects (`-L`)
- `https://get.k3s.io` — the official K3s install script from Rancher
- `| sh -` — pipes the downloaded script directly into the shell to execute it

**What gets installed:**
- K3s binary at `/usr/local/bin/k3s`
- `kubectl` symlink (the Kubernetes CLI)
- `crictl` symlink (container runtime CLI)
- `ctr` symlink (containerd CLI)
- A `systemd` service that starts K3s automatically on boot
- An uninstall script at `/usr/local/bin/k3s-uninstall.sh`

**What K3s automatically deploys inside the cluster:**
- **CoreDNS** — internal DNS for the cluster
- **Traefik** — ingress controller for HTTP/HTTPS routing
- **metrics-server** — collects resource usage stats
- **local-path-provisioner** — handles persistent storage
- **svclb (ServiceLB)** — built-in load balancer

---

## Step 5 — Verify K3s is Running

```bash
sudo systemctl status k3s
```

**What to look for:**
```
Active: active (running)
```

This confirms K3s is running as a `systemd` service and will automatically start on reboot.

---

## Step 6 — Check Your Node

```bash
sudo kubectl get nodes
```

**Expected output:**
```
NAME         STATUS   ROLES           AGE   VERSION
k3s-master   Ready    control-plane   70s   v1.35.5+k3s1
```

- **NAME** — your hostname, confirming the node is registered correctly
- **STATUS: Ready** — the node is healthy and can accept workloads
- **ROLES: control-plane** — this node runs the Kubernetes control plane
- **VERSION** — the K3s/Kubernetes version running

---

## Step 7 — Fix kubectl Permissions

By default, the K3s config file is owned by root, so every `kubectl` command requires `sudo`. This is inconvenient. Let's fix it:

```bash
# Create the .kube directory in your home folder
mkdir -p ~/.kube

# Copy the K3s config to your user's kube config location
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

# Change ownership so your user can read it
sudo chown $USER:$USER ~/.kube/config

# Set the KUBECONFIG environment variable for this session
export KUBECONFIG=~/.kube/config

# Make it permanent across all future sessions
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
```

**Test it without sudo:**
```bash
kubectl get nodes
```

If you see your node without needing `sudo`, you're all set. ✅

---

## Verifying the Full Installation

Let's look at all pods running inside the cluster:

```bash
kubectl get pods -A
```

**My output:**
```
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
kube-system   coredns-8db54c48d-7kdlk                   1/1     Running     0          2m43s
kube-system   helm-install-traefik-5k2cz                0/1     Completed   2          2m41s
kube-system   helm-install-traefik-crd-2gpc4            0/1     Completed   0          2m41s
kube-system   local-path-provisioner-5d9d9885bc-ms548   1/1     Running     0          2m43s
kube-system   metrics-server-786d997795-vkf99           1/1     Running     0          2m43s
kube-system   svclb-traefik-7cb7fbe6-xvdqt              2/2     Running     0          2m13s
kube-system   traefik-9bcdbbd9-4tgt6                    1/1     Running     0          2m13s
```

### What each pod does:

| Pod | Status | Purpose |
|-----|--------|---------|
| `coredns` | Running | Internal DNS — lets pods find each other by name instead of IP address |
| `helm-install-traefik` | Completed | One-time job that installed Traefik using Helm. Completed = done successfully |
| `helm-install-traefik-crd` | Completed | Installed Traefik's Custom Resource Definitions. Completed = done successfully |
| `local-path-provisioner` | Running | Automatically provisions disk storage when pods request it |
| `metrics-server` | Running | Collects CPU and memory usage from nodes and pods. Powers `kubectl top` |
| `svclb-traefik` | Running | ServiceLB — forwards external traffic into the cluster to reach Traefik |
| `traefik` | Running | Ingress controller — routes HTTP/HTTPS requests from outside to your services |

All statuses are healthy — `Running` means actively working, `Completed` means the job finished successfully. ✅

---

## Useful Commands Reference

```bash
# Cluster info
kubectl get nodes                    # list all nodes
kubectl get pods -A                  # list all pods in all namespaces
kubectl get pods -n kube-system      # list pods in a specific namespace
kubectl get all                      # list everything in current namespace

# Inspecting resources
kubectl describe node k3s-master     # detailed info about a node
kubectl describe pod <pod-name>      # detailed info + events for a pod
kubectl logs <pod-name>              # view logs from a pod

# Service management
sudo systemctl status k3s            # check K3s service status
sudo systemctl restart k3s           # restart K3s
sudo journalctl -u k3s -f            # follow K3s logs in real time
```

---

## Key Takeaways

- K3s is full Kubernetes in a lightweight package — perfect for learning and homelabs
- Disabling swap is best practice for Kubernetes — it needs predictable memory management
- Naming your nodes clearly (e.g., `k3s-master`) is good infrastructure hygiene
- K3s automatically deploys CoreDNS, Traefik, metrics-server, and storage provisioner — you don't need to set these up manually
- Always fix kubectl permissions so you don't need sudo for every command

---

## What's Next

- **Part 2** — Deploying my first application (nginx) with a Deployment and Service
- **Part 3** — Writing YAML manifests and understanding the Kubernetes object model
- **Part 4** — Setting up Ingress with Traefik to route traffic by domain name
- **Part 5** — Installing Helm and deploying real-world applications

---

*This blog is part of my Kubernetes homelab portfolio. All commands were run on a real Ubuntu VM — nothing is theoretical.*
