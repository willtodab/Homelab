# Kubernetes Homelab Bootstrap & Troubleshooting Runbook

This document records the initial build of the homelab Kubernetes cluster and, more importantly, explains **why each layer exists and how to troubleshoot it**.

The goal is not just to have a working cluster. The goal is to understand which subsystem owns a failure when something breaks.

> **Public-copy note:** hostnames, usernames, MAC addresses, and node/LAN IP addresses in this document are sanitized examples. `192.0.2.0/24` is an RFC 5737 documentation network and is not the live homelab subnet. Never commit kubeconfigs, private keys, certificates, bootstrap tokens, or other live credentials.

> Core troubleshooting question: **Which layer is failing, and what evidence proves it?**

---

## 1. Cluster Goal

Initial cluster design:

```text
                               HOME LAN
                             192.0.2.0/24
                                   |
                +------------------+------------------+
                |                                     |
        control-plane1 / primary Linux server       Windows 11 host
        Physical Ubuntu                            Hyper-V
        CONTROL PLANE                                  |
        192.0.2.10                                  +-- k8s-worker1
                                                   |   Ubuntu Server VM
                                                   |   192.0.2.21
                                                   |
                                                   +-- k8s-worker2
                                                       Ubuntu Server VM
                                                       192.0.2.22

        Future:
        separate physical Linux machine
        +-- k8s-worker3
```

### Intended workload split

The cluster is initially intended for:

- Prometheus
- Grafana
- Alertmanager
- Loki
- Argo CD / GitOps
- automation jobs
- CronJobs
- test applications
- Kubernetes troubleshooting labs

Existing infrastructure such as DNS/DHCP, application, storage, reverse-proxy, and remote-access services remains outside Kubernetes during the learning phase.

### Redundancy model

The initial cluster has:

- 1 physical control-plane node
- 2 virtual worker nodes on one Windows/Hyper-V host
- planned third worker on a separate physical machine

This gives useful **worker/workload redundancy**, but it is **not yet control-plane HA**.

If `control-plane1` fails, existing containers on workers may continue running, but the API server, scheduler, controllers, and etcd are unavailable until the control plane returns.

A future HA design should use an odd number of control-plane nodes, normally three, behind a stable API endpoint/load balancer or virtual IP.

---

# 2. Layer 0: Hardware Virtualization

The Windows host uses Hyper-V to run the two Ubuntu worker VMs.

Required firmware features were enabled in UEFI:

- CPU virtualization support (Intel VT-x/VMX or AMD-V/SVM)
- I/O virtualization support where available (VT-d/IOMMU)

The important distinction is that **Hyper-V Manager being installed does not prove that the hypervisor itself loaded at boot**.

### Verify in Windows

```powershell
systeminfo
```

Also check:

```powershell
bcdedit /enum | findstr /i hypervisor
```

Hyper-V can be installed with:

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

If the management tools exist but VMs report that the hypervisor is not running:

```powershell
bcdedit /set hypervisorlaunchtype auto
```

Then reboot.

### Failure ownership

If a VM cannot even power on, the problem is below Kubernetes. Investigate:

1. BIOS/UEFI virtualization settings
2. Windows boot configuration
3. Hyper-V Windows feature
4. Hyper-V services
5. VM configuration

---

# 3. Hyper-V Networking

An external Hyper-V virtual switch was created:

```text
K8s-External
```

It is attached to the Windows host's physical Ethernet adapter.

Conceptually:

```text
                         +-- Windows vNIC
Physical Ethernet NIC -- vSwitch -- k8s-worker1 vNIC
                         +-- k8s-worker2 vNIC
```

The physical NIC becomes the uplink for the virtual switch. Windows itself connects through a virtual adapter similar to:

```text
vEthernet (K8s-External)
```

The switch was created with management-OS access enabled so Windows could continue to use the same physical Ethernet connection.

### Why an external switch?

The workers should behave like ordinary machines on the home LAN. Each gets:

- its own MAC address
- its own LAN IP
- a local DHCP reservation
- direct connectivity to `control-plane1`
- direct connectivity to other nodes

