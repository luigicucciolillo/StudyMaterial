# Seccomp and eBPF for Runtime Security in Kubernetes

## Abstract

Linux workloads and containers share a common kernel, making control over kernel-facing operations an important element of runtime security. Two mechanisms discussed in this paper are **seccomp (Secure Computing)** and **eBPF-based runtime enforcement**.

Seccomp restricts the system calls that a process is permitted to execute according to a predefined security profile. It provides a comparatively static security boundary that must be established before the protected process starts. eBPF, by contrast, allows restricted programs to execute inside the Linux kernel and can be used to observe and enforce runtime behavior dynamically.

Using Kubernetes as the primary environment, the presentation examines how seccomp profiles can be managed through the Security Profiles Operator and how eBPF-based enforcement can be implemented through Tetragon. Rather than treating the technologies as mutually exclusive, the speakers argue that their different properties make them complementary security mechanisms.

---

# 1. Introduction

Modern containerized applications frequently execute on shared Linux kernels. Although containers provide isolation mechanisms, applications running within containers still interact with the kernel through system calls and other kernel interfaces.

Seccomp was introduced as a mechanism for constraining this interaction. It predates modern container environments and was originally intended to restrict what applications running on a shared Linux system could ask the kernel to perform.

For example, an application may legitimately need system calls related to opening files or creating sockets while having no legitimate need to invoke operations such as system shutdown or reboot. Seccomp provides a mechanism for defining which system calls should be available to a process.

The presentation positions seccomp as one of the early Linux sandboxing mechanisms: its purpose is to constrain an application to a defined set of operations against shared system resources.

---

# 2. Seccomp: Secure Computing

## 2.1 Basic Model

Seccomp operates through a **security profile** associated with a process.

The profile defines the set of system calls that the process is permitted to execute. Importantly, the profile must exist before the process is started.

Once a process enters the protected mode defined by the seccomp profile, the permitted system-call interface is restricted according to that configuration.

Depending on the configured behavior, seccomp can be used to:

- observe or record particular system calls;
- restrict a process to an approved set of system calls;
- combine monitoring and enforcement;
- reject disallowed system calls;
- terminate a process that attempts a prohibited operation.

The central security concept is therefore:

> A process receives only the kernel system-call interface that it is expected to require.

System calls outside that approved interface can generate events or trigger enforcement actions.

---

# 3. The Static Nature of Seccomp Profiles

A significant operational characteristic of seccomp is that the required policy must be known before the process starts.

If the permitted system-call list changes, the process generally needs to be associated with a new profile and restarted under that profile.

The workflow described in the presentation is therefore approximately:

1. determine the system calls required by the application;
2. create the seccomp profile;
3. place the profile where the runtime can access it;
4. start the process using the profile;
5. if the profile changes, stop the process;
6. update or replace the profile;
7. restart the process under the new configuration.

This model reflects the environment in which seccomp was originally created. However, it becomes operationally significant in dynamic container and Kubernetes environments, where applications and workloads are frequently created, replaced, and updated.

---

# 4. Managing Seccomp in Kubernetes

The presentation identifies the **Security Profiles Operator** as an important tool for using technologies such as seccomp, AppArmor, and SELinux in Kubernetes.

One of the practical challenges of seccomp is ensuring that profiles are generated and distributed to the appropriate nodes before containers requiring those profiles are started.

The Security Profiles Operator helps address this problem by managing the distribution of security profiles.

It can also assist with **profile discovery**. Instead of requiring an operator to know every required system call in advance, workload behavior can be observed to determine which system calls normally occur.

This provides a learning workflow:

**Observe workload → Record system calls → Construct profile → Enforce profile**

The workload can be observed for a period considered representative enough to determine the required system calls before the resulting profile is used for enforcement. 

---

# 5. eBPF

## 5.1 Extending Kernel Behavior

eBPF provides a mechanism for loading restricted programs into the Linux kernel.

