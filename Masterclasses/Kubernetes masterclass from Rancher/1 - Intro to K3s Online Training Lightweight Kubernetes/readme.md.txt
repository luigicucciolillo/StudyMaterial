# Intro to K3s Online Training: Lightweight Kubernetes

Ready to get some training on using K3s, the lightweight Kubernetes distribution?

K3s is a lightweight implementation of Kubernetes that is easy to install and can run on x86 and ARM infrastructure with only 512 MB of RAM required to run it. 

K3s is geared towards teams that need to deploy applications quickly and reliably to resource-constrained environments. Some use cases for K3s are edge, Single Board Computers, IoT, and CI.

Watch the introductory training on K3s to learn how to:

Install and upgrade K3s
Deploy K3s on ARM
Manage multiple K3s clusters and deploy applications
Using K3d on the desktop
Using K3os to deliver a container operating system
Setup and configure a High Value install of K3s

---

link to master class [here](https://www.youtube.com/watch?v=vRjk3r9fwFo)

---

# from official [repo](https://github.com/k3s-io/k3s) - [website](https://k3s.io)

## K3s - Lightweight Kubernetes

Lightweight Kubernetes. Production ready, easy to install, half the memory, all in a binary less than 100 MB.

Great for:

- Edge
- IoT
- CI
- Development
- ARM
- Embedding k8s
- Situations where a PhD in k8s clusterology is infeasible

### What is this?

K3s is a fully conformant production-ready Kubernetes distribution with the following changes:

- It is packaged as a single binary.
- It adds support for sqlite3 as the default storage backend. Etcd3, MariaDB, MySQL, and Postgres are also supported.
- It wraps Kubernetes and other components in a single, simple launcher.
- It is secure by default with reasonable defaults for lightweight environments.
- It has minimal to no OS dependencies (just a sane kernel and cgroup mounts needed).
- It eliminates the need to expose a port on Kubernetes worker nodes for the kubelet API by exposing this API to the Kubernetes control plane nodes over a websocket tunnel.

K3s bundles the following technologies together into a single cohesive distribution:

- Containerd & runc
- Flannel for CNI
- CoreDNS
- Metrics Server
- Traefik for ingress
- Klipper-lb as an embedded service load balancer provider
- Kube-router netpol controller for network policy
- Helm-controller to allow for CRD-driven deployment of helm manifests
- Kine as a datastore shim that allows etcd to be replaced with other databases
- Local-path-provisioner for provisioning volumes using local storage
- Host utilities such as iptables/nftables, ebtables, ethtool, & socat

These technologies can be disabled or swapped out for technologies of your choice.

Additionally, K3s simplifies Kubernetes operations by maintaining functionality for:

- Managing the TLS certificates of Kubernetes components
- Managing the connection between worker and server nodes
- Auto-deploying Kubernetes resources from local manifests in realtime as they are changed.
- Managing an embedded etcd cluster

### What's with the name?

We wanted an installation of Kubernetes that was half the size in terms of memory footprint. Kubernetes is a 10 letter word stylized as k8s. So something half as big as Kubernetes would be a 5 letter word stylized as K3s. A '3' is also an '8' cut in half vertically. There is neither a long-form of K3s nor official pronunciation.

### Is this a fork?

No, it's a distribution. A fork implies continued divergence from the original. This is not K3s's goal or practice. K3s explicitly intends not to change any core Kubernetes functionality. We seek to remain as close to upstream Kubernetes as possible. However, we maintain a small set of patches (well under 1000 lines) important to K3s's use case and deployment model. We maintain patches for other components as well. When possible, we contribute these changes back to the upstream projects, for example, with SELinux support in containerd. This is a common practice amongst software distributions.

K3s is a distribution because it packages additional components and services necessary for a fully functional cluster that go beyond vanilla Kubernetes. These are opinionated choices on technologies for components like ingress, storage class, network policy, service load balancer, and even container runtime. These choices and technologies are touched on in more detail in the What is this? section.

### How is this lightweight or smaller than upstream Kubernetes?

There are two major ways that K3s is lighter weight than upstream Kubernetes:

- The memory footprint to run it is smaller
- The binary, which contains all the non-containerized components needed to run a cluster, is smaller

The memory footprint is reduced primarily by running many components inside of a single process. This eliminates significant overhead that would otherwise be duplicated for each component.

The binary is smaller by removing third-party storage drivers and cloud providers, explained in more detail below.

### Getting Started

[Quick Install](https://docs.k3s.io/quick-start)
[Achictecture](https://docs.k3s.io/architecture)
[FAQ](https://docs.k3s.io/faq)
[Contribute](https://github.com/k3s-io/k3s/blob/master/CONTRIBUTING.md)