# Kubernetes concepts — clusters, nodes, pods, namespaces, and services

> **Author:** perseeland  
> **Date:** May 28, 2026  
> **Series:** Kubernetes Homelab Journey — Part 2

---

## Introduction

Before deploying any application on Kubernetes, you need to understand what you're actually working with. This post covers the core building blocks of Kubernetes — what they are, how they relate to each other, and what's already running inside your K3s cluster.

Think of this as the mental model you need before touching anything in Part 3.

---

## The big picture — everything is nested

Kubernetes organises everything in layers, from biggest to smallest:

```
Cluster
  └── Node(s)
        └── Namespace(s)
              └── Pod(s)
                    └── Container(s)
```

Each layer contains the one below it. Let's go through each one.

---

## The cluster

The **cluster** is the entire Kubernetes environment. Everything Kubernetes manages lives inside it. Think of it like a factory building — the building itself doesn't do the work, but it contains everything that does.

Your cluster has one address: the **API server**. Every interaction — whether from `kubectl`, an app, or an internal component — goes through the API server.

When you run:
```bash
kubectl get nodes
```
You are talking to the cluster's API server.

**Your cluster:** K3s v1.35.5, single-node, running on Ubuntu 26.04 LTS.

---

## The node

A **node** is a physical or virtual machine inside the cluster. It's where the actual work happens — where containers run. Think of it like a floor inside the factory, with workers and machinery on it.

In a production cluster you'd have many nodes — if one goes down, the others keep running and Kubernetes reschedules the affected pods automatically. In your setup, you have one node that does everything.

A node has two responsibilities:
- Running the **control plane** — the brain of the cluster
- Running **pods** — the actual workloads

```bash
kubectl get nodes
```
```
NAME         STATUS   ROLES           AGE   VERSION
k3s-master   Ready    control-plane   ...   v1.35.5+k3s1
```

**Your node:** `k3s-master` — the hostname we set during installation.

### What lives on a node

Every node runs two core processes:

| Process | What it does |
|---------|-------------|
| `kubelet` | The node agent — talks to the API server, starts/stops pods |
| `kube-proxy` | Handles network routing so pods can find each other |

K3s bundles these automatically — you never had to install them separately.

---

## The pod

A **pod** is the smallest thing Kubernetes manages. It wraps one (or sometimes more) containers and gives them a shared IP address, storage, and network identity. Think of it like a workstation on the factory floor — one worker, one job.

Key things to understand about pods:

- **Pods are disposable.** They get created, killed, and replaced constantly. You never edit a running pod — you update the thing that creates it and Kubernetes replaces it.
- **Every pod gets its own IP address** — but that IP disappears when the pod dies. This is why Services exist (more on that below).
- **Pods live inside namespaces** — a logical grouping system inside the cluster.

```bash
kubectl get pods -A
```

The `-A` flag means "all namespaces" — you'll see pods from `kube-system` and any other namespaces you've created.

---

## Namespaces

A **namespace** is a logical divider inside a cluster. Like departments in a company — HR, Engineering, Finance all share the same building but are separated from each other.

Your cluster has two namespaces that matter right now:

| Namespace | Purpose |
|-----------|---------|
| `kube-system` | All K3s system pods — the infrastructure that runs the cluster |
| `default` | Where your apps go when you don't specify a namespace |

```bash
kubectl get namespaces
```

Namespaces let you:
- Separate environments (dev, staging, production) on one cluster
- Apply different permissions to different teams
- Set resource limits per namespace (e.g. staging gets 2GB RAM max)

---

## What's already running in your cluster

When K3s installed, it automatically deployed several system pods in the `kube-system` namespace. Here's what every single one does:

```bash
kubectl get pods -n kube-system
```

### CoreDNS

```
coredns-8db54c48d-7kdlk   1/1   Running
```

**What it is:** The internal DNS server for your cluster.

**What it does:** Every pod and service inside the cluster gets a DNS name. When your frontend pod wants to talk to your backend pod, it doesn't use an IP — it uses a name like `my-backend-service.default.svc.cluster.local`. CoreDNS resolves those names to the correct IP addresses automatically.

Without CoreDNS, pods could only find each other by IP address — which changes every time a pod restarts.

---

### Traefik

```
traefik-9bcdbbd9-4tgt6   1/1   Running
```

**What it is:** The ingress controller — the front door of your cluster.

**What it does:** Traefik sits at the edge of your cluster and routes incoming HTTP/HTTPS traffic to the right service based on rules you define. It's what allows you to run multiple apps on the same cluster and route them by domain name:

```
myapp.com   → Traefik → Service A → App A pods
myapi.com   → Traefik → Service B → App B pods
```

K3s ships Traefik pre-installed. In a standard Kubernetes setup, you'd have to install an ingress controller manually.

---

### svclb-traefik (ServiceLB)

```
svclb-traefik-7cb7fbe6-xvdqt   2/2   Running
```

**What it is:** K3s's built-in load balancer.

**What it does:** In cloud Kubernetes (AWS, GCP, Azure), when you create a `LoadBalancer` type Service, the cloud provider automatically provisions an external load balancer for you. On a bare-metal or VM setup like yours, there's no cloud provider to do that. `svclb` fills that gap — it's K3s's own solution that forwards external traffic from your node's IP into the cluster.

This is what makes your cluster reachable from the outside world without a cloud provider.

---

### metrics-server

```
metrics-server-786d997795-vkf99   1/1   Running
```