This avoids hiding the workers behind Hyper-V's Default Switch/NAT network.

### Useful Windows commands

```powershell
Get-VMSwitch
```

```powershell
Get-VMNetworkAdapter -VMName "k8s-worker1"
```

### Useful Linux commands

```bash
ip -br address
ip route
ip link show eth0
ping -c 3 192.0.2.1
ping -c 3 192.0.2.10
```

If the VM has no usable LAN address, **do not troubleshoot Kubernetes yet**. Fix Hyper-V networking, DHCP, or Ubuntu networking first.

---

# 4. Worker VM Configuration

The current virtual workers are Ubuntu Server Generation 2 Hyper-V VMs.

## k8s-worker1

- 4 vCPU
- approximately 6 GB configured RAM
- 60 GB dynamically expanding VHDX
- static memory
- automatic checkpoints disabled
- Generation 2 / UEFI
- Secure Boot template: **Microsoft UEFI Certificate Authority**
- external switch: `K8s-External`
- static MAC: `02:00:00:00:01:01`
- initial LAN IP: `192.0.2.21`

## k8s-worker2

- 4 vCPU
- 4 GB configured RAM
- 60 GB dynamically expanding VHDX
- static memory
- automatic checkpoints disabled
- Generation 2 / UEFI
- Secure Boot template: **Microsoft UEFI Certificate Authority**
- external switch: `K8s-External`
- static MAC: `02:00:00:00:01:02`
- initial LAN IP: `192.0.2.22`

> Ubuntu reports somewhat less usable RAM than the configured Hyper-V value because some address space is reserved by firmware/kernel/virtual hardware.

### Linux Secure Boot gotcha

A Generation 2 Ubuntu VM may refuse to boot an otherwise-valid ISO if the Hyper-V Secure Boot template is still set to the Windows-only template.

For Ubuntu/Linux use:

```text
Settings
 -> Security
 -> Enable Secure Boot
 -> Template: Microsoft UEFI Certificate Authority
```

---

# 5. Machine Identity

Each node needs a unique and stable identity.

Current node names:

```text
control-plane1
k8s-worker1
k8s-worker2
```

Check identity with:

```bash
hostnamectl
hostname -I
cat /etc/machine-id
sudo cat /sys/class/dmi/id/product_uuid
ip link show
```

Stable Hyper-V MAC addresses allow local DHCP reservations to keep worker IPs predictable.

Duplicate hostnames, MAC addresses, machine IDs, or UUIDs can create extremely confusing registration and networking problems.

---

# 6. Linux Kernel Prerequisites

The following modules were loaded on all Kubernetes nodes:

```text
overlay
br_netfilter
```

Persistent module configuration:

```text
/etc/modules-load.d/k8s.conf
```

Created with:

```bash
printf "overlay\nbr_netfilter\n" | sudo tee /etc/modules-load.d/k8s.conf
```

## Why `overlay`?

Container images are made from filesystem layers. OverlayFS combines those layers into the filesystem a container sees.

Conceptually:

```text
base image layer
+ package/application layer
+ writable container layer
= container filesystem
```

## Why `br_netfilter`?

Pod/container traffic often traverses Linux bridges. `br_netfilter` makes bridged packets visible to the kernel's network filtering hooks so Kubernetes networking behaves as expected.

---

# 7. Kernel Network Settings

The following sysctls were enabled on every node:

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

Persistent configuration:

```text
/etc/sysctl.d/k8s.conf
```

Created with:

```bash
printf "net.bridge.bridge-nf-call-iptables = 1\nnet.bridge.bridge-nf-call-ip6tables = 1\nnet.ipv4.ip_forward = 1\n" | sudo tee /etc/sysctl.d/k8s.conf
```

Applied with:

```bash
sudo sysctl --system
```

Verify:

```bash
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
```

### Why IPv4 forwarding?

A Kubernetes node partly acts like a router. Packets may need to travel:

```text
pod interface -> node network stack -> another pod/node
```

