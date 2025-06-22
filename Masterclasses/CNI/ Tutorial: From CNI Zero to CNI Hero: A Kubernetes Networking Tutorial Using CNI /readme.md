# Tutorial: From CNI Zero to CNI Hero: A Kubernetes Networking Tutorial Using CNI

Doug Smith & Tomofumi Hayashi, Red Hat

Doug and Tomo are here to show you just how CNI works from the ground level up. You’ll go from zero CNI knowledge to CNI hero after we walk you through all the layers of using CNI for the first time. How do you configure a CNI plugin? When does CNI execute? How do you build a CNI plugin? How do you debug one? We’ll answer all those questions, and more. In our hands-on tutorial we’ll give you a tour of all the moving parts when it comes to CNI, from how CNI is executed by the runtime, to how a CNI plugin operates. We’ll even walk through our opinionated boilerplate for writing a CNI plugin that interfaces with Kubernetes by accessing the Kubernetes API, from our experience developing CNI plugins, such as Multus and Whereabouts IPAM CNI. You’ll walk away with sample code and techniques so that you can not only better manage and understand how CNI is working behind the scenes, but also an ability to use it creatively to solve problems and create new functionality for your clusters.

---

Following notes has been generated from Chatgpt based on the transcription of the mastercall.

Enjoy the masterclass.

---

# Kubernetes CNI (Container Network Interface) - Lecture Notes

## Session Title: From CNI Zero to CNI Hero

### Instructors:

* **Tomo** – Red Hat, OpenShift, CNI/Multi-CNI maintainer
* **Doug Smith** – Network Plumbing WG, co-maintainer of Multi-CNI

---

## 1. Introduction to CNI

### What is CNI?

* CNI = Container Network Interface
* A specification that provides networking to containers
* Used by Kubernetes and other container orchestrators

### Core Responsibilities:

* Assign IP addresses to pods
* Set up virtual interfaces
* Apply optional network rules (e.g., iptables, MTU size)
* Tear down network setup when pods are deleted

### CNI Lifecycle:

* Called on **pod creation** and **pod deletion**
* Interacts with **container runtime** (e.g., containerd, CRI-O)
* Executes plugins defined in a JSON config file

---

## 2. CNI Architecture & Flow

### High-Level Flow (Pod creation):

1. User submits a pod via `kubectl`
2. Kubelet passes request to container runtime
3. Runtime triggers `libcni` to read CNI config
4. Config specifies which CNI plugins to invoke
5. Plugins set up network interfaces inside the pod's netns

### Components:

* `libcni`: Library used by runtimes to invoke plugins
* CNI plugin binaries: `/opt/cni/bin/`
* CNI config files: `/etc/cni/net.d/*.conf` or `*.conflist`

---

## 3. Pod and Linux Networking

### Pod != Container:

* A **Pod** can contain multiple containers
* Containers in the same pod share:

  * Network namespace
  * Loopback device
  * Assigned IP (via CNI)

### Namespaces used:

* **net**, **pid**, **mount** — isolated by Linux kernel

---

## 4. CNI Configuration Format

### Format:

* JSON file
* Preferred suffix: `.conflist`
* Kubernetes picks the **lexicographically first** file

### Sample Structure:

```json
{
  "cniVersion": "1.0.0",
  "name": "mynet",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni0",
      "ipam": {
        "type": "host-local",
        "subnet": "10.22.0.0/16"
      }
    }
  ]
}
```

### Key Fields:

* `type`: Name of the CNI binary to run
* `ipam`: IP Address Management plugin (e.g., `host-local`, `dhcp`)

---

## 5. Input & Output of CNI Plugins

### Inputs:

* **Environment Variables**: `CNI_COMMAND`, `CNI_CONTAINERID`, etc.
* **Standard Input**: JSON configuration file

### Outputs:

* **stdout**: JSON "result object" (interfaces, IPs, MACs)
* **stderr**: Error messages
* **Exit Code**:

  * `0` = Success
  * Non-zero = Failure

### Result Object Format:

```json
{
  "cniVersion": "1.0.0",
  "interfaces": [...],
  "ips": [...],
  "routes": [...]
}
```

---

## 6. Plugin Chaining (Multiplugin Setup)

* Allows execution of multiple plugins sequentially
* Example: `bridge` ➝ `tuning`
* Second plugin receives `prevResult` from the first
* Enables modular networking setup

---

## 7. Capabilities and Runtime Config

### Capabilities:

* Used for Kubernetes annotations (e.g., bandwidth, port mapping)
* Injected dynamically at runtime by kubelet

### Examples:

* `portMappings`, `bandwidth`
* Config updated before runtime executes plugin

---

## 8. Writing a Custom CNI Plugin

### Basic Requirements:

* Read environment variables
* Read stdin (CNI config)
* Create interface (e.g., using `netlink`)
* Output JSON result

### Language:

* Typically Go, but any language works

### Considerations:

* Version compatibility (`cniVersion`)
* `DEL` command must not fail (per spec)
* Plugins must handle multiple invocations

---

## 9. Hands-On Lab Summary

### Setup:

* Use **kind** to create local Kubernetes cluster
* Disable default CNI

### Tasks:

1. Install **Flannel** plugin manually
2. Troubleshoot plugin setup
3. Create **dummy** CNI plugin in bash
4. Observe behavior via `kubectl exec` and `describe`

### Observations:

* Dummy plugin reports fake IP — not reflected in real `ip a` output
* Flannel plugin creates real interface (e.g., `eth0`)

---

## 10. Debugging Tips

| Issue                | Action                                |
| -------------------- | ------------------------------------- |
| Pods stuck `Pending` | Check CNI config in `/etc/cni/net.d/` |
| Plugin missing       | Ensure binary is in `/opt/cni/bin/`   |
| Wrong plugin runs    | Check filename sort order             |
| Invalid plugin type  | Verify `type` matches binary name     |

---

## 11. Windows Considerations

* Core principles are same
* Paths and binaries differ
* Some plugins (e.g., `macvlan`) not supported on Windows
* Use Kubelet config to locate binary/config directories
* Scripts may need to be converted to PowerShell

---

## 12. Further Resources

* 📘 Spec Docs: `spec.md`, `conventions.md`
* 🌍 Official Site: [https://cni.dev](https://cni.dev)
* 🧪 Plugins: [https://github.com/containernetworking/plugins](https://github.com/containernetworking/plugins)
* 🎥 Community Talks: CNCF, OpenShift, SIG Network

---

## 13. Summary

* CNI is a core building block of Kubernetes networking
* Handles setup/teardown of pod-level networking
* Plugin-based architecture allows flexibility
* Learning CNI internals empowers you to debug and extend K8s networking

---

**Assignment:**

1. Install a Kind cluster and disable default CNI
2. Deploy a CNI plugin (Flannel or Calico)
3. Build a minimal dummy plugin that logs its inputs
4. Observe result in Kubernetes API vs actual pod interface

---