The presentation describes these programs as capable of:

- extracting information;
- executing logic;
- making decisions based on observed events.

The programs operate under safety restrictions. They must not be capable of crashing the kernel and cannot execute indefinitely.

eBPF programs can attach to several areas of Linux kernel activity, including:

- networking;
- socket operations;
- file access;
- file-descriptor operations;
- system-call execution.

This ability to attach logic to kernel events provides the basis for both runtime observability and runtime enforcement.

---

# 6. Dynamic Enforcement with eBPF

One of the primary advantages attributed to eBPF in the presentation is its **dynamic nature**.

An eBPF program or policy can be modified while workloads remain operational.

For example, an organization may decide that an additional sensitive system call should be blocked across multiple clusters or namespaces. An eBPF-based mechanism can update the enforcement behavior without necessarily requiring:

- a Kubernetes node restart;
- a management workload restart;
- an application restart.

This capability is particularly relevant to distributed environments containing multiple clusters, namespaces, or cloud providers.

The speakers also characterize eBPF as minimally invasive because much of the relevant processing can occur directly inside the kernel rather than repeatedly transferring events to user space.

---

# 7. Runtime Questions Addressable Through eBPF

The presentation describes eBPF runtime visibility through the kinds of operational and security questions it can answer.

Examples include:

### Process execution

- Which binaries have executed?
- Which binaries are currently executing?
- Which versions of binaries or libraries are present?

### Vulnerability-related observation

- Is a potentially vulnerable library version active inside the environment?

### Network behavior

- Which network connections have workloads created?
- Has a workload unexpectedly begun communicating with another destination?

### File activity

- Have sensitive files been accessed by a workload?

### System-call activity

- Which system calls has a workload executed?
- Has a sensitive system call been invoked?
- Does system-call activity indicate potentially dangerous behavior?

The common theme is that eBPF can expose runtime behavior from multiple kernel layers rather than looking only at one interface.

---

# 8. Tetragon

The eBPF-based implementation discussed in the presentation is **Tetragon**.

Tetragon is described as an eBPF-based system providing:

- runtime security;
- observability;
- enforcement.

In Kubernetes, Tetragon can operate as an agent deployed across cluster nodes. The presentation describes deployment through a DaemonSet. On Linux virtual machines outside Kubernetes, an equivalent agent can operate as a system-managed binary.

Using eBPF, Tetragon can observe multiple categories of kernel activity, including:

- data and file access;
- Linux namespaces;
- process capabilities;
- process execution;
- system activity;
- network connectivity.

The system therefore extends beyond observing system calls alone and can associate kernel events with additional runtime context.

---

# 9. Container Runtimes Complicate Seccomp Policies

An important observation made during the presentation concerns the relationship between seccomp and container runtimes.

A seccomp policy applied to a container must account not only for the application's system calls but also for system calls required by the container runtime.

The speakers give an example in which different runtimes require different system-call sets. In their experiment:

- `crun` used 65 system calls for runtime-related operations;
- `runc` used 78.

Some system calls required by container runtime operation may also be considered security-sensitive.

Consequently, the policy cannot always simply prohibit every system call considered dangerous. Some calls may need to remain available because the runtime itself depends on them.

This introduces an important distinction between:

**required by the runtime**

and

**expected from the application**.



---

# 10. Combining Seccomp and eBPF

The presentation explicitly rejects the idea that seccomp and eBPF must be treated as competing alternatives.

A system may use both.

Consider a system call that must remain permitted because the container runtime requires it. Seccomp cannot simply remove that system call from the profile without affecting runtime operation.

However, an eBPF-based runtime security mechanism can observe whether the **application itself** attempts to execute the same operation.

This enables a layered policy:

**Seccomp**

Allows the system-call set necessary for the application and container runtime.

**eBPF/Tetragon**

Observes or blocks sensitive runtime behavior in a more context-aware manner.