**What it is:** The cluster's resource monitor.

**What it does:** Collects CPU and memory usage from every node and pod in real time. This powers the `kubectl top` command:

```bash
kubectl top nodes
kubectl top pods
```

It also feeds data to Kubernetes' autoscaler — if you ever set up Horizontal Pod Autoscaling (automatically adding more pods when CPU is high), it relies on metrics-server to know when to scale.

---

### local-path-provisioner

```
local-path-provisioner-5d9d9885bc-ms548   1/1   Running
```

**What it is:** The storage manager.

**What it does:** When a pod needs to save data to disk (a database, file uploads, logs), it creates a `PersistentVolumeClaim` — a request for storage. The local-path-provisioner automatically creates a folder on your node's disk and hands it to the pod.

Without this, pods could only store data in memory — everything would be lost when the pod restarts.

---

### helm-install-traefik (Completed)

```
helm-install-traefik-5k2cz   0/1   Completed
```

**What it is:** A one-time install job.

**What it does:** When K3s first started, it ran this job to install Traefik using Helm (Kubernetes' package manager). `Completed` is the correct status — the job finished successfully and doesn't need to run again. You can ignore it.

---

## The control plane — inside the node

The control plane is the brain of the cluster. It's what makes Kubernetes "smart" — it watches what's running and constantly works to match reality to what you've asked for.

It has three key components, all running inside your `k3s-master` node:

### API server
Every request — from `kubectl`, from other cluster components, from apps — goes through the API server. It's the single entry point to the cluster. When you run `kubectl apply -f deployment.yaml`, you're sending that file to the API server.

### Scheduler
When a new pod needs to run, the scheduler decides which node to place it on. It looks at available resources (CPU, RAM), pod requirements, and placement rules to make the best decision. On a single-node cluster it always picks your one node — but it becomes critical when you have many nodes.

### Controller manager
Runs a set of controllers — loops that constantly watch the cluster state. The most important one is the **ReplicaSet controller**: if you say "I want 3 copies of this pod" and one dies, the controller notices and creates a replacement immediately.

---

## Services — the networking glue

Pods have unstable IP addresses — they change every time a pod is replaced. A **Service** solves this by giving you a stable, permanent endpoint that always routes to the right pods, no matter how many times they've been replaced.

A Service uses **labels and selectors** to find its pods. You tag pods with a label (e.g. `app: my-backend`), and the Service automatically routes traffic to any pod with that label.

```
You → Service (stable IP: 10.96.0.1) → Pod A  ┐
                                        Pod B  ┘  both have label app: my-backend
```

### The three Service types

#### ClusterIP (default)
Internal only. The Service gets a stable IP that only other pods inside the cluster can reach. Use this for pod-to-pod communication — like a frontend talking to a backend, or an app talking to its database.

```
Frontend pod → ClusterIP Service → Database pod
(not reachable from outside the cluster)
```

#### NodePort
Opens a port on your actual node (between 30000–32767) so you can reach the Service from outside by hitting `http://your-server-ip:30080`. Good for learning and testing.

```
Your browser → node-ip:30080 → NodePort Service → Pod
```

#### LoadBalancer
The production way to expose services externally. In K3s, `svclb` handles this automatically — it binds to port 80/443 on your node and forwards traffic to the right service.

```
Your browser → port 80 → svclb → LoadBalancer Service → Pod
```

---

## How it all connects — full traffic flow

When you eventually deploy an app and visit it in your browser, here is the exact path the traffic takes:

```
Browser
  ↓ hits your server IP on port 80
svclb (ServiceLB)
  ↓ forwards to Traefik
Traefik (ingress controller)
  ↓ matches routing rule (e.g. domain name or path)
Service
  ↓ load balances across matching pods
Pod A / Pod B / Pod C
  ↓ your app handles the request
Response back to browser
```

Every component you saw in `kubectl get pods -A` plays a role in this chain.

---

## Key concepts summary

| Concept | What it is | Analogy |
|---------|-----------|---------|
| Cluster | The entire Kubernetes environment | The factory building |
| Node | A machine inside the cluster | A floor in the factory |
| Pod | The smallest deployable unit — wraps a container | A workstation on the floor |
| Namespace | Logical separation inside a cluster | Departments in the factory |
| Service | Stable network endpoint for a set of pods | The reception desk — always in the same place |
| CoreDNS | Internal DNS — pods find each other by name | The company phone directory |
| Traefik | Routes external HTTP traffic to services | The front door + receptionist |
| svclb | K3s load balancer — receives external traffic | The building's main entrance |
| metrics-server | Collects CPU/RAM stats | The factory floor monitor |
| local-path-provisioner | Provisions disk storage for pods | The warehouse manager |
| Control plane | The brain — API server, scheduler, controller manager | Factory management |

---

## Useful commands

```bash
# Cluster overview
kubectl get nodes
kubectl get namespaces
kubectl cluster-info

# See everything running
kubectl get pods -A
kubectl get pods -n kube-system

# Inspect a specific pod
kubectl describe pod <pod-name> -n kube-system

# See resource usage
kubectl top nodes
kubectl top pods -A

# See all services
kubectl get services -A
```

---

## What's next

Now that you understand the building blocks, Part 3 will put them all into practice — deploying a real application, creating a Deployment and a Service, and watching traffic flow all the way from your browser to your pod.

---

*This blog is part of my Kubernetes homelab portfolio. Running on a real K3s cluster on Ubuntu 26.04 LTS.*