Without forwarding, cross-network traffic cannot be routed correctly.

---

# 8. Swap

Swap was disabled on every node for the initial cluster build.

Immediately:

```bash
sudo swapoff -a
```

A backup of `/etc/fstab` was created before modification:

```bash
sudo cp /etc/fstab /etc/fstab.pre-k8s
```

Swap entries were commented so swap stays disabled after reboot.

Verify:

```bash
swapon --show
```

Expected result: no output.

### Why disable swap?

For this initial build, disabling swap keeps memory accounting and kubelet behavior predictable. Kubernetes can be configured for swap-aware behavior, but that adds another variable while learning the base system.

If kubelet refuses to start after a reboot, checking swap should be an early troubleshooting step.

---

# 9. Cgroups

All nodes reported:

```text
cgroup2fs
```

The hosts use systemd with cgroup v2, so both kubelet and containerd were configured to use the **systemd cgroup driver**.

Cgroups are what enforce process/resource boundaries such as CPU and memory allocation.

A kubelet/container-runtime cgroup-driver mismatch can cause unstable node behavior or failed pod creation.

---

# 10. Container Runtime: containerd

Kubernetes does not directly create containers.

The stack is:

```text
Kubernetes desired state
        |
        v
     kubelet
        |
        | CRI
        v
    containerd
        |
        v
       runc
        |
        v
 Linux namespaces + cgroups + processes
```

## CRI

CRI means **Container Runtime Interface**.

Kubelet uses CRI to ask the runtime to perform actions such as:

- pull an image
- create a pod sandbox
- start a container
- stop a container
- inspect container state

The containerd socket is:

```text
/run/containerd/containerd.sock
```

---

# 11. containerd on `control-plane1`

`control-plane1` already used containerd underneath Docker.

Docker-managed workloads live in containerd's `moby` namespace. Kubernetes uses the `k8s.io` namespace.

Conceptually:

```text
Docker -------> containerd namespace: moby
Kubernetes ---> containerd namespace: k8s.io
```

The same containerd daemon can therefore support both Docker and Kubernetes.

## Important discovery

The Docker-installed config contained:

```toml
disabled_plugins = ["cri"]
```

Containerd v2 translated that internally to the old gRPC CRI plugin being disabled.

Live config inspection showed:

```text
version = 3
disabled_plugins = ['io.containerd.grpc.v1.cri']
SystemdCgroup = false
```

The containerd configuration was backed up and changed to enable CRI and systemd cgroups.

Current essential configuration:

```toml
version = 3
disabled_plugins = []

[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc]
  runtime_type = 'io.containerd.runc.v2'

[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
  SystemdCgroup = true
```

After restarting containerd, CRI reported:

```text
io.containerd.cri.v1   images    ok
io.containerd.cri.v1   runtime   ok
io.containerd.grpc.v1  cri       ok
```

Existing Docker workloads remained running:

```text
<existing-docker-service-1>
<existing-docker-service-2>
<existing-docker-service-3>
```

### Troubleshooting containerd

```bash
systemctl status containerd
```

```bash
journalctl -u containerd -b
```

```bash
sudo ctr plugins ls | grep -i cri
```

```bash
sudo containerd config dump | grep -E 'disabled_plugins|SystemdCgroup'
```

Docker workloads:

```bash
docker ps
```

Kubernetes/containerd workloads:

```bash
sudo crictl ps -a
sudo crictl pods
```

Do not expect `docker ps` to become the normal way to inspect Kubernetes workloads.

---

# 12. Kubernetes Packages

The following packages were installed on all nodes:

```text
kubeadm
kubelet
kubectl
```

All nodes are on Kubernetes v1.36.x. The control plane initialized at:

```text
v1.36.3
```

The packages were held to prevent ordinary unattended package upgrades from unexpectedly changing Kubernetes versions:

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

## What each program does

### kubeadm

Bootstrap/lifecycle tool.

It:

- initializes a control plane
- creates cluster certificates
- creates bootstrap tokens
- writes Kubernetes configuration
- joins workers/control-plane nodes
- assists with controlled upgrades