For example, the system might allow a system call required by the runtime while generating an alert if an application process unexpectedly invokes it.

The speakers identify this combination as one of the major strengths of using the two technologies together.

---

# 11. Declarative Enforcement Policies in Tetragon

The presentation describes Tetragon enforcement using declarative policies represented as Kubernetes custom resources.

A policy can identify:

- the Kubernetes namespace to which it applies;
- workloads using pod-label selectors;
- sensitive system calls;
- the required action.

Actions can include:

- generating an alert;
- blocking the system call.

The policy can therefore conceptually express:

> For workloads matching this Kubernetes identity, take this action when one of these sensitive kernel operations occurs.

A major difference compared with seccomp is that the policy can be modified after a pod has already started.

The user operates against a high-level declarative policy while the underlying eBPF implementation can evolve independently.

---

# 12. Context-Rich Runtime Events

Tetragon events contain contextual information associated with the observed behavior.

The example described in the presentation associates a sensitive system-call event with information such as:

- Kubernetes namespace;
- pod;
- process;
- Linux capabilities;
- Linux namespaces;
- process credentials;
- attempted system call.

This differs from the more limited information available through traditional seccomp events.

The speakers later characterize this difference approximately as:

**Seccomp event**

> A particular PID attempted a particular system call.

**Tetragon event**

> A particular pod, on a particular node, at a particular time, performed the operation, together with additional process and Kubernetes metadata.

This richer runtime context can simplify investigation and policy analysis. 

---

# 13. Enforcement Must Occur in the Kernel

The presentation distinguishes **observability** from **enforcement**.

For observability, some event filtering may occur outside the kernel.

For enforcement, however, user-space filtering is not sufficient because the security decision must occur in line with the operation being controlled.

Therefore, the matching logic required to decide whether a policy applies must ultimately be available inside the kernel.

This creates a mapping problem:

**Kubernetes identity → Linux process → cgroup → eBPF policy**

Tetragon resolves this by associating Kubernetes workloads with Linux cgroups.

---

# 14. Mapping Kubernetes Workloads to eBPF Policies

Containers are organized through Linux cgroups.

The hierarchy described in the talk contains Kubernetes pods and the containers belonging to those pods beneath the Linux cgroup hierarchy.

Tetragon maintains BPF maps that associate policies with the cgroups corresponding to workloads.

Conceptually:

```text
Kubernetes Policy
        ↓
Namespace / Pod Selector
        ↓
Tetragon
        ↓
Linux cgroup
        ↓
BPF Map
        ↓
eBPF Program
        ↓
Process
```

When a policy is applied, Tetragon determines which Kubernetes workloads match it and associates those workloads with the corresponding cgroups.

The eBPF programs can then determine whether a particular policy applies to the process currently executing an operation.

When pods or labels change, Tetragon receives updates and adjusts its mappings accordingly.

---

# 15. The Policy Activation Race Window

Dynamic policy application introduces another security consideration.

A container may begin starting before Tetragon has received the Kubernetes API notification and completed the necessary BPF policy configuration.

This creates a possible interval between:

```text
Container creation
        ↓
Container starts
        ↓
Tetragon receives update
        ↓
BPF enforcement becomes active
```

During that interval, the intended policy might not yet be active.

The presentation describes an additional runtime-hook mechanism to address this situation.

When a new container is about to start, a runtime hook communicates with the Tetragon agent. The container is permitted to continue starting only after the relevant policy state has been configured.

The resulting flow becomes:

```text
Container runtime
        ↓
Runtime hook
        ↓
Tetragon agent
        ↓
Policy / BPF map configured
        ↓
Acknowledgement
        ↓
Container starts
```

If the Tetragon agent is unavailable and policy enforcement cannot be guaranteed, the described configuration can prevent the container from starting.

Trusted namespaces or workloads can be configured differently so that essential services remain able to start.

---

# 16. Demonstration

The presentation includes a demonstration using a Kubernetes environment running on AWS.

