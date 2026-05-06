# 🚀 Inside RHOSP: Complete Architecture & End-to-End Flow

> **Red Hat OpenStack Platform — A distributed cloud operating system powering enterprise private clouds, telecom environments, and large-scale datacenters worldwide.**

[![OpenStack](https://img.shields.io/badge/OpenStack-RHOSP-red?style=flat-square&logo=openstack)](https://www.redhat.com/en/technologies/linux-platforms/openstack-platform)
[![KVM](https://img.shields.io/badge/Hypervisor-KVM%20%2F%20libvirt-orange?style=flat-square)](https://www.linux-kvm.org/)
[![RabbitMQ](https://img.shields.io/badge/Messaging-RabbitMQ-purple?style=flat-square&logo=rabbitmq)](https://www.rabbitmq.com/)
[![MariaDB](https://img.shields.io/badge/Database-MariaDB%20Galera-blue?style=flat-square&logo=mariadb)](https://mariadb.com/)
[![HAProxy](https://img.shields.io/badge/LB-HAProxy%20VIP-brightgreen?style=flat-square)](http://www.haproxy.org/)

-----

## 📋 Table of Contents

1. [What is RHOSP?](#-what-is-rhosp)
1. [Traditional OpenStack Architecture](#-traditional-openstack-architecture-rpmsystemd)
1. [Modern RHOSP — Containerized Architecture](#-modern-rhosp-containerized-architecture)
1. [End-to-End VM Launch Flow](#-end-to-end-vm-launch-flow-13-steps)
1. [Control Plane vs Compute Plane](#-control-plane-vs-compute-plane)
1. [Database & Messaging Architecture](#-database--messaging-architecture)
1. [Networking Flow](#-networking-flow)
1. [Storage Architecture](#-storage-architecture)
1. [SSL / TLS Flow](#-ssl--tls-flow)
1. [RHOSP vs OpenShift Comparison](#-rhosp-vs-openshift-comparison)
1. [Key Architectural Learnings](#-key-architectural-learnings)

-----

## 🌐 What is RHOSP?

**RHOSP (Red Hat OpenStack Platform)** is an enterprise-grade **Infrastructure-as-a-Service (IaaS)** platform — not simply a virtualization layer. It is a **full-stack distributed cloud operating system** that orchestrates compute, networking, storage, identity, and messaging across multiple nodes in a coordinated control plane.

|Resource                |Description                                |
|------------------------|-------------------------------------------|
|🖥️ **Virtual Machines**  |KVM-based VMs are the primary workload unit|
|🌐 **Virtual Networks**  |SDN via Neutron + Open vSwitch / OVN       |
|💾 **Persistent Volumes**|Block storage managed by Cinder            |
|🌍 **Floating IPs**      |External-to-internal network mapping       |
|⚡ **Hypervisor**        |KVM / QEMU managed via libvirt             |


> 💡 **Key insight:** Each OpenStack service owns its own API endpoint, its own database schema, and its own RabbitMQ queue bindings. They are independently scalable, loosely coupled distributed services.

-----

## 🏗️ Traditional OpenStack Architecture (RPM/systemd)

In the original (pre-containerized) RHOSP, all services ran directly on Linux hosts as RPM packages managed by `systemd`.

### Controller Node — Brain of the Platform

```
┌─────────────────────────────────────────────────────┐
│                   CONTROLLER NODE                   │
│            (All OpenStack APIs live here)           │
├─────────────┬───────────────────┬───────────────────┤
│  Identity   │    Compute API    │   Network API     │
│  Keystone   │    Nova API       │   Neutron Server  │
├─────────────┼───────────────────┼───────────────────┤
│  Image API  │   Volume API      │   Web UI          │
│  Glance     │   Cinder          │   Horizon         │
├─────────────┴───────────────────┴───────────────────┤
│  📨 RabbitMQ    🗄️ MariaDB Galera    ⚖️ HAProxy VIP  │
└─────────────────────────────────────────────────────┘
              Managed by: systemctl (systemd)
```

### Compute Node — Where VMs Actually Run

```
┌─────────────────────────────────────────┐
│              COMPUTE NODE               │
├─────────────────────────────────────────┤
│  ⚙️  nova-compute  (listens to RabbitMQ) │
│  🔧  libvirt       (hypervisor API)      │
│  ⚡  KVM / QEMU    (hardware virt)       │
│  🔀  Open vSwitch  (virtual switching)   │
│  🖥️  [VM] [VM] [VM] [VM]  (workloads)   │
└─────────────────────────────────────────┘
```

### Storage Node

```
┌─────────────────────────────────────────┐
│              STORAGE NODE               │
├─────────────────────────────────────────┤
│  💽  cinder-volume    (block agent)      │
│  🐙  Ceph OSD         (distributed)      │
│  🔌  iSCSI / FC SAN   (enterprise SAN)   │
│  📦  Swift            (object storage)   │
└─────────────────────────────────────────┘
```

### ⚠️ Problems with Traditional Architecture

|Problem                     |Impact                                        |
|----------------------------|----------------------------------------------|
|❌ RPM dependency conflicts  |Services could break each other during updates|
|❌ Difficult, risky upgrades |Full cluster downtime often required          |
|❌ Tightly coupled to host OS|Hard to scale individual services             |
|❌ Package version mismatches|Cross-node inconsistencies                    |

-----

## 🟣 Modern RHOSP — Containerized Architecture

Modern RHOSP runs all control plane services inside **Podman containers**, eliminating the dependency and upgrade problems of the RPM era.

```
┌──────────────────────────────────────────────────────────────┐
│                CONTAINERIZED CONTROL PLANE                   │
│                    (Podman / systemd)                        │
├──────────────┬──────────────┬──────────────┬────────────────┤
│🟣 nova-api   │🟣 keystone   │🟣 neutron    │🟣 glance-api  │
│   container  │   container  │   container  │   container    │
├──────────────┼──────────────┼──────────────┼────────────────┤
│🟣 cinder-api │🟣 rabbitmq   │🟣 mariadb    │🟣 horizon      │
│   container  │   container  │   container  │   container    │
└──────────────┴──────────────┴──────────────┴────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                   COMPUTE PLANE (bare metal)                 │
│              User workloads remain as KVM VMs                │
├──────────────────────────────────────────────────────────────┤
│   nova-compute  │  libvirt  │  KVM/QEMU  │  Open vSwitch    │
│   [VM] [VM] [VM] [VM] [VM] [VM] [VM] [VM] [VM]              │
└──────────────────────────────────────────────────────────────┘
```

> 🔑 **Critical distinction:**
> 
> - **Containers run OpenStack** (control plane services)
> - **OpenStack runs VMs** (user workloads)
> 
> These are two completely separate layers. RHOSP remains a **VM-first IaaS platform**.

-----

## ⚡ End-to-End VM Launch Flow (13 Steps)

```
 USER
  │
  ▼
[01] 🌐 User accesses Horizon Dashboard or REST API
  │       User submits: POST /v2.1/servers with flavor, image, network
  │
  ▼
[02] 📡 DNS resolves domain → HAProxy VIP
  │       e.g. openstack.company.internal → 10.0.0.100
  │
  ▼
[03] ⚖️ HAProxy VIP — TLS Termination + Load Balancing
  │       Terminates HTTPS, distributes to active controller nodes
  │
  ▼
[04] 🔑 Keystone — Authentication & Authorization
  │       Validates user credentials → Issues scoped Keystone token
  │       Token encodes: user_id, project_id, roles, expiration
  │
  ▼
[05] ⚙️ Nova API — VM Request Processing
  │       Validates flavor/image/network params
  │       Writes new instance record to nova_db (status: BUILD)
  │       Calls Glance and Placement APIs
  │
  ▼
[06] 🖼️ Glance — Image Validation
  │       Confirms image exists and status = ACTIVE
  │       Returns image metadata (format, size, location)
  │
  ▼
[07] 📍 Placement API — Compute Node Selection
  │       Queries resource providers across all compute hosts
  │       Applies: RAM filter, CPU filter, disk filter, affinity rules
  │       Returns: best-fit compute host
  │
  ▼
[08] 📨 RabbitMQ — Async Message Dispatch
  │       Nova Conductor publishes build task to AMQP exchange
  │       Routing key targets selected compute host's queue
  │       API returns 202 Accepted immediately (async pattern)
  │
  ▼
[09] 🖥️ nova-compute — Receives & Executes Task
  │       nova-compute agent on target host consumes message
  │       Pulls boot image from Glance (or Ceph directly)
  │
  ▼
[10] 🌐 Neutron — Network Configuration
  │       Creates virtual port, allocates IP from DHCP pool
  │       Programs Open vSwitch flows (VXLAN tunnel + local flows)
  │       Applies security group rules via iptables/nftables
  │
  ▼
[11] 💽 Cinder — Storage Attachment
  │       Provisions or attaches block volume via iSCSI/Ceph
  │       Presents block device to compute node OS
  │
  ▼
[12] ⚡ KVM / libvirt — VM Creation
  │       libvirt generates domain XML from Nova instance spec
  │       Calls QEMU/KVM to create + boot the virtual machine
  │       Assigns vCPUs, vRAM, disk, network tap interface
  │
  ▼
[13] ✅ VM Status → ACTIVE
        Nova updates nova_db: status = ACTIVE
        Neutron confirms port binding complete
        User sees ACTIVE status in Horizon / API response
```

-----

## 🧠 Control Plane vs Compute Plane

### Control Plane — “Brain / Orchestration Layer”

Runs on **Controller Nodes**. Handles all decisions, API requests, and state management.

|Service           |Role                                         |
|------------------|---------------------------------------------|
|**Keystone**      |Identity, authentication, token issuance     |
|**Nova API**      |Compute API, instance lifecycle orchestration|
|**Neutron Server**|Network API, SDN controller                  |
|**Glance**        |VM image registry and metadata store         |
|**Placement**     |Resource inventory and scheduling decisions  |
|**Cinder API**    |Block storage API and volume lifecycle       |
|**RabbitMQ**      |Internal async message bus (AMQP)            |
|**MariaDB Galera**|Persistent state store for all services      |
|**HAProxy**       |VIP, load balancing, TLS termination         |
|**Horizon**       |Web-based dashboard UI                       |

### Compute Plane — “Actual Workload Execution Layer”

Runs on **Compute Nodes**. Executes instructions from control plane and hosts VMs.

|Component        |Role                                              |
|-----------------|--------------------------------------------------|
|**nova-compute** |Compute agent, listens to RabbitMQ, drives libvirt|
|**KVM**          |Hardware-accelerated virtualization engine        |
|**libvirt**      |Hypervisor management API (domain XML)            |
|**QEMU**         |Machine emulator used with KVM acceleration       |
|**Open vSwitch** |Virtual switching, VXLAN overlay, OpenFlow rules  |
|**neutron-agent**|Configures OVS and security groups locally        |
|**Running VMs**  |The actual user workloads                         |

-----

## 🗄️ Database & Messaging Architecture

### MariaDB Galera Cluster — One DB Per Service

```
┌─────────────────────────────────────────────────────┐
│              MariaDB Galera Cluster                 │
│           (Synchronous Multi-Master HA)             │
├──────────┬──────────┬──────────┬──────────┬─────────┤
│ nova_db  │neutron_db│cinder_db │glance_db │keystone │
│          │          │          │          │   _db   │
└──────────┴──────────┴──────────┴──────────┴─────────┘
     ↑          ↑          ↑          ↑          ↑
   Nova      Neutron    Cinder     Glance    Keystone
   API        API        API        API        API
```

> 📌 **Each service owns its own schema.** Services never share tables — all cross-service communication happens through APIs or RabbitMQ, never direct DB queries.

### RabbitMQ — Internal Messaging Bus

```
Nova API ──publish──► [exchange] ──route──► [nova-compute queue]
                                                     │
                                              nova-compute
                                              (consumer)

Neutron API ──publish──► [exchange] ──route──► [neutron-agent queue]
                                                       │
                                                neutron-agent
                                                (consumer)
```

RabbitMQ decouples the **API layer** (synchronous, user-facing) from the **execution layer** (asynchronous, compute-side), enabling:

- Retry on failure
- Backpressure handling
- Independent scaling of API vs compute

-----

## 🌐 Networking Flow

### North-South Traffic (External → VM)

```
Internet User
     │
     ▼  HTTPS
Floating IP (Neutron) ──NAT──► Neutron Router (qrouter namespace)
                                        │
                                        ▼
                              Security Groups (iptables/nftables)
                                        │
                                        ▼
                              Open vSwitch (VXLAN Overlay)
                              br-int → br-tun → tunnel
                                        │
                                        ▼
                              VM tap interface (eth0)
                                        │
                                        ▼
                                   Running VM
```

### Tenant Networking Concepts

|Concept             |Implementation                                      |
|--------------------|----------------------------------------------------|
|**Tenant Isolation**|Each project gets isolated virtual network(s)       |
|**VXLAN Overlay**   |Layer 2 over UDP, spans multiple compute hosts      |
|**Security Groups** |Stateful per-VM firewall via iptables/nftables      |
|**Floating IP**     |1:1 NAT, external IP maps to internal private IP    |
|**Open vSwitch**    |Programmed via OpenFlow, handles all local switching|

-----

## 💾 Storage Architecture

### Glance — Image Storage Backends

```
Glance API  ──────►  Ceph RBD       (recommended for scale)
            ──────►  Swift           (object storage)
            ──────►  NFS / Local FS  (simple deployments)
```

VM boot images are served from Glance. With Ceph backend, nova-compute can clone images directly via RBD, enabling **copy-on-write** boot — fast and storage-efficient.

### Cinder — Block Volume Storage Backends

```
Cinder API  ──────►  Ceph RBD       (distributed, preferred)
            ──────►  iSCSI / FC SAN  (enterprise SAN integration)
            ──────►  NFS Volume      (simple NAS deployments)
```

Cinder provides persistent block devices that outlive individual VMs. Key features: **live attach/detach**, **volume snapshots**, **online resize**, **multi-attach** (select backends).

-----

## 🔒 SSL / TLS Flow

```
User Browser
    │
    │  HTTPS (TLS 1.2/1.3)
    ▼
HAProxy VIP  ──[TLS termination]──►  OpenStack API endpoints (HTTP)
    │                                        │
    │                                        ▼
    │                                  Keystone token
    │                                        │
    │                                        ▼
    │                                 Nova / Neutron / Cinder
    │                                        │
    │                                        ▼
    │                                    RabbitMQ (AMQP)
    │                                        │
    └─────────────────────────────────►  Compute Node
```

**TLS termination** occurs at HAProxy. Certificates are typically issued by **FreeIPA** (Red Hat IdM) or an external PKI integrated via TripleO/Director. Internal control-plane communication may use mutual TLS depending on deployment profile.

-----

## ⚡ RHOSP vs OpenShift Comparison

|Dimension        |🔴 RHOSP (OpenStack)               |🟣 OpenShift (OCP)                  |
|-----------------|----------------------------------|-----------------------------------|
|**Platform Type**|Infrastructure-as-a-Service (IaaS)|Platform-as-a-Service (PaaS)       |
|**Workload Unit**|Virtual Machines (KVM)            |Containers / Pods (CRI-O)          |
|**Orchestration**|Nova + Placement API              |Kubernetes Scheduler               |
|**State Store**  |MariaDB Galera (per service)      |etcd (cluster-wide KV store)       |
|**Messaging**    |RabbitMQ (AMQP)                   |kube-apiserver watch loops         |
|**Networking**   |Neutron + OVS / OVN               |OVN-Kubernetes / CNI plugins       |
|**Storage**      |Cinder (block) + Glance (image)   |CSI drivers + PersistentVolume     |
|**Identity**     |Keystone (token-based)            |OAuth2 / Kubernetes RBAC           |
|**Use Case**     |Private cloud, NFV, VM workloads  |App platform, CI/CD, microservices |
|**Deployment**   |TripleO / Director / Ansible      |Installer + Cluster Operators (CVO)|

### ⚠️ Critical Difference: Data Store

```
RHOSP Flow:
  API Request → Keystone Token → Nova API → MariaDB (state) → RabbitMQ (async) → Compute Node

OpenShift Flow:
  API Request → kube-apiserver → etcd (state) → Controller reconcile loop → Worker Node
```

> 🔑 **RHOSP does NOT use etcd.** This is a common confusion. RHOSP uses **MariaDB Galera** as its primary datastore and **RabbitMQ** for async communication — a fundamentally different architecture from Kubernetes-based platforms.

-----

## 📚 Key Architectural Learnings

### 1. Distributed Cloud Operating System

RHOSP is not simply a hypervisor manager. Every service (Nova, Neutron, Cinder, Keystone, Glance) is an independently deployable distributed service with its own API, database, and message queue bindings.

### 2. Service Isolation via Owned Databases

Each OpenStack service owns its own database schema. Cross-service calls happen through **REST APIs** (synchronous) or **RabbitMQ messages** (asynchronous) — never via direct database queries.

### 3. HAProxy + RabbitMQ + MariaDB = Control Plane Backbone

- **HAProxy** → High availability VIP and TLS termination
- **MariaDB Galera** → Durable state for all services
- **RabbitMQ** → Async decoupling of API from execution

### 4. API + Async Messaging Dual Communication Pattern

User-facing operations are synchronous (REST APIs return immediately). Long-running execution (VM boot, volume attach, network provisioning) is driven asynchronously via RabbitMQ messages.

### 5. Modern RHOSP = Containers Running OpenStack, VMs Run By OpenStack

The control plane services run in **Podman containers**. User workloads run as **KVM virtual machines**. Two distinct container philosophies operating at two separate layers of the stack.

### 6. No etcd — Fundamentally Different from Kubernetes

RHOSP uses **MariaDB** (not etcd) as its state store and **RabbitMQ** (not API server watch loops) for internal communication. RHOSP and OpenShift are complementary platforms, not competitors.

-----

## 🏗️ Full Architecture Reference Diagram

```
                         ┌──────────────────────┐
                         │    USER / CLIENT      │
                         │  Horizon / API / CLI  │
                         └──────────┬───────────┘
                                    │ HTTPS
                         ┌──────────▼───────────┐
                         │   HAProxy VIP         │
                         │  TLS Termination + LB │
                         └──────────┬───────────┘
                                    │
              ┌─────────────────────▼──────────────────────┐
              │              CONTROL PLANE                  │
              │                                             │
              │  [Keystone] [Nova API] [Neutron] [Glance]  │
              │  [Cinder API] [Placement] [Horizon]         │
              │                                             │
              │  [MariaDB Galera] ←── All service state    │
              │  [RabbitMQ] ←── Async internal messaging   │
              └──────────────┬──────────────────────────────┘
                             │ AMQP (RabbitMQ)
              ┌──────────────▼──────────────────────────────┐
              │              COMPUTE PLANE                   │
              │                                             │
              │  nova-compute → libvirt → KVM/QEMU          │
              │  neutron-agent → Open vSwitch               │
              │  cinder-volume → iSCSI / Ceph               │
              │                                             │
              │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐          │
              │  │ VM  │ │ VM  │ │ VM  │ │ VM  │  ...     │
              │  └─────┘ └─────┘ └─────┘ └─────┘          │
              └─────────────────────────────────────────────┘
```

-----

## 🌍 Use Cases

RHOSP powers mission-critical infrastructure across:

- **Telecommunications** — NFV (Network Function Virtualization) for 4G/5G core network functions
- **Financial Services** — Private cloud with strict data sovereignty and compliance requirements
- **Enterprise Datacenters** — VM workloads with predictable performance and hardware control
- **Government / Defense** — Air-gapped private clouds with full infrastructure ownership
- **Research & HPC** — Large-scale compute clusters with GPU passthrough via PCI SR-IOV

-----

## 📄 License

This documentation is provided for educational and reference purposes.

-----

*Architecture Reference · Red Hat OpenStack Platform · KVM · Neutron · Nova · Keystone · Cinder · Glance · RabbitMQ · MariaDB · HAProxy*