It is not the component continuously running workloads.

### kubelet

The node agent running on every Kubernetes machine.

Its job is essentially:

> What pods should exist on this machine, and does observed reality match the desired state?

If not, kubelet works with containerd to reconcile the difference.

### kubectl

The administrator/client CLI.

```text
kubectl -> Kubernetes API server -> cluster state/controllers
```

`kubectl` generally talks to the API server rather than manipulating containers directly.

---

# 13. Stable Kubernetes API Endpoint

A local DNS record was created:

```text
k8s-api.lab.example -> 192.0.2.10
```

This currently resolves directly to `control-plane1`.

Verify:

```bash
getent ahostsv4 k8s-api.lab.example
```

Expected IPv4 address:

```text
192.0.2.10
```

### Why use a DNS endpoint instead of the raw IP?

Today:

```text
k8s-api.lab.example -> control-plane1
```

Future HA design:

```text
k8s-api.lab.example
        |
        v
load balancer / virtual IP
        |
        +-- control-plane1
        +-- control-plane2
        +-- control-plane3
```

Workers and clients can continue using the same API endpoint even if the backend control-plane topology changes.

The Kubernetes API listens on TCP port:

```text
6443
```

Basic reachability test:

```bash
curl -k https://k8s-api.lab.example:6443/livez
```

Expected response when the API server is healthy:

```text
ok
```

---

# 14. Network Ranges

Home LAN:

```text
192.0.2.0/24
```

Existing Docker networks on `control-plane1` at bootstrap time:

```text
172.17.0.0/16
172.18.0.0/16
```

Kubernetes pod network:

```text
10.244.0.0/16
```

Kubernetes service network:

```text
10.96.0.0/12
```

These ranges were deliberately chosen not to overlap the home LAN or existing Docker bridge networks.

Network overlap is a classic source of deeply confusing routing problems.

---

# 15. kubeadm Configuration

The initial control plane was built from:

```text
/etc/kubernetes/kubeadm-config.yaml
```

Configuration:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "192.0.2.10"
  bindPort: 6443
nodeRegistration:
  name: "control-plane1"
  criSocket: "unix:///run/containerd/containerd.sock"
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
clusterName: "homelab"
kubernetesVersion: "v1.36.3"
controlPlaneEndpoint: "k8s-api.lab.example:6443"
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  dnsDomain: "cluster.local"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

The configuration was validated before initialization:

```bash
sudo kubeadm config validate --config /etc/kubernetes/kubeadm-config.yaml
```

Images were pre-pulled with:

```bash
sudo kubeadm config images pull --config /etc/kubernetes/kubeadm-config.yaml
```

The control plane was initialized with:

```bash
sudo kubeadm init --config /etc/kubernetes/kubeadm-config.yaml
```

---

# 16. What `kubeadm init` Created

## PKI / certificates

```text
/etc/kubernetes/pki
```

Kubernetes components use certificates to authenticate to one another and to the API server.

## Kubeconfig files

```text
/etc/kubernetes/admin.conf
/etc/kubernetes/controller-manager.conf
/etc/kubernetes/scheduler.conf
/etc/kubernetes/kubelet.conf
```

A kubeconfig generally describes:

- which API server to contact
- which certificate authority to trust
- which identity/credential to present
- which cluster/context to use

## Static pod manifests

```text
/etc/kubernetes/manifests
```

Contains manifests for control-plane components such as:

```text
kube-apiserver
kube-controller-manager
kube-scheduler
etcd
```

The kubelet directly watches this directory and asks containerd to run those components.

This is an important bootstrap concept: **the API server itself can run as a static pod before the scheduler is available.**

## etcd

Etcd is the authoritative database for Kubernetes cluster state.

Typical local data location:

```text
/var/lib/etcd
```

It stores objects such as:

- Deployments
- Pods
- Services
- Secrets
- ConfigMaps
- Nodes
- RBAC objects
- controller state

If etcd is lost without a usable backup, the cluster loses its authoritative state.

---