The described environment contained ten nodes, with Tetragon deployed on each node using a DaemonSet.

A Kubernetes namespace was designated as sensitive, and an NGINX pod was placed inside it.

A Tetragon policy was then applied to that namespace to restrict a collection of sensitive system calls.

The policy was configured both to:

1. block the selected operations; and
2. generate events when an attempt occurred.



---

# 17. Demonstrated Enforcement

After the policy was installed, commands were executed inside the NGINX pod that attempted operations associated with sensitive system calls.

The demonstration included attempts involving operations such as:

- changing the root filesystem;
- mounting a filesystem;
- performing a pivot-root operation.

The attempted actions failed with permission-related errors.

At the same time, Tetragon produced enforcement events identifying the workload, process, namespace, attempted operation, and enforcement result.

The demonstration therefore showed both sides of runtime security:

```text
Sensitive operation
        ↓
eBPF policy match
        ↓
Operation blocked
        +
Security event generated
```



---

# 18. Seccomp and eBPF Comparison

The speakers conclude by comparing the two mechanisms.

| Characteristic | Seccomp | eBPF / Tetragon |
|---|---|---|
| Primary model | System-call restriction | Runtime observation and enforcement |
| Policy timing | Defined before process start | Can be changed dynamically |
| Restart required for policy changes | Generally yes | Not necessarily |
| System-call restriction | Yes | Yes, in the presented implementation |
| Kubernetes metadata awareness | Limited | Rich Kubernetes context |
| Policy selectors | Process/profile based | Namespace, labels, workload identity and other runtime attributes |
| Runtime observability | More limited | Broad |
| Kernel requirements | Mature seccomp support | Depends on required eBPF features |
| Deployment consideration | Profile must exist before process start | Runtime agent and eBPF environment required |

The table summarizes only distinctions made during the presentation.

---

# 19. Advantages of eBPF-Based Enforcement

According to the presentation, key advantages include:

### Dynamic policies

Policies can be changed without restarting the protected Kubernetes application or rebooting the underlying node.

### Flexible identity

Policies can use Kubernetes-related metadata such as:

- namespaces;
- pod labels.

The speakers also mention the possibility of using information such as:

- binary hashes;
- Linux capabilities;
- Linux namespaces;
- Unix credentials.

### Rich observability

Runtime events can contain substantially more workload context than a basic system-call event.

### Broader security visibility

eBPF can attach to several kernel layers rather than being limited to the system-call filtering model.



---

# 20. Limitations of eBPF-Based Enforcement

The presentation also identifies requirements and limitations.

The appropriate BTF metadata must be available on the node for the discussed eBPF functionality.

More advanced eBPF features may also require newer Linux kernels.

Consequently, the available enforcement mechanisms can depend on the kernel version deployed across the infrastructure.

---

# 21. Advantages of Seccomp

Seccomp provides an important property that differs from dynamically installed eBPF enforcement:

**the process begins execution with its restrictions already active.**

Because the profile is associated with the process at startup, there is no equivalent period in which the process has started while waiting for the seccomp profile to become active.

The presentation also emphasizes the maturity of tooling surrounding seccomp profile discovery and management.

Tools can observe application behavior and record the system calls required during execution. This allows operators to build application-specific allowlists from observed runtime behavior.

---

# 22. Limitations of Seccomp

The most significant operational limitation discussed is policy iteration.

Changing a seccomp policy generally requires restarting the protected process. In production environments, frequent application restarts solely for security policy changes may be undesirable.

A second limitation arises from the container runtime itself.

The seccomp profile must permit system calls needed not only by the application but also by the underlying runtime.

A third limitation is contextual visibility.

Seccomp was created before the modern container orchestration model. Consequently, its events do not naturally contain the same level of Kubernetes metadata exposed by systems such as Tetragon.

---

# 23. Relationship with SELinux and AppArmor

