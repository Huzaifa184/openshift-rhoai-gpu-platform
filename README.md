# Red Hat OpenShift AI : GPU Platform

Production-pattern OpenShift AI 3.4.0 cluster on bare-metal ESXi with GPU passthrough,
KServe model serving, and a verified DeepSeek R1 inference endpoint. All self managed

---

## Platform 

| Component | Status | Comments |
|-----------|--------|----------|
| OpenShift 4.21 UPI | Running | Connected, self-hosted DNS + LB |
| NVIDIA GPU Operator | Running | Driver 580.126.20 · DCGM metrics live |
| RHOAI 3.4.0 | Running | DataScienceCluster reconciled |
| MinIO AIStor | Running | In-cluster S3 · 899GB capacity |
| KServe RawDeployment | Running | InferenceService Ready=True |
| DeepSeek R1 1.5B w8a8 | Serving | 12–17 tok/s · chain-of-thought confirmed |

---

## Hardware

### ESXi Host : Dell Precision Workstation T5810

| Component | Specification |
|-----------|---------------|
| CPU | Intel Xeon E5-2620 v4 · 8 cores · 16 threads · 2.10 GHz base · 3.00 GHz turbo · Broadwell-EP 14nm |
| RAM | 96 GB ECC DDR4 |
| Storage | 128 GB SSD (ESXi OS) · 2 TB HDD (VM storage) |
| GPU | NVIDIA GeForce GTX 1660 Super · TU116 · Turing (SM 7.5) · 6 GB GDDR6 |
| Hypervisor | VMware ESXi 7.0 |

### VMware Workstation Pro Host : Lenovo Legion 5 16IRX9

| Component | Specification |
|-----------|---------------|
| CPU | Intel Core i7-14650HX · 16 cores · 24 threads · 2.20 GHz base · Raptor Lake-HX |
| RAM | 64 GB DDR5 |
| Storage | 2 TB NVMe |
| GPU | NVIDIA GeForce RTX 4060 Laptop · 8 GB GDDR6 *(not used for cluster workloads)* |
| Hypervisor | VMware Workstation Pro (Windows) |

---

<img width="6048" height="5548" alt="IMG_5489 - Copy (2)" src="https://github.com/user-attachments/assets/da09e065-4a11-4047-a7a8-ad9c406814c7" />



## Cluster Topology

| Role | Count | Hypervisor | Host |
|------|-------|------------|------|
| Master nodes | 3× RHCOS VMs | VMware Workstation Pro | Laptop |
| Worker nodes | 3× RHCOS VMs | ESXi 7.0 | Dell Workstation |
| GPU worker (worker1) | 1× RHCOS VM | ESXi 7.0 · DirectPath I/O | Dell Workstation |

## Cluster Specifications

**OpenShift:** 4.21 · Connected UPI · Self-hosted HAProxy + BIND9
**RHOAI:** 3.4.0 Self-Managed

| Node | Role | CPU | RAM | Storage | Notes |
|------|------|-----|-----|---------|-------|
| master1.openshift-ai.huzaifa.lab | control-plane | 8 cores | 16 GB | 100 GB | Nested · Workstation Pro |
| master2.openshift-ai.huzaifa.lab | control-plane | 8 cores | 16 GB | 100 GB | Nested · Workstation Pro |
| master3.openshift-ai.huzaifa.lab | control-plane | 8 cores | 16 GB | 100 GB | Nested · Workstation Pro |
| worker1.openshift-ai.huzaifa.lab | worker · GPU | 16 cores | 64 GB | 200 GB | ESXi · GTX 1660 Super passthrough |
| worker2.openshift-ai.huzaifa.lab | worker | 8 cores | 24 GB | 200 GB | ESXi |
| worker3.openshift-ai.huzaifa.lab | worker | 8 cores | 24 GB | 200 GB | ESXi |
| ocp-svc.huzaifa.lab | bastion · HAProxy · DNS | 4 cores | 8 GB | 150 GB | RHEL 10 · Workstation Pro |