# 17. kubectl Access for the Administrative User

The administrative kubeconfig was copied to the normal user's home directory:

```bash
mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
chmod 600 "$HOME/.kube/config"
```

This allows:

```bash
kubectl get nodes
```

without `sudo`.

Important: this is not because the Linux user was added to a special Kubernetes group. The kubeconfig contains a powerful Kubernetes administrator identity.

Protect:

```text
~/.kube/config
```

as an administrative credential.

---

# 18. Initial Control-Plane State

Immediately after `kubeadm init`, the control-plane pods were healthy:

```text
etcd-control-plane1                      Running
kube-apiserver-control-plane1            Running
kube-controller-manager-control-plane1   Running
kube-scheduler-control-plane1            Running
kube-proxy                       Running
```

But the node initially reported:

```text
control-plane1   NotReady
```

and CoreDNS pods were:

```text
Pending
```

This was expected because a CNI pod network had not yet been installed.

---

# 19. Flannel / CNI Networking

Flannel is the cluster's initial CNI network plugin.

Kubernetes defines the desired networking model, but a CNI plugin implements pod networking.

There are now two distinct network layers:

```text
NODE NETWORK
192.0.2.0/24

POD NETWORK
10.244.0.0/16
```

Conceptually:

```text
Pod on worker1
10.244.1.x
      |
      | Flannel/VXLAN encapsulation
      v
worker1 LAN IP 192.0.2.x
      |
      | physical/home LAN
      v
worker2 LAN IP 192.0.2.x
      |
      | decapsulation
      v
Pod on worker2
10.244.2.x
```

Flannel normally uses a VXLAN overlay to carry pod packets across the existing node network.

Flannel was installed from a pinned release manifest rather than a floating `latest` URL.

Once Flannel initialized:

- pod networking became available
- CoreDNS could start
- `control-plane1` transitioned from `NotReady` to `Ready`

### Inspect Flannel

```bash
kubectl get pods -n kube-flannel -o wide
```

```bash
kubectl logs -n kube-flannel <flannel-pod-name>
```

Useful node-side inspection:

```bash
ip -br link
ip route
ls -la /etc/cni/net.d
```

Common interfaces include:

```text
cni0
flannel.1
```

---

# 20. What `Ready` Means

`Ready` means kubelet is reporting that the node is healthy enough to accept Kubernetes workloads.

It does **not** mean every service in the cluster is healthy.

Inspect node health with:

```bash
kubectl describe node control-plane1
```

Important conditions include:

```text
Ready
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
```

If a node reports `NotReady`, `kubectl describe node` is much more useful than staring at the word `NotReady` alone.

---

# 21. Worker Enrollment

Workers are prepared with:

- Ubuntu Server
- stable hostname/MAC/IP
- swap disabled
- required kernel modules
- required sysctls
- containerd
- CRI enabled
- systemd cgroup driver
- kubelet
- kubeadm
- kubectl

A worker becomes an actual cluster member only after running a `kubeadm join` command generated by the control plane.

Generate a fresh command on `control-plane1`:

```bash
kubeadm token create --print-join-command
```

Run the resulting command on each worker with `sudo`.

Example shape only:

```bash
sudo kubeadm join k8s-api.lab.example:6443 \
  --token <temporary-bootstrap-token> \
  --discovery-token-ca-cert-hash sha256:<cluster-ca-hash>
```

> Do not commit live bootstrap tokens or other secrets to Git.

The token proves the worker is authorized to enroll. The CA hash helps the worker prove it is talking to the intended Kubernetes cluster.

---

# 22. Kubernetes Desired State Mental Model

Kubernetes is fundamentally a **desired-state reconciliation system**.

Example:

```text
Desired state:  3 web pods
Observed state: 2 web pods
```

A controller notices the mismatch and works to create the missing pod.

A strong troubleshooting pattern is therefore:

1. What desired state was submitted?
2. Which controller owns that object?
3. What observed state does Kubernetes report?
4. At which layer did reconciliation stop?

This is more useful than memorizing dozens of commands without understanding why they exist.