During the question-and-answer section, the speakers were asked whether Tetragon eliminates the need for SELinux or AppArmor.

The answer presented these mechanisms as addressing different areas of host security.

Seccomp focuses particularly on controlling the interface between processes and the kernel through system calls.

SELinux and AppArmor can extend protection to other shared host resources, including filesystem access.

The discussion therefore advocates a layered approach rather than treating the technologies as strict replacements for one another.

The speakers also note that eBPF enforcement can address areas beyond system calls, including filesystem-related activity, depending on the policy and implementation.

---

# 24. Scope of Seccomp Profiles

Another question concerned the granularity of seccomp profiles.

The presentation states that profiles can effectively be applied per process and therefore can be defined independently for different containers.

If a pod contains multiple containers, separate profiles can be associated with individual containers rather than requiring one profile to apply to the entire host or pod.

---

# 25. Cluster-Wide Tetragon Policy Distribution

For Kubernetes clusters containing many nodes, Tetragon does not require policies to be manually installed node by node.

The presentation describes an operator responsible for the custom-resource management.

When a policy is applied through Kubernetes, the Tetragon components distribute or update the relevant policy state on the appropriate nodes.

This maintains the Kubernetes declarative model:

```text
kubectl apply
      ↓
Kubernetes API
      ↓
Tetragon Operator / Agent
      ↓
Relevant Nodes
      ↓
eBPF Enforcement
```



---

# 26. Observability and Enforcement Can Coexist

The final technical discussion concerns the distinction between processing events externally and enforcing operations inside the kernel.

The two approaches are not mutually exclusive.

A system can collect runtime events and send them for external analysis while simultaneously applying selected enforcement decisions inside the kernel.

This provides two complementary functions:

```text
Kernel event
   │
   ├──→ Observability → External processing / analysis
   │
   └──→ Enforcement → Immediate in-kernel decision
```

For security enforcement, the actual blocking decision must occur where the operation can still be prevented.

For broader observability, events can continue to be collected and correlated externally.

---

# 27. Architectural Interpretation

The presentation can be summarized as describing three security layers.

## Layer 1 — Static system-call boundary

**Seccomp**

```text
Application
     ↓
Allowed syscall?
   ↙      ↘
 Yes      No
  ↓        ↓
Kernel   Block / Event
```

Its strength is that the restriction exists from process startup.

---

## Layer 2 — Dynamic kernel-aware runtime policy

**eBPF / Tetragon**

```text
Application behavior
        ↓
Kernel event
        ↓
eBPF program
        ↓
Runtime identity + policy
        ↓
Observe / Alert / Block
```

Its strength is dynamic, context-aware enforcement.

---

## Layer 3 — Broader host access control

**SELinux / AppArmor**

These mechanisms address areas of host security beyond the system-call interface, including access to resources such as filesystems.

The speakers therefore present these technologies as overlapping layers of defense rather than strict substitutes.

---

# 28. Conclusion

The central argument of the presentation is not that seccomp should be replaced by eBPF, nor that eBPF makes traditional Linux security mechanisms obsolete.

Instead, the technologies provide different security properties.

Seccomp establishes a restricted system-call interface before a process begins execution. This provides a strong and predictable initial boundary but makes subsequent policy modification comparatively static.

eBPF enables security logic to observe and react to runtime behavior directly inside the Linux kernel. Tetragon builds on this capability to provide Kubernetes-aware observability and enforcement using workload identity and runtime context.

Together, the mechanisms provide complementary capabilities:

```text
Seccomp
   ↓
Reduce available system-call surface
   +
eBPF / Tetragon
   ↓
Observe and dynamically control runtime behavior
   +
SELinux / AppArmor
   ↓
Protect additional host resources
```

The resulting model is one of **layered runtime security**: static restrictions can minimize the accessible kernel interface while dynamic eBPF policies provide contextual monitoring and enforcement for behavior that cannot be adequately expressed through a fixed system-call allowlist alone.