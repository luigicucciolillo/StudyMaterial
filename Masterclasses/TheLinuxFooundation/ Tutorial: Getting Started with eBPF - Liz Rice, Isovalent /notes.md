# Getting Started with eBPF
## Lesson-style notebook based on Liz Rice's tutorial transcript

1h 30min video conference - 25 may 2023 - link to video  [here](https://www.youtube.com/watch?v=TJgxjVTZtfw)


> **Source fidelity note**  
> This notebook is derived **only** from the attached tutorial transcript, treated as the authoritative reference. The material has been reorganized into a lesson format and obvious speech-to-text spellings have been normalized for readability (for example: *eBPF*, *kernel*, *Cilium*, *SELinux*). No external technical material has been added.

### Learning objectives
By the end of the lesson, you should be able to explain:

- what eBPF is and why dynamically extending kernel behavior matters;
- the relationship between user space, system calls, the kernel, and eBPF programs;
- how eBPF programs attach to events and receive contextual information;
- why eBPF maps are important for sharing information;
- what BCC and `bpftool` contribute to the development workflow;
- how eBPF bytecode and the eBPF virtual machine fit into execution;
- how eBPF is used in networking, especially with XDP;
- what the verifier checks and why it is central to eBPF safety;
- how eBPF programs are typically managed in production systems.

---

### Suggested study method
Read each lesson section, stop at the **Check your understanding** prompt, and answer it before expanding your notes or moving on.

# 1. What is eBPF?

The tutorial introduces eBPF as an ability to run **custom programs within the kernel**. The key idea is that these programs can change or influence kernel behavior **without rebuilding the kernel and without rebooting the machine**.

The acronym originally expands to **extended Berkeley Packet Filter**, but the tutorial emphasizes that the name is no longer a useful description of the technology because eBPF does much more than packet filtering.

### Why this matters
The kernel participates in almost every interesting interaction with hardware or system resources. Examples given in the tutorial include:

- reading or writing files;
- reading from the network;
- allocating memory;
- actions initiated through system calls.

Changing normal kernel code is difficult: a change must be accepted into the kernel, distributed, and eventually deployed on machines that may be running kernels several years old. eBPF changes this model by allowing a program to be loaded dynamically.

**Core idea:**

```text
Traditional kernel change
    source change -> kernel patch -> release -> deployment -> reboot / adoption delay

With eBPF
    write program -> load program -> attach to event -> behavior changes dynamically
```

> **Source window:** ~03:27–06:21

### Check your understanding
1. Why does the tutorial call eBPF a “game-changing” technology?
2. Why can normal kernel evolution take a long time to reach production systems?
3. What operational step can eBPF avoid when changing behavior on a running machine?

# 2. User space, system calls, and kernel events

Applications normally execute in **user space**. When they need the kernel to perform an operation, they make **system calls**. Application developers often interact with higher-level APIs, while the underlying system calls are abstracted away.

The tutorial's model is:

```text
+------------------------------+
|          User space          |
| application / process        |
+---------------+--------------+
                |
                | system call
                v
+------------------------------+
|            Kernel            |
| files | memory | networking  |
| process / security behavior  |
+------------------------------+
```

An eBPF program can be attached to many kinds of events. Examples mentioned in the tutorial include:

- kernel functions (**kprobes**);
- user-space functions (**uprobes**);
- tracepoints;
- network packets at different locations in the stack;
- Linux Security Module hooks;
- performance events.

When the selected event occurs, the attached eBPF program runs.

> **Source window:** ~06:29–09:00

### Important consequence
The type of event affects what the program can sensibly do and what context is available to it.

### Check your understanding
Suppose two eBPF programs are attached to different kinds of events. Why should you not assume that they receive the same contextual information?

# 3. Why eBPF is especially interesting in Kubernetes

The tutorial connects eBPF to Kubernetes through a simple observation:

> Many containers can run on one node, but they share the **same host kernel**.

This means that applications in many pods eventually interact with the same kernel when they perform networking, file operations, privilege-related actions, and other system activities.

```text
Pod A ----\
Pod B -----+----> shared node kernel
Pod C ----/          |
                      +--> files
                      +--> networking
                      +--> process / privilege operations
```

If the kernel is instrumented, eBPF can provide visibility into activity across those workloads. The tutorial also notes that eBPF can potentially influence behavior—for example, by preventing an operation for security reasons or changing how network packets are handled.

Each Kubernetes **node** has its own kernel, so eBPF instrumentation is ultimately applied per node.

> **Source window:** ~09:03–10:34 and ~25:35–27:06

### Check your understanding
Why can kernel-level instrumentation observe activity from many containers without modifying every individual application?

# 4. First program: “Hello World” and event context

The tutorial's first hands-on example is a very small eBPF program that emits tracing information whenever it is triggered. In the example, the trigger is associated with the `execve` system call.

The point of the exercise is not the tracing output itself. It demonstrates this execution model:

```text
Event occurs
    |
    v
attached eBPF program executes
    |
    v
program can use event/context information
```

## Context
An eBPF program receives a context parameter whose meaning depends on the program type and attachment point.

The tutorial gives examples of contextual information such as:

- the user-space process that caused an event;
- executable information;
- process ID;
- timestamps;
- network-packet-related metadata for networking programs.

As programs become more advanced, understanding kernel data structures and the event context becomes increasingly important.

> **Source window:** ~10:41–18:42

### Professor's note
The tutorial explicitly says that basic exercises can be simple, but more advanced eBPF work can become kernel programming quite quickly. Its advantage is not that kernel concepts disappear, but that eBPF provides a safer and more structured execution environment than writing arbitrary kernel modules.

### Check your understanding
What determines the structure and meaning of an eBPF program's context parameter?

# 5. eBPF maps: sharing data between kernel and user space

The tutorial describes simple trace output as useful for demonstration but not as a scalable way to exchange information. Real eBPF systems may run many programs, so a shared tracing stream quickly becomes inconvenient.

The alternative introduced is an **eBPF map**.

A map is a data structure that can be accessed from eBPF programs and, depending on the map type and use case, from user space as well.

### Roles described in the tutorial
Maps can be used to:

- send information from kernel space to user space;
- provide information from user space to eBPF programs;
- share information between multiple eBPF programs;
- store key/value data;
- support efficient event transfer with map types intended for that purpose.

```text
          +--------------------+
          |   eBPF program A   |
          +---------+----------+
                    |
                    v
              +-----------+
              | eBPF map  |
              +-----------+
               ^         |
               |         v
+--------------+--+   +--+----------------+
| eBPF program B |   | user-space program |
+----------------+   +---------------------+
```

The tutorial also stresses a performance pattern: do useful filtering in the kernel and send only necessary information to user space, reducing repeated kernel/user-space transitions.

> **Source window:** ~15:20–16:18 and ~18:44–20:40

### Check your understanding
Why can maps be preferable to repeatedly transferring every raw event directly into user space?

# 6. BCC: making early eBPF experiments easier

The early tutorial examples use **BCC**. The transcript describes BCC as a Python library/framework that performs substantial “heavy lifting”:

1. it takes eBPF C code;
2. invokes the compiler;
3. loads the resulting program into the kernel;
4. attaches it to an event;
5. lets Python handle user-space logic around the program.

BCC is presented as a useful beginner tool and a convenient way to write simple instrumentation scripts.

## Portability concern
The tutorial explains why compiling on the target machine was historically useful: kernel data structures can differ between kernel versions. Compiling against the machine where the program will run gives the compiler information that matches that kernel.

The tutorial then introduces **Compile Once – Run Everywhere (CO-RE)** as a way to improve portability across kernel versions, while noting that it is a more advanced topic and is not covered deeply in the hands-on session.

> **Source window:** ~22:21–25:32

### Check your understanding
Why can compiling an eBPF program on one machine and moving it to another machine with a different kernel create compatibility concerns?

# 7. From C source to eBPF bytecode

Under the abstraction provided by BCC, the C program is compiled into an object containing **eBPF bytecode**.

The tutorial presents eBPF as a software virtual machine with its own instruction set and general-purpose registers. The object file contains the bytecode instructions that make up the program and information about maps used by the program.

A simplified lifecycle from the transcript is:

```text
C / supported source language
        |
        v
compiler
        |
        v
object file with eBPF bytecode + map definitions
        |
        v
BPF system call
        |
        v
verifier
        |
        v
program loaded into kernel
        |
        v
program attached to an event
```

The important point is that the verifier operates on the **bytecode**, not on the original C source.

> **Source window:** ~28:45–31:32 and ~36:39–39:32

### Check your understanding
If an error message from the verifier refers to bytecode-level behavior, why might the corresponding problem be less obvious when you look only at the original source line?

# 8. `bpftool`: a low-level eBPF toolbox

The tutorial introduces **`bpftool`** as a “Swiss army knife” for eBPF programs and maps.

Functions attributed to it in the transcript include:

- loading programs into the kernel;
- inspecting eBPF programs currently present on the system;
- reading map contents;
- manipulating maps;
- exposing low-level information such as bytecode.

The lesson positions `bpftool` below a full production system: it is valuable for inspection and manipulation, while production eBPF applications generally add their own user-space management layer.

> **Source window:** ~31:34–33:10

### Study prompt
How is `bpftool` different in role from BCC in the tutorial?

# 9. eBPF and networking

Networking is one of the tutorial's major examples because eBPF programs can attach at several points in the network stack.

## Example: mitigating a “packet of death”
The tutorial describes a class of kernel vulnerability in which a specially crafted packet could trigger faulty packet processing and crash the kernel.

Without eBPF, mitigating such a kernel vulnerability could require installing a new kernel and rebooting machines. With eBPF, a program could instead be loaded dynamically to inspect packets and drop the malicious form before vulnerable processing occurs.

This example illustrates why eBPF can be useful for **dynamic mitigation**.

The transcript also notes that an eBPF networking program can do more than drop traffic. Depending on the attachment point and program, it can inspect, modify, redirect, or pass packets.

> **Source window:** ~39:55–43:01

### Check your understanding
What operational advantage does the tutorial's packet-mitigation example demonstrate?

# 10. Container networking and Cilium

The tutorial uses Cilium to illustrate how eBPF can improve container networking.

A container or Kubernetes pod normally has its own network namespace. The host and pod networking namespaces are commonly connected through a virtual Ethernet link. A packet may therefore traverse several parts of the host networking stack before reaching the pod.

```text
physical NIC
    |
    v
host network stack
    |
    v
virtual Ethernet connection
    |
    v
pod network namespace
```

The tutorial contrasts this with an eBPF-based approach that can intercept packets and make forwarding decisions more directly.

It also highlights a Kubernetes-specific issue: pod IP addresses can change dynamically. Cilium can operate with Kubernetes-aware identity information rather than treating changing IP addresses as the only meaningful identity.

> **Source window:** ~43:05–46:07 and ~26:19–27:17

### Check your understanding
Why can frequent pod creation and changing IP addresses make traditional rule management noisy in Kubernetes environments?

# 11. XDP: an early networking hook

The networking lab focuses on **XDP**, expanded in the tutorial as **Express Data Path**.

The tutorial places the XDP hook very early in packet reception: conceptually, a packet reaches a physical network interface and can encounter the XDP program before normal kernel network-stack processing has progressed very far.

This early position is useful for high-performance decisions.

### Actions highlighted in the lab
The tutorial focuses on two return outcomes:

- `XDP_PASS` — continue normal processing;
- `XDP_DROP` — discard the packet.

The exercise uses ping/ICMP traffic to demonstrate dynamically changing whether packets are passed or dropped.

The transcript also describes **XDP offload**, where some network devices can execute this processing even earlier, potentially avoiding CPU/kernel handling for packets that can be decided on the device.

> **Source window:** ~46:10–49:26

### Mental model

```text
incoming packet
      |
      v
   [ XDP ] ---- XDP_DROP ---> discarded
      |
   XDP_PASS
      |
      v
normal kernel networking
```

### Check your understanding
Why is an early packet-processing hook attractive for filtering and other high-performance networking tasks?

# 12. The eBPF verifier

The verifier is one of the central safety mechanisms described in the tutorial.

It examines the **eBPF bytecode** before the program is allowed to run. The transcript says it analyzes possible execution paths and keeps track of possible values in registers to assess whether the program is safe.

## Examples of checks described

### 12.1 Pointer safety
A program must explicitly establish that a pointer is valid before dereferencing it. C code might compile successfully, yet the verifier can still reject the resulting eBPF program.

### 12.2 Completion / complexity
The verifier checks that a program is expected to run to completion. The tutorial discusses a complexity limit used during analysis, and notes that this limit had grown substantially compared with older kernels.

### 12.3 Appropriate helper functions
The available helper functions depend on program type and context. A networking-specific helper would not make sense for a program attached to an unrelated kernel event, and a process-oriented helper may not make sense while handling a packet where no user-space process is the relevant context.

```text
source compiles
      |
      v
bytecode generated
      |
      v
+-----------------+
|    verifier     |
| pointer safety  |
| execution paths|
| helper validity |
+-----------------+
      |
 pass | reject
      v
load into kernel
```

> **Source window:** ~56:57–61:23

### Important nuance from the tutorial
Verifier messages can be difficult to interpret because the verifier reasons about bytecode and tracked register state. The line where a failure becomes visible may not be the source line where the problematic value originated.

### Check your understanding
Why is “valid C” not sufficient to guarantee that an eBPF program will be accepted?

# 13. eBPF privileges and enterprise trust

The tutorial makes an important operational point: the ability to load eBPF programs is extremely powerful and should not be granted casually.

The verifier helps prevent unsafe execution patterns, but that does **not** mean every verified program is automatically desirable or trustworthy. A program can intentionally perform powerful operations such as intercepting or changing network traffic.

The transcript therefore highlights questions of:

- who is allowed to load programs;
- where programs came from;
- whether the vendor or source is trusted;
- how program provenance might be validated.

The speaker also explains why signing eBPF programs is complicated by code relocation and portability mechanisms, and states that this was an area of ongoing work rather than a completely solved problem in the context of the tutorial.

> **Source window:** ~51:01–54:33

### Check your understanding
What is the difference between a verifier proving that a program is safe to execute and an organization deciding that the program is trustworthy to deploy?

# 14. Getting data into real user-space systems

The terminal output in a tutorial is only one possible consumer of eBPF data.

Once information has been moved into user space—for example through a map—the user-space application can process it however it wants. The tutorial explicitly gives the example of sending information onward to a system such as **Prometheus** instead of simply printing it.

```text
eBPF program
    |
    v
eBPF map / event channel
    |
    v
user-space agent
    |
    +--> terminal
    +--> metrics / monitoring pipeline
    +--> application-specific processing
```

> **Source window:** ~64:37–65:36

### Check your understanding
Which part of the system is responsible for turning low-level eBPF data into dashboards, metrics, or other application-level output?

# 15. Managing eBPF programs in production

A production eBPF application may require multiple kernel programs rather than one.

The tutorial gives a system-call example:

- one eBPF program can attach at **system-call entry** to capture arguments;
- another can attach at **system-call exit** to capture results;
- maps can be used to coordinate information between them.

More sophisticated projects can use many programs.

## User-space agents
The tutorial explains that production-ready eBPF systems commonly include a user-space agent that manages the lifecycle of kernel programs.

For the Cilium example in Kubernetes, that agent is described as running once per node (for example, through a DaemonSet) and taking care of loading and updating eBPF programs.

A notable design property discussed is that if the agent stops, the already-loaded kernel programs and maps can remain in place, preserving the data plane even though it may stop adapting to new changes until the agent returns.

> **Source window:** ~65:39–69:30

### Check your understanding
Why is a user-space lifecycle manager useful even though `bpftool` can load and inspect programs manually?

# 16. There is no single universal eBPF application architecture

The tutorial repeatedly emphasizes that design choices depend heavily on the application.

Examples include:

- how multiple programs coordinate;
- how events are transported from kernel to user space;
- which map types are used;
- how programs are deployed and managed;
- how data from many nodes is correlated.

The transcript notes that eBPF continues to evolve, and that newer mechanisms can replace older patterns as kernels gain more efficient capabilities.

> **Source window:** ~69:37–71:15

### Professor's takeaway
Learn the stable mental model first:

```text
hook/event -> eBPF program -> maps/helpers -> user-space management/consumer
```

Then treat the exact implementation pattern as application- and kernel-version-dependent.

# 17. Development workflow and language choices

The tutorial describes the eBPF development ecosystem as still developing.

It mentions work around:

- code coverage for eBPF programs;
- CI/CD approaches;
- testing across environments and kernels;
- studying mature eBPF projects as examples of engineering practice.

Projects mentioned as places to learn from include Cilium, Falco, and Meta's Katran project.

## Languages
The eBPF virtual machine ultimately executes bytecode, so the original source language is not what the verifier sees.

The tutorial states that compilers can generate eBPF bytecode from languages including:

- C;
- Rust.

It also discusses user-space components written in other languages, such as Go or Python. BCC is an example where Python is used for the user-space side while the eBPF program itself is compiled separately.

The transcript specifically notes that garbage-collected language behavior does not naturally map to the eBPF VM in the same way as compiled low-level code.

> **Source window:** ~71:32–75:60

### Check your understanding
Why does writing an eBPF program in Rust not eliminate the need for the verifier?

# 18. Concept map

```text
                         eBPF
                          |
        +-----------------+------------------+
        |                 |                  |
     Hooks/events       Programs          User space
        |                 |                  |
  kprobe / tracepoint     |            loaders / agents
  LSM / perf / network    |                  |
  XDP / other hooks       |                  |
        |                 v                  |
        +--------> eBPF bytecode <-----------+
                         |
                      verifier
                         |
                      kernel
                         |
             +-----------+-----------+
             |                       |
           helpers                   maps
             |                       |
     context-aware actions     shared/state/event data
                                     |
                                     v
                              user-space consumers
```

This diagram summarizes the relationships described throughout the tutorial rather than introducing a new architecture.

# 19. Review questions

Try to answer these without looking back at the previous sections.

1. What problem does eBPF solve that ordinary kernel patching makes operationally difficult?
2. What is the relationship between a user-space application, a system call, and the kernel?
3. Name four kinds of events or hooks to which the tutorial says eBPF programs can attach.
4. Why is Kubernetes a particularly natural environment for kernel-level instrumentation?
5. What is the purpose of the context argument passed to an eBPF program?
6. What problem do eBPF maps solve?
7. What does BCC abstract away for a beginner?
8. Why can kernel-version differences affect eBPF portability?
9. What does the compiler produce before a program is loaded into the kernel?
10. What role does `bpftool` play?
11. What makes XDP useful for high-performance packet processing?
12. What are `XDP_PASS` and `XDP_DROP` used for in the lab?
13. Give three categories of checks performed by the verifier according to the tutorial.
14. Why can code compile successfully yet fail verification?
15. Why should permission to load eBPF programs be tightly controlled?
16. What role does a user-space agent play in a production eBPF tool?
17. Why might a system use two eBPF programs around one system call?
18. Why is there no single universal pattern for managing all eBPF programs?
19. Why does the verifier operate independently of whether the original program was written in C or Rust?
20. What is the simplest end-to-end mental model for an eBPF application?

# 20. Suggested answers

<details>
<summary>Open after attempting the questions</summary>

1. eBPF allows kernel behavior to be extended dynamically without waiting for a kernel change to propagate through release/deployment cycles and without necessarily rebooting.
2. A user-space application requests kernel services through system calls; the kernel performs the privileged/system-level operation.
3. Examples: kprobes, uprobes, tracepoints, network packet hooks, Linux Security Module hooks, and performance events.
4. Many containers on a node share one kernel, so kernel instrumentation can observe or influence activity across workloads.
5. It provides information related to the event and program type that triggered execution.
6. They provide structured shared data between eBPF programs and/or user space, avoiding reliance on a single trace-output stream.
7. Compiling, loading, attaching, and part of the user-space interaction around the eBPF program.
8. Kernel data structures can differ across kernel versions.
9. An object containing eBPF bytecode and relevant definitions such as maps.
10. It is a low-level tool for inspecting, loading, and manipulating eBPF programs and maps.
11. It runs very early in packet reception and can make decisions before ordinary network-stack processing goes far.
12. They tell processing to continue or to drop the packet.
13. Pointer-safety checks, execution/completion complexity checks, and validation that helpers are appropriate for the program context.
14. The verifier imposes eBPF-specific safety constraints on the generated bytecode beyond ordinary language compilation.
15. Verified eBPF programs can still intentionally perform extremely powerful operations, including altering networking behavior.
16. It manages program lifecycle—loading, updating, coordinating, and connecting kernel-side programs to user-space functionality.
17. Entry gives access to arguments, while exit gives access to results; maps can correlate the two.
18. Requirements vary by application, and kernel/eBPF capabilities continue to evolve.
19. The verifier analyzes generated eBPF bytecode rather than the original source language.
20. **hook/event → eBPF program → helpers/maps → user-space manager or consumer**.

</details>

# 21. Source map

This notebook reorganizes the tutorial by topic. The following timestamp ranges from the transcript were used as the principal source for each lesson:

| Topic | Approx. transcript window |
|---|---:|
| eBPF definition and motivation | 03:27–06:21 |
| User space, kernel, hooks | 06:29–09:00 |
| Kubernetes relevance | 09:03–10:34 |
| Hello World and context | 10:41–18:42 |
| Maps | 15:20–20:40 |
| BCC and portability | 22:21–25:32 |
| Bytecode / VM / loading | 28:45–31:32 |
| `bpftool` | 31:34–33:10 |
| Networking | 39:55–46:07 |
| XDP | 46:10–49:26 |
| Enterprise trust / program provenance | 51:01–54:33 |
| Verifier | 56:57–61:23 |
| Context structures | 62:40–64:24 |
| User-space consumption | 64:37–65:36 |
| Production program lifecycle | 65:39–71:15 |
| Workflow and languages | 71:32–75:60 |

**Reference file:** `Tutorial Getting Started with eBPF - Liz Rice, Isovalent.txt`

---
## Repository note

This notebook is designed to be readable directly on GitHub. It contains no required executable setup and can be used as lecture notes, a study handout, or a base for adding your own practical labs later.