---

# 23. Troubleshooting From the Bottom Up

When something breaks, work upward through the stack.

## Layer 1: Is the machine alive?

```bash
ping <node-ip>
ssh <admin-user>@<node-ip>
```

If not, investigate:

- VM power state
- Hyper-V
- physical networking
- virtual switch
- DHCP
- Ubuntu boot state

This is not yet a Kubernetes problem.

---

## Layer 2: Is Linux healthy?

```bash
uptime
free -h
df -h
ip -br address
ip route
systemctl --failed
```

Look for:

- memory exhaustion
- full filesystems
- missing routes
- failed services
- unexpected reboot/load

---

## Layer 3: Is containerd healthy?

```bash
systemctl status containerd
```

```bash
journalctl -u containerd -b
```

```bash
sudo crictl ps -a
```

If containerd is down, kubelet cannot create or manage containers.

---

## Layer 4: Is kubelet healthy?

```bash
systemctl status kubelet
```

```bash
journalctl -u kubelet -b
```

Kubelet logs commonly reveal problems involving:

- swap
- CRI/runtime availability
- CNI/network initialization
- certificates
- API server connectivity
- node configuration

---

## Layer 5: Can the API server be reached?

```bash
getent ahostsv4 k8s-api.lab.example
```

```bash
curl -k https://k8s-api.lab.example:6443/livez
```

```bash
kubectl cluster-info
```

If the API server is unavailable on the control-plane machine, inspect static control-plane containers:

```bash
sudo crictl ps -a
```

Then inspect a specific container:

```bash
sudo crictl logs <container-id>
```

---

## Layer 6: Is the node healthy?

```bash
kubectl get nodes -o wide
```

```bash
kubectl describe node <node-name>
```

Read the `Conditions` and `Events` sections.

---

## Layer 7: Did the pod schedule?

```bash
kubectl get pods -A -o wide
```

```bash
kubectl describe pod <pod-name> -n <namespace>
```

A `Pending` pod may indicate:

- insufficient CPU/RAM
- taints/tolerations
- node affinity rules
- missing storage
- scheduler constraints

---

## Layer 8: Did the application/container start?

```bash
kubectl logs <pod-name> -n <namespace>
```

For a specific container:

```bash
kubectl logs <pod-name> -n <namespace> -c <container-name>
```

For the previous crashed instance:

```bash
kubectl logs <pod-name> -n <namespace> --previous
```

`CrashLoopBackOff` is usually much closer to an application/configuration problem than a physical networking problem.

---

## Layer 9: Read cluster events

```bash
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Events often reveal exactly what Kubernetes attempted and why it failed.

---

# 24. Symptom -> Likely Owner

| Symptom | Likely subsystem |
|---|---|
| VM will not power on | UEFI/BIOS, Hyper-V, VM configuration |
| VM has no LAN IP | Hyper-V switch, virtual NIC, DHCP, Ubuntu networking |
| Hostname resolves incorrectly | local DNS |
| `containerd` inactive | containerd/systemd/runtime configuration |
| `kubelet` constantly restarts | node config, CRI, swap, certificates, API connectivity |
| `kubectl` cannot connect | DNS, API server, kubeconfig, certificates |
| Node is `NotReady` | kubelet, CNI, runtime, resource pressure |
| Pod is `Pending` | scheduler, resources, taints, storage |
| Pod is `CrashLoopBackOff` | application process/configuration |
| Pod is `ImagePullBackOff` | image name, registry, credentials, DNS/network |
| Pods cannot reach each other | CNI/Flannel, routing, forwarding, firewall |
| Service names do not resolve | CoreDNS or service configuration |
| Existing pods run but nothing new schedules | API server / scheduler / control-plane problem |

---

# 25. Fast Health-Check Commands

## Cluster overview

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
```

## Node details

```bash
kubectl describe node <node-name>
```

## API health

```bash
curl -k https://k8s-api.lab.example:6443/livez
```

## Recent events