---
<img width="949" height="504" alt="image" src="https://github.com/user-attachments/assets/e0332e2d-55fb-4dbe-b84e-cd3128ecc0b3" />
> OpenShift reports slightly lower values than provisioned due to RHCOS kernel and system process reservations, expected behaviour

---
<img width="1891" height="1014" alt="image" src="https://github.com/user-attachments/assets/b6b91083-fa30-49a2-8ad7-d43309f57e2f" />


---


<img width="859" height="511" alt="image" src="https://github.com/user-attachments/assets/32f5b78a-ad3b-4d48-b32c-d8d98739681b" />
---


## Architecture

### Topology

**Hybrid hypervisor : deliberate, not default.**
Master nodes run nested inside VMware Workstation Pro on a Windows laptop.
Worker nodes run on bare-metal ESXi on a separate physical workstation.
The GPU-bearing worker (worker1) is on bare-metal ESXi : the only viable path
for PCI passthrough on consumer hardware.

**Connected UPI : not IPI.**
Self-hosted HAProxy and BIND9 on a dedicated RHEL 10 bastion node. Mirrors the
install pattern used in regulated environments where infrastructure is explicitly
managed rather than abstracted away. Full control over DNS and load-balancing
at the cost of manual provisioning.

### Infrastructure Setup

- **Static IP allocation** : All nodes (masters, workers, bastion) assigned static IPs
  on the internal home network prior to installation. No DHCP dependency for cluster nodes
  ensures predictable DNS resolution and HAProxy backend targeting across reboots.

- **Bridged networking on VMware Workstation Pro** : Master node VMs configured with
  bridged network adapters, placing them directly on the physical LAN segment alongside
  the ESXi-hosted workers. This allows all cluster nodes to communicate on the same L2
  domain regardless of which hypervisor hosts them : a requirement for OVN-Kubernetes
  and the OpenShift API VIP to function correctly across the hybrid topology.

- **Bastion node (RHEL 10 VM on Workstation Pro)** : Single VM serving three roles:
  - **HAProxy** : Layer 4 load balancer handling API (6443), Machine Config (22623),
    and ingress (80/443) VIPs. Configured with separate frontends and backends for
    bootstrap, master, and worker node pools.
  - **BIND9 (named)** : Authoritative DNS for the `openshift-ai.huzaifa.lab` zone.
    Forward and reverse lookup zones configured for all cluster nodes, API VIP,
    and wildcard ingress (`*.apps`). Upstream forwarding to home router for
    external resolution.
  - **Apache httpd** : Served RHCOS ignition files (`bootstrap.ign`, `master.ign`,
    `worker.ign`) generated by the OpenShift installer during UPI bootstrap.
    Nodes pulled ignition over HTTP on first boot to complete provisioning.

- **UPI install flow** : `install-config.yaml` generated and customised before
  manifests were created. Cluster manifests and ignition configs produced via
  `openshift-install create ignition-configs`. RHCOS nodes booted from ISO,
  pointed at the bastion HTTP server for ignition, and bootstrapped in sequence:
  bootstrap → masters (etcd quorum) → workers → bootstrap node removed.

- **Configuration files** : Redacted versions of `install-config.yaml`,
  `haproxy.cfg`, `named.conf`, and forward/reverse zone files available
  in [`infra/`](infra/) for reference.

### Stack

| Layer | Operator / Tool | Version |
|-------|----------------|---------|
| Node Feature Discovery | NFD Operator | 4.21.0 |
| GPU Enablement | NVIDIA GPU Operator (Certified) | 26.3.2 |
| AI Platform | Red Hat OpenShift AI (RHODS) | 3.4.0 |
| Model Serving | KServe · vLLM ServingRuntime | v0.18.0 |
| Object Storage | MinIO AIStor Operator | 2026.4.13 |
| Persistent Storage | Local Storage Operator | 4.21.0 |
| Service Mesh | OpenShift Service Mesh | 3.2.0 |
| GPU Sharing | NVIDIA Device Plugin · time-slicing | 26.3.2 |
