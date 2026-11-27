# OKD with Ansible

Installs and configures OKD or OCP clusters on bare-metal home-lab
hardware from an Ansible inventory.

## Language

### Distributions

**Distribution**:
The product an Inventory installs: OKD or OCP.

**OKD**:
The community Distribution, running on SCOS.
_Avoid_: SCOS (for the product), OpenShift

**OCP**:
Red Hat's subscription Distribution, running on RHCOS and needing a
Red Hat pull secret.
_Avoid_: OpenShift, Red Hat OpenShift

### Hosts and naming

**Infra Host**:
The machine outside the cluster that serves the install-time
services: load balancing, ignition hosting and network boot. A
cluster without an external Load Balancer needs no Infra Host.
_Avoid_: lbs, infra node, RPI, load balancer (for the machine)

**Load Balancer**:
The HAProxy function on the Infra Host that spreads API and ingress
traffic across nodes.
_Avoid_: lbs

**Control Plane Node**:
A cluster node that runs the API server, etcd and the other
cluster-management components.
_Avoid_: master

**Inventory**:
A directory under `inventories/` describing one cluster's hosts and
settings. Selected with the `CLUSTER_NAME` make variable, which is a
misnomer.
_Avoid_: cluster (when meaning the inventory)

**Cluster Name**:
The DNS label that names the cluster, such as `okd4`. Several
Inventories may share one Cluster Name.

**Base Domain**:
The parent DNS domain the cluster lives under, such as `example.com`.

**Cluster Domain**:
The Cluster Name joined to the Base Domain, such as
`okd4.example.com`. Every node and cluster endpoint is named under it.
_Avoid_: full domain

### Installation

**Bootstrap Installation**:
An installation method where a temporary Bootstrap Node brings up the
control plane, and each node is installed from CoreOS media with an
ignition file served by the Infra Host.
_Avoid_: bootstrap flow, ignition installation, UPI

**Agent-Based Installation**:
An installation method where every node boots one generated ISO and
the nodes form the cluster among themselves, with no Bootstrap Node.
_Avoid_: agent install

**Bootstrap Node**:
The temporary machine in a Bootstrap Installation that brings up the
first control plane and is retired once it is running. The same
machine may afterwards be reinstalled as a worker.
_Avoid_: bootstrap machine

### Images

**Mirror Registry**:
A registry the cluster pulls images from in place of their source
registry.
_Avoid_: disconnected registry, custom container registry

**Pull-Through Cache**:
A Mirror Registry that fetches images from the source registry on
demand and keeps a copy.
_Avoid_: docker proxy

**Disconnected Installation**:
An installation whose nodes cannot reach the source registries, so
every image comes from a Mirror Registry.
_Avoid_: air-gapped