```bash
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

## kubelet

```bash
systemctl status kubelet
journalctl -u kubelet -b
```

## containerd

```bash
systemctl status containerd
journalctl -u containerd -b
sudo ctr plugins ls | grep -i cri
```

## Host network

```bash
ip -br address
ip route
```

## Kernel prerequisites

```bash
lsmod | grep -E '^(overlay|br_netfilter)\b'
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
```

## Swap

```bash
swapon --show
```

---

# 26. Important Files and Directories

| Path | Purpose |
|---|---|
| `/etc/kubernetes/kubeadm-config.yaml` | Initial kubeadm cluster configuration |
| `/etc/kubernetes/admin.conf` | Cluster-admin kubeconfig generated by kubeadm |
| `~/.kube/config` | Normal user's kubectl configuration |
| `/etc/kubernetes/pki/` | Kubernetes certificates and keys |
| `/etc/kubernetes/manifests/` | Static control-plane pod manifests |
| `/var/lib/etcd/` | Local etcd database |
| `/etc/containerd/config.toml` | containerd runtime configuration |
| `/etc/modules-load.d/k8s.conf` | persistent kernel modules |
| `/etc/sysctl.d/k8s.conf` | persistent Kubernetes networking sysctls |
| `/etc/cni/net.d/` | CNI configuration |
| `/var/lib/kubelet/` | kubelet state/configuration |

Do not casually delete or overwrite these directories while troubleshooting.

---

# 27. Things That Should NOT Be Committed to Git

Do not commit:

- live `kubeadm join` tokens
- private keys from `/etc/kubernetes/pki`
- `admin.conf`
- `~/.kube/config`
- Kubernetes Secrets containing real credentials
- registry credentials
- Tailscale auth keys
- passwords/API tokens

Useful configuration and documentation absolutely should be version-controlled, but credentials should not.

---

# 28. Current Build Status

At the point this document was created:

```text
[READY] Windows hardware virtualization
[READY] Hyper-V
[READY] K8s-External virtual switch
[READY] k8s-worker1 VM
[READY] k8s-worker2 VM
[READY] stable node identities / DHCP planning
[READY] swap disabled
[READY] kernel modules
[READY] forwarding / bridge sysctls
[READY] containerd on all current nodes
[READY] CRI enabled
[READY] systemd cgroup driver
[READY] kubeadm / kubelet / kubectl
[READY] k8s-api.lab.example DNS endpoint
[READY] control-plane1 control plane
[READY] Flannel CNI
[READY] control-plane1 node status
[NEXT ] join k8s-worker1
[NEXT ] join k8s-worker2
[LATER] add separate physical k8s-worker3
[LATER] deploy monitoring stack
[LATER] deploy GitOps/automation tooling
[LATER] controlled failure/redundancy labs
[LATER] design three-node control-plane HA
```

---

# 29. Planned Next Steps

1. Join `k8s-worker1` and `k8s-worker2`.
2. Verify Flannel and kube-proxy DaemonSets appear on all nodes.
3. Verify pod-to-pod networking across nodes.
4. Deploy a small replicated test workload.
5. Expose it through a Service.
6. Practice cordon/drain and node failure.
7. Add the separate physical worker.
8. Deploy Prometheus/Grafana/Alertmanager.
9. Add Loki/logging.
10. Add Argo CD and move application configuration into GitOps.
11. Build backup/recovery procedures, especially for etcd.
12. Intentionally break components and diagnose from the bottom up.

---

# 30. Guiding Principle

The cluster is not one giant subsystem called "Kubernetes." It is a stack:

```text
hardware / firmware
        ↓
hypervisor / physical machine
        ↓
Ethernet / IP / DNS
        ↓
Linux kernel
        ↓
cgroups / namespaces / filesystems
        ↓
containerd + CRI
        ↓
kubelet
        ↓
CNI / Flannel
        ↓
Kubernetes API + etcd + controllers + scheduler
        ↓
Services / workloads / applications
```

When something fails, find the **lowest broken layer first**.

A higher layer cannot repair a lower layer it depends on.

That is the central troubleshooting model for this homelab.
