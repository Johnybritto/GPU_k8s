# Comprehensive Study Guide: Kubernetes GPU Architecture & Device Plugin Framework

In Kubernetes, the control plane is natively brilliant at managing core resources like CPU and memory. However, it has no native understanding of specialized hardware like GPUs, FPGAs, or high-performance network cards.

To prevent the core Kubernetes code from bloating with vendor-specific logic, Kubernetes relies on a framework called **Extended Resources**. This allows external hardware vendors to advertise and manage their custom hardware within the cluster.

---

## Architectural Foundations of GPUs in Kubernetes

Integrating GPUs into Kubernetes is highly complex because standard containerized applications and GPUs operate on fundamentally different hardware and software paradigms. To understand the solution, we must first look at how CPUs and GPUs differ, and how the Linux operating system handles resource management.

### 1. Hardware & Execution Architectures: CPU vs. GPU

#### **CPU: Master of Context Switching & Branch Prediction**

A CPU can theoretically only execute one instruction at a time. It creates the illusion of simultaneous multi-tasking by incredibly fast **context switching**.

* **Execution:** The CPU saves the state (registers) of one process, loads the state for the next, executes a slice of time, and then repeats this cycle constantly.
* **Silicon Layout:** Because standard software relies heavily on unpredictable logic (like `if/else` branches), a large portion of a CPU's physical silicon is dedicated to **branch prediction**. This hardware guesses the next instruction so it can rapidly feed data into the CPU's execution units.
* **Memory Management:** The CPU handles memory efficiently by mapping virtual memory to physical RAM, dynamically allocating exact needs in granular 4K pages.

#### **GPU: The Unstoppable Matrix Cruncher**

GPUs do not have branch predictors. They are designed for massive, predictable mathematical workloads, like crunching matrices of pixels.

* **Execution (Kernels):** In GPU terminology, a computation applied to data is called a **kernel**. Once a GPU kernel starts executing, it cannot be stopped, paused, or context-switched. It monopolizes the GPU entirely until the calculation is 100% complete.
* **Silicon Layout:** Without the need for branch predictors, GPUs pack a massive matrix of arithmetic logic units (ALUs) onto the hardware. A single control logic unit might dictate the operations for 64 ALUs simultaneously.
* **Memory Management:** Data must be manually transferred from the host CPU's RAM into the GPU's memory before a kernel can execute. For the sake of speed and simplicity, GPU drivers allocate memory in massive, imprecise "chunks". If a process requests 8KB, the driver might reserve 2MB behind the scenes. Furthermore, any process interacting with a GPU is exposed to the entirety of the GPU's memory capacity (e.g., 16GB), even if it only uses a fraction of it.

```text
====================================================================
 DIAGRAM 1: EXECUTION FLOW (CPU vs GPU)
====================================================================

[ CPU Time-Slicing ]
(Simultaneous illusion via rapid state changes)
Process A (run) -> Save State -> Load State -> Process B (run) -> ...

[ GPU Kernel Execution ]
(Strict, blocking execution)
App A Kernel [=================== 100% Monopolized ===================] 
App B Kernel                                                          -> (Waits to run)

```

---

### 2. The Linux Kernel API, Cgroups, and Namespaces

To understand why GPUs struggle in Kubernetes, we have to look at how standard container engines (like Docker or containerd) are built.

* **The Linux Kernel as an API:** Standard applications do not interact with hardware directly. Instead, the application makes **"system calls"** (API calls) to the Linux kernel, which manages the hardware on the application's behalf. Everything your application does must go through this kernel API.
* **Control Groups (cgroups):** To control how much hardware an application can use, Linux uses cgroups. You can think of a cgroup like an HTTP API rate limit. Instead of limiting web requests, cgroups group these kernel system calls together and limit the actual physical resources (like memory, CPU, or network bandwidth) that a process is allowed to consume.
* **Namespaces:** Alongside cgroups, the kernel uses namespaces to segregate processes so that they have their own isolated view of the system. For example, a "mount namespace" gives an application the illusion that it has its own private, isolated file system, when in reality it is just restricted to a specific subfolder on the host.

Standard containers are essentially just wrappers built around these native Linux `cgroups` and `namespaces`.

---

### 3. The GPU Problem: Bypassing the Kernel

This native Linux API system works perfectly for standard applications, but it breaks down completely with GPUs.

Instead of interacting with the open-source Linux kernel via standard system calls, GPU workloads are routed through a **closed-source, proprietary GPU driver**. Because this proprietary driver takes full control of the computations and hardware interactions, the Linux kernel takes a step back.

As a result, the standard Linux cgroups and namespaces simply do not exist within the GPU driver. Unless the creators of the closed-source driver (like NVIDIA) explicitly write isolation features into their code, the native Linux tools cannot be used to limit or segregate GPU access. This complete bypass of standard limits is exactly why GPUs feel like "second-class citizens" in Kubernetes.

```text
====================================================================
 DIAGRAM 2: THE ISOLATION GAP
====================================================================
NATIVE KUBERNETES/LINUX       GPU WORKLOADS
[ Container Engine ]         [ GPU Workload ]
       |                            | (Bypasses Linux limits)
       v                            v
[ cgroups & namespaces ]     [ Closed-Source GPU Driver ]
       |                            |
       v                            v
[ CPU & Host Memory ]        [ Physical GPU ]
====================================================================

```

