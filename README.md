# Homelab

A hands-on homelab built for learning **Linux systems administration, networking, containers, Kubernetes, automation, monitoring, and infrastructure troubleshooting**.

This project started as a way to host a few services at home and gradually turned into my own small infrastructure environment. I use it to learn by actually deploying services, breaking things, troubleshooting them, rebuilding them, and documenting what I learned along the way.

My background is in the electrical trade, so I tend to approach infrastructure the same way I approach electrical troubleshooting: understand the system, identify which subsystem owns the problem, gather evidence, and work backward from there.

---

## 🏗️ Current Architecture

The lab is a mix of **physical hardware and virtual machines**, with dedicated systems handling different infrastructure roles.

```text
                          Home Network
                               │
                     ┌─────────┴─────────┐
                     │                   │
                 DNS / Pi-hole       Homelab Services
                     │                   │
               ┌─────▼─────┐       ┌────▼─────┐
               │   MINIX   │       │  marine  │
               │ Pi-hole / │       │  Linux   │
               │    DNS    │       │  Server  │
               └───────────┘       └────┬─────┘
                                        │
                              Kubernetes Control Plane
                                        │
                     ┌──────────────────┼──────────────────┐
                     │                  │                  │
              ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
              │ k8s-worker1 │   │ k8s-worker2 │   │ k8s-worker3 │
              │  Hyper-V VM │   │  Hyper-V VM │   │  Physical   │
              └─────────────┘   └─────────────┘   └─────────────┘
```

The environment is intentionally heterogeneous. Some nodes are virtualized, some are physical, and hardware ranges from modern systems to older machines repurposed for infrastructure work.

Part of the goal is learning how to manage systems that aren't perfectly uniform — because real infrastructure rarely is.

---

## ☸️ Kubernetes

The lab currently includes a multi-node Kubernetes cluster built manually with **kubeadm** rather than using a prepackaged Kubernetes distribution.

### Cluster

| Node | Type | Role |
|---|---|---|
| `marine` | Physical Linux server | Control plane |
| `k8s-worker1` | Hyper-V VM | Worker |
| `k8s-worker2` | Hyper-V VM | Worker |
| `k8s-worker3` | Physical laptop | Worker |

The cluster uses:

- Kubernetes
- containerd
- Flannel CNI
- CoreDNS
- kube-proxy
- kubeadm / kubelet / kubectl

I'm using the cluster as a platform for learning workloads, manifests, networking, persistent storage, service discovery, monitoring, scheduling, and eventually more automation.

Rather than treating Kubernetes as a black box, the goal is to understand what each component is doing and what actually happens when something fails.

---

## 🐳 Containers & Services

Before building the Kubernetes cluster, much of the lab ran directly through Docker.

Services I've deployed or worked with include:

- **Plex** — media server
- **Home Assistant** — home automation
- **Pi-hole** — network DNS and ad blocking
- **Traefik** — reverse proxy and service routing
- **Homepage** — homelab service dashboard
- **Tailscale** — secure remote access
- **Docker / Docker Compose**
- **Kubernetes workloads**

Some services remain Docker-based while others are being deployed or migrated into Kubernetes.

That gives me a useful environment for comparing the two approaches instead of learning Kubernetes in isolation.

---

## 🌐 Networking & DNS

Networking has become a major part of this project.

The lab includes work with:

- Static addressing
- Local DNS
- DNS troubleshooting
- Pi-hole
- Reverse proxies
- Internal service hostnames
- Tailscale
- Remote access
- Linux network configuration
- Container networking
- Kubernetes networking
- Flannel CNI
- Service discovery

One of my main goals is understanding the entire path a request takes:

```text
Client
  ↓
DNS
  ↓
Network
  ↓
Reverse Proxy / Kubernetes Service
  ↓
Pod or Container
  ↓
Application
```

When something doesn't work, I want to be able to identify **which layer failed and why** rather than just trying commands until something fixes it.

---

## 🖥️ Linux Systems Administration

Most of the infrastructure runs Linux, primarily Ubuntu Server.

Areas I'm actively working with include:

- systemd
- journalctl
- SSH
- users and groups
- permissions and ownership
- package management
- networking
- filesystems
- Bash
- services and processes
- Docker
- containerd
- virtualization
- Kubernetes

I also write small Bash utilities to automate repetitive administration tasks.

Those scripts live in a separate repository:

**[`bash-scripts`](../bash-scripts)**

---

## 📁 Repository Structure

```text
Homelab/
├── docker/
│   └── Docker service documentation and configuration
│
├── kubernetes/
│   ├── manifests
│   ├── workloads
│   └── Kubernetes documentation
│
├── pihole/
│   └── DNS / Pi-hole documentation
│
├── .gitignore
└── README.md
```

Individual directories contain more detailed documentation for specific services and projects.

The root README is intended to provide the big-picture view of how everything fits together.

---

## 🔧 How I Use This Lab

This isn't intended to be a production environment or a collection of copy-pasted tutorials.

I use the lab to practice the same process repeatedly:

1. Learn how a subsystem works.
2. Deploy something with it.
3. Verify the expected behavior.
4. Break something — intentionally or otherwise.
5. Gather evidence.
6. Determine which subsystem owns the problem.
7. Fix it.
8. Document what happened.

A lot of the most useful learning in this repository comes from troubleshooting things that **didn't** work the first time.

---

## 🧠 What I'm Learning

My current focus is building toward professional Linux systems administration and infrastructure work.

### Current

- Linux administration
- Bash scripting
- Networking
- Docker
- Kubernetes
- DNS
- Reverse proxies
- Git

### Next

- Prometheus
- Grafana
- More Kubernetes storage
- Kubernetes ingress
- Secrets/configuration management
- Python for system administration
- Ansible
- Infrastructure automation
- GitOps
- Cluster monitoring and alerting
- Backup and recovery strategies

Longer term, I'm especially interested in **Linux infrastructure, Kubernetes, distributed systems, automation, and HPC environments**.

---

## 🔐 Security

This repository is public, so sensitive information is intentionally excluded.

That includes things such as:

- Passwords
- API keys
- Tokens
- SSH private keys
- Private configuration
- Sensitive internal network information

Public configuration and documentation may also be sanitized where necessary.

---

## 🚧 Project Status

This homelab is actively evolving.

Hardware gets repurposed. Services move between hosts. Docker workloads become Kubernetes workloads. Configurations get rewritten as I learn better ways to do things.

That's part of the point.

The repository is both documentation of the environment **and a record of my progression as I learn to build and administer increasingly complex Linux infrastructure.**

---

## 🛠️ Technologies

`Linux` · `Ubuntu Server` · `Bash` · `Git` · `Docker` · `containerd` · `Kubernetes` · `kubeadm` · `Flannel` · `Traefik` · `Pi-hole` · `Tailscale` · `Home Assistant` · `Plex` · `Hyper-V` · `DNS` · `SSH`

---

*Built, broken, troubleshot, rebuilt, and documented in my homelab.*