---

## The 4-Layer Kubernetes GPU Architecture

Because Kubernetes relies on cgroups and namespaces, it natively does not know where GPUs are located, how to schedule them, or how to pass them into an isolated container. To bridge this massive gap, Kubernetes relies on a four-layer architecture:

* **Layer 1: The Physical GPU**
The actual hardware installed on the node.
* **Layer 2: The GPU Driver**
Installed on the host Linux machine, this driver interfaces with the hardware and exposes it as a file descriptor (e.g., mounting it as `/dev/nvidia0`).
* **Layer 3: NVIDIA Container Toolkit**
Because standard containers are isolated, they cannot see the GPU driver natively. The NVIDIA Container Toolkit solves this by patching the Container Runtime Interface (CRI). It uses an Open Container Initiative (OCI) **pre-start hook** that triggers before the container process starts. This hook essentially "punches a hole" in the container's isolation to dynamically inject the GPU driver (`/dev/nvidia0`), environment variables, and necessary host libraries.
* **Layer 4: NVIDIA Device Plugin (The Broker)**
Kubernetes still needs to know which nodes have GPUs so it can schedule pods efficiently. The Device Plugin is deployed as a DaemonSet across the cluster. It scans the underlying node, discovers the GPU, and registers it with the node's **Kubelet**. This plugin makes the node "GPU aware," acting as a broker that allows the Kubernetes scheduler to undergo its standard process of filtering (finding GPU nodes), scoring (ranking the best node), and binding (assigning the pod).

```text
====================================================================
 DIAGRAM 3: THE 4-LAYER INTEGRATION STACK
====================================================================

      [ Kubernetes Scheduler ] <-- 4. Device Plugin tells scheduler where GPUs live
                 |
      [ Kubelet (Node Agent) ] 
                 |
[========================================]
[  POD / CONTAINER (Fully Isolated)      ]
[                                        ]
[  <-- 3. Container Toolkit injects      ]
[        drivers & libs via OCI hook    ]
[========================================]
                 |
                 v
      [ 2. GPU Driver (/dev/nvidia0) ] <-- Bypasses cgroups/namespaces
                 |
                 v
         [ 1. Physical GPU ]

```

---

## How It Works: The Device Plugin Framework

Kubernetes manages GPUs as extended resources by leveraging a Device Plugin that bridges the gap between physical hardware and the Kubernetes scheduler.

1. **Discovery:** A vendor-specific device plugin (such as the NVIDIA Device Plugin) runs on your GPU-enabled nodes, typically as a DaemonSet. It scans the physical host to detect available GPUs.
2. **Registration:** The plugin communicates with the local kubelet via a gRPC interface, advertising the hardware under a custom, fully-qualified domain name like `nvidia.com/gpu` or `amd.com/gpu`.
3. **Advertisement:** The kubelet reports these numbers back to the API Server, making the cluster scheduler aware that the node has a specific quantity of custom resources available for incoming workloads.

---

## Strict Rules of Extended Resources

Because GPUs are treated as "opaque" black-box tokens rather than continuous streams of compute, they come with a few strict scheduling limitations:

* **Integer Only:** You cannot natively request a fraction of a GPU (like `0.5` cpu). You must request whole numbers (`1`, `2`, etc.). Each container gets exclusive access to that physical GPU device.
* **Limits vs. Requests:** For extended resources, Kubernetes strictly schedules workloads based on **Limits**.
* If you only specify a limit of `nvidia.com/gpu: 1`, Kubernetes automatically sets the request to match it.
* If you specify both, they must be identical.
* You cannot specify a request without defining a limit.



### Example Pod Specification

Here is how a container asks for a GPU extended resource:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-ml-workload
spec:
  containers:
    - name: training-container
      image: tensorflow/tensorflow:latest-gpu
      resources:
        limits:
          nvidia.com/gpu: "1" # Requesting 1 full GPU extended resource

```

---

## Core Resources vs. Extended Resources

| Feature | Core Resources (CPU / Memory) | Extended Resources (GPUs / FPGAs) |
| --- | --- | --- |
| **Native Integration** | Built into the core Kubernetes scheduler. | Discovered externally via Device Plugins. |
| **Resource Naming** | Simple strings (e.g., `cpu`, `memory`). | Domain-prefixed strings (e.g., `nvidia.com/gpu`). |
| **Granularity** | Supports fractional values (e.g., `250m` CPU). | Strictly integer-based by default. |
| **Overcommitting** | Allowed (Requests can be lower than Limits). | Not allowed (Requests must equal Limits). |

---

## Overcoming the "Whole GPU" Limitation

Because dedicating an entire enterprise GPU (like an NVIDIA A100 or H100) to a minor inference job is incredibly expensive and wasteful, the ecosystem has evolved to circumvent the basic integer rule:

* **Time-Slicing:** Interleaving multiple workloads on a single GPU by rapidly swapping execution context (similar to how standard CPUs context-switch).
* **Multi-Instance GPU (MIG):** Physically partitioning a single high-end GPU into multiple independent, smaller hardware instances at the silicon level. Each instance has dedicated memory and compute resources, providing clean hardware-level isolation.
