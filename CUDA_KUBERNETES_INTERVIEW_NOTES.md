# CUDA for Kubernetes / GPU Platform Interviews

## 1. What is CUDA?

**CUDA (Compute Unified Device Architecture)** is NVIDIA's parallel-computing platform and programming model that allows applications to use NVIDIA GPUs for general-purpose computation.

The key idea is parallelism:

```text
AI / ML Application
        ↓
CUDA libraries/runtime
        ↓
NVIDIA Driver
        ↓
NVIDIA GPU
        ↓
Many calculations run in parallel
```

GPUs are particularly useful for matrix/tensor operations, neural-network training, inference, scientific computing, and image/video workloads.

---

## 2. CUDA is NOT the GPU driver

Keep the layers separate:

```text
AI Application
    ↓
PyTorch / TensorFlow
    ↓
CUDA runtime / CUDA libraries
    ↓
NVIDIA Driver
    ↓
NVIDIA GPU
```

- **NVIDIA GPU**: Physical accelerator hardware.
- **NVIDIA Driver**: Lets the operating system communicate with the GPU.
- **CUDA**: Programming/runtime ecosystem used by applications for GPU computation.
- **PyTorch/TensorFlow**: Frameworks that use CUDA underneath when running on NVIDIA GPUs.

Useful node-level command:

```bash
nvidia-smi
```

This helps inspect GPU state, driver information, memory use, utilization, temperature, and running GPU processes.

---

## 3. CUDA Toolkit

The CUDA Toolkit contains components used to build and run CUDA software, including:

- CUDA runtime
- Development libraries
- Header files
- Compiler (`nvcc`)
- Profiling/debugging tools

CUDA source files commonly use the `.cu` extension.

A Kubernetes platform engineer does not normally need to write CUDA kernels, but should understand how the software stack fits together.

---

## 4. Important CUDA-related libraries

### cuBLAS
GPU-accelerated linear algebra, especially matrix operations.

### cuDNN
Optimized deep-learning primitives used by frameworks such as PyTorch and TensorFlow.

### NCCL
**NVIDIA Collective Communications Library**.

Used for high-performance communication between GPUs, especially for distributed and multi-GPU training.

Quick memory:

```text
CUDA  = GPU compute
cuDNN = Deep-learning primitives
NCCL  = Multi-GPU communication
```

---

## 5. CUDA inside Kubernetes

The complete mental model:

```text
                 Kubernetes Scheduler
                        |
                nvidia.com/gpu
                        |
               NVIDIA Device Plugin
                        |
                  GPU Worker Node
                        |
               +----------------+
               | AI/ML Container|
               |                |
               | PyTorch        |
               |    ↓           |
               | CUDA Runtime   |
               | cuDNN / NCCL   |
               +-------|--------+
                       |
           NVIDIA Container Toolkit
                       |
                 NVIDIA Driver
                       |
                   NVIDIA GPU
```

---

## 6. NVIDIA Driver

The driver runs on the GPU worker node.

Typical node stack:

```text
GPU Worker Node
-------------------------
Linux
NVIDIA Driver
containerd / CRI-O
kubelet
-------------------------
NVIDIA GPU
```

If `nvidia-smi` does not work on the node, the problem is likely at the GPU/driver/OS layer before Kubernetes scheduling is even considered.

---

## 7. NVIDIA Container Toolkit

Containers do not automatically get access to host GPU devices.

The NVIDIA Container Toolkit enables GPU-aware containers to use the host's NVIDIA driver and GPU devices.

```text
Container
   |
NVIDIA Container Toolkit
   |
Host NVIDIA Driver
   |
Physical GPU
```

Memory aid:

**Container Toolkit = makes the GPU usable inside containers.**

---

## 8. NVIDIA Device Plugin

Kubernetes must know that a worker node has GPUs.

The NVIDIA Device Plugin commonly runs as a DaemonSet and advertises GPU resources to kubelet/Kubernetes.

Example:

```text
GPU Node
   |
NVIDIA Device Plugin
   |
kubelet
   |
Kubernetes sees:
nvidia.com/gpu: 4
```

Check with:

```bash
kubectl describe node <gpu-node>
```

You may see:

```text
Capacity:
  nvidia.com/gpu: 4

Allocatable:
  nvidia.com/gpu: 4
```

---

## 9. How a Pod requests a GPU

Example:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

If a node has 4 allocatable GPUs and a pod requests 1, Kubernetes can schedule that pod there if all other scheduling constraints are satisfied.

Remember:

**Device Plugin answers: which pod gets a GPU?**

**CUDA answers: how does the application compute on that GPU?**

---

## 10. Host vs Container responsibility

A useful operational split:

```text
HOST
-------------------------
Linux
NVIDIA Driver
GPU
-------------------------

CONTAINER
-------------------------
Application
PyTorch / TensorFlow
CUDA runtime/libraries
-------------------------
```

This is one of the most important concepts for troubleshooting version issues.

---

## 11. CUDA / Driver compatibility

The software chain must be compatible:

```text
Application
   ↓
PyTorch/TensorFlow version
   ↓
CUDA runtime/libraries in container
   ↓
NVIDIA Driver on host
   ↓
GPU hardware
```

Common problems:

- CUDA runtime too new for the installed driver
- Driver not loaded correctly
- Wrong container image
- PyTorch/TensorFlow built for a different CUDA version
- GPU not exposed correctly to the container

Do not assume every GPU problem is a Kubernetes problem.

---

## 12. NVIDIA GPU Operator

For larger Kubernetes estates, NVIDIA GPU Operator can automate deployment and lifecycle management of important GPU software components.

Conceptually:

```text
GPU Operator
    |
    +-- GPU driver components
    +-- Container Toolkit
    +-- Device Plugin
    +-- GPU monitoring components
    +-- supporting GPU software
```

Exact behavior depends on the cluster design and whether drivers are managed externally.

Interview answer:

> For a larger GPU Kubernetes estate I would evaluate NVIDIA GPU Operator rather than manually configuring each GPU node, while keeping driver and CUDA compatibility controlled through tested versions and staged upgrades.

---

## 13. MIG

**MIG = Multi-Instance GPU**.

Supported NVIDIA GPUs can be partitioned into multiple isolated GPU instances.

```text
Physical GPU
     |
  +--+--+--+
  |  |  |  |
MIG MIG MIG MIG
```

This can improve utilization and isolation.

### MIG vs time slicing

- **MIG**: Hardware-level partitioning/isolation on supported GPUs.
- **Time slicing**: Multiple workloads share GPU execution time.

Interview line:

> Depending on workload isolation and performance requirements, I would choose dedicated GPUs, MIG partitioning, or time slicing rather than assuming every workload needs an entire GPU.

---

## 14. Multi-GPU training and NCCL

Example:

```text
Node 1
GPU0 GPU1 GPU2 GPU3

Node 2
GPU0 GPU1 GPU2 GPU3
```

Distributed training needs communication among GPUs.

```text
        Training Job

GPU0 ─┐
GPU1 ─┤
GPU2 ─┼── NCCL communication
GPU3 ─┤
GPU4 ─┤
GPU5 ─┘
```

For multi-node training, network performance matters greatly.

Related terms you may hear:

- High-speed Ethernet
- InfiniBand
- RDMA
- GPUDirect RDMA

A platform engineer should understand why communication bandwidth/latency can become the bottleneck.

---

## 15. GPU monitoring

A pod being `Running` does not mean its GPU is healthy or efficiently used.

Typical stack:

```text
NVIDIA GPU
    ↓
DCGM
    ↓
DCGM Exporter
    ↓
Prometheus
    ↓
Grafana
    ↓
Alertmanager
```

Important metrics:

- GPU utilization
- GPU memory utilization
- Temperature
- Power
- ECC errors
- Throttling
- GPU availability
- Idle GPUs

A major platform concern is **allocated-but-idle GPU capacity**, because GPUs are expensive.

---

## 16. Troubleshooting: GPU Pod Pending

Start with:

```bash
kubectl get pod
kubectl describe pod <pod>
```

Then check in this order:

```text
Pod Pending
    ↓
Does pod request nvidia.com/gpu?
    ↓
Does any node advertise GPU?
    ↓
kubectl describe node
    ↓
Is GPU allocatable?
    ↓
Taints/tolerations?
    ↓
nodeSelector/affinity?
    ↓
ResourceQuota?
    ↓
NVIDIA Device Plugin healthy?
```

Important point:

A **Pending** pod has not started yet, so do not begin troubleshooting CUDA first. Start with Kubernetes scheduling.

---

## 17. Troubleshooting: Pod Running but CUDA does not work

If the pod is already running:

```text
Pod Running
    ↓
Does node nvidia-smi work?
    ↓
NVIDIA driver healthy?
    ↓
Container Toolkit configured?
    ↓
GPU visible inside container?
    ↓
CUDA/runtime compatibility?
    ↓
Does framework detect CUDA?
```

For PyTorch, application teams may check:

```python
torch.cuda.is_available()
```

If Kubernetes allocated the GPU but the framework cannot use CUDA, scheduling probably worked and the problem is deeper in the runtime/application stack.

---

## 18. Troubleshooting: GPU allocated but utilization is 0%

Possible causes:

```text
GPU allocated
     ↓
Is application actually using CUDA?
     ↓
Does framework detect GPU?
     ↓
Is workload accidentally running on CPU?
     ↓
CUDA/runtime compatibility?
     ↓
Application configuration?
```

Kubernetes allocating a GPU does **not** guarantee that the application is actually using the GPU.

---

## 19. Troubleshooting poor GPU performance

Look end-to-end:

```text
Application
    ↓
CPU preprocessing
    ↓
Memory
    ↓
Storage / Dataset
    ↓
Network
    ↓
GPU
    ↓
Multi-GPU communication
```

Potential bottlenecks:

- CPU starvation
- Slow storage/data loading
- Network latency/bandwidth
- GPU memory pressure
- Thermal/power throttling
- NCCL communication overhead
- Poor application batching/configuration

---

## 20. Platform Team vs ML Team

Conceptual ownership split:

```text
Platform / Infrastructure Team
        |
        +-- GPU servers
        +-- Linux
        +-- NVIDIA drivers
        +-- Container runtime
        +-- NVIDIA Container Toolkit
        +-- Kubernetes
        +-- Device Plugin / GPU Operator
        +-- Scheduling policies
        +-- Monitoring
        +-- Security
        +-- Capacity management

ML / Application Team
        |
        +-- Python
        +-- PyTorch/TensorFlow
        +-- CUDA-compatible application image
        +-- Model
        +-- Training/inference configuration
```

For a Kubernetes platform engineer, the strongest position is:

> I build and operate the secure, reliable Kubernetes/GPU platform on which ML teams run training and inference workloads.

---

# Interview Answers to Memorize

## What is CUDA?

> CUDA is NVIDIA's parallel-computing platform and programming model that enables applications such as PyTorch and TensorFlow to use NVIDIA GPUs for computation. In Kubernetes, I separate CUDA from Kubernetes GPU integration: the host has the NVIDIA GPU and driver, the container carries the application and required CUDA runtime/libraries, NVIDIA Container Toolkit enables GPU access inside the container, and the NVIDIA Device Plugin exposes GPU resources to Kubernetes so workloads can request resources such as nvidia.com/gpu.

## CUDA vs Device Plugin

> The NVIDIA Device Plugin integrates GPUs with Kubernetes scheduling and allocation. CUDA is the compute/runtime layer that applications use after they get access to the GPU.

## CUDA vs Container Toolkit

> CUDA provides the GPU programming/runtime environment. NVIDIA Container Toolkit enables containers to access NVIDIA GPUs and required host-driver capabilities.

## CUDA vs cuDNN vs NCCL

> CUDA is the general GPU compute platform, cuDNN provides optimized deep-learning operations, and NCCL handles high-performance communication among GPUs for multi-GPU or distributed training.

## How would you operationalize GPUs in Kubernetes?

> I would use dedicated GPU node pools, appropriate labels and taints, NVIDIA Device Plugin or GPU Operator, tested driver/runtime versions, ResourceQuota and scheduling controls, and DCGM/Prometheus/Grafana for GPU utilization and health. For utilization efficiency I would evaluate dedicated GPUs, MIG, or time slicing depending on workload requirements.

---

# One Diagram to Remember

```text
               Kubernetes Scheduler
                       |
               nvidia.com/gpu
                       |
              NVIDIA Device Plugin
                       |
                 GPU Worker
                       |
              +----------------+
              | AI/ML Container|
              |                |
              | PyTorch        |
              |    ↓           |
              | CUDA Runtime   |
              | cuDNN / NCCL   |
              +-------|--------+
                      |
          NVIDIA Container Toolkit
                      |
                NVIDIA Driver
                      |
                  NVIDIA GPU
                      |
              DCGM Monitoring
                      |
             Prometheus/Grafana
```

If this stack is clear, most Kubernetes + CUDA interview questions become much easier to reason through.


---

# MIG and GPU Time-Slicing — Kubernetes Configuration

## MIG vs Time-Slicing

**MIG (Multi-Instance GPU)** partitions a supported NVIDIA GPU into isolated GPU instances with defined compute/memory resources. **Time-slicing** does not physically partition the GPU; multiple workloads share execution time on the same GPU.

| Area | MIG | Time-Slicing |
|---|---|---|
| Physical partitioning | Yes, supported GPUs only | No |
| Isolation | Stronger | Lower |
| Predictability | Better | Lower |
| Model | Separate GPU instances | Same GPU shared over time |
| Typical use | Production workloads needing isolation | Smaller/intermittent/dev/inference workloads |

## Normal Dedicated GPU Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-app
spec:
  containers:
  - name: app
    image: my-ai-app:latest
    resources:
      limits:
        nvidia.com/gpu: 1
```

## MIG at Kubernetes YAML Level

MIG is **not created by application Pod YAML**. The platform administrator configures MIG on a supported GPU. NVIDIA Device Plugin/GPU Operator then advertises the resulting MIG resources to Kubernetes.

Depending on the GPU and configuration, resources may look like:

```text
nvidia.com/mig-1g.10gb
nvidia.com/mig-2g.20gb
```

The exact profile names depend on the GPU model/configuration.

A workload then requests an advertised MIG resource:

```yaml
resources:
  limits:
    nvidia.com/mig-1g.10gb: 1
```

```text
Platform Admin
      ↓
Configure MIG on GPU
      ↓
GPU Operator / Device Plugin
      ↓
Advertises MIG resources
      ↓
Kubernetes Scheduler
      ↓
Pod requests required MIG profile
```

The Pod **requests** a MIG instance; it does not create the partition.

## Where Time-Slicing Is Configured

Time-slicing is also **not configured in the application Pod**. It is configured at the **NVIDIA Device Plugin layer**, commonly through a ConfigMap and, in GPU Operator environments, through device-plugin configuration managed/referenced by the operator.

Example device-plugin configuration:

```yaml
version: v1
sharing:
  timeSlicing:
    resources:
    - name: nvidia.com/gpu
      replicas: 4
```

Example ConfigMap structure:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nvidia-device-plugin-config
  namespace: nvidia-device-plugin
data:
  config.yaml: |
    version: v1
    sharing:
      timeSlicing:
        resources:
        - name: nvidia.com/gpu
          replicas: 4
```

The exact namespace, ConfigMap name and how it is referenced depend on how NVIDIA Device Plugin/GPU Operator is installed.

```text
Device Plugin ConfigMap
        |
        | timeSlicing
        | replicas: 4
        ↓
NVIDIA Device Plugin
        ↓
GPU Worker Node
        ↓
Multiple schedulable shared references
        ↓
Kubernetes Scheduler
        ↓
Application Pod requests nvidia.com/gpu: 1
```

### What does replicas: 4 mean?

It does **not** create four GPUs, divide the GPU into guaranteed 25% partitions, or provide MIG-style hardware isolation. It makes each physical GPU available as multiple schedulable shared references according to the device-plugin time-slicing configuration.

```text
1 Physical GPU
      |
timeSlicing replicas: 4
      |
 +----+----+----+----+
 |    |    |    |    |
Pod1 Pod2 Pod3 Pod4
       share
    the same GPU
```

The application can still request:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

## Two Configuration Layers

```text
PLATFORM CONFIGURATION
        |
        +-- GPU Operator / Device Plugin
        +-- MIG configuration
        +-- Time-slicing configuration / ConfigMap
                     ↓
            Advertised GPU resources
                     ↓
APPLICATION POD YAML
        |
        +-- nvidia.com/gpu: 1
        OR
        +-- exposed MIG profile
```

**Platform team controls how GPUs are exposed/shared. Application teams request the GPU resource they need.**

## Useful Checks

```bash
kubectl get pods -A | grep -i nvidia
kubectl get daemonset -A | grep -i nvidia
kubectl get configmap -A | grep -i nvidia
kubectl describe node <gpu-node>
```

Look under node **Capacity** and **Allocatable** for NVIDIA resources.

## Interview Answers

**Where is time-slicing configured?**

> I configure time-slicing at the NVIDIA device-plugin layer, normally through a ConfigMap referenced by the NVIDIA Device Plugin or managed through GPU Operator. It is platform-level configuration, not application Pod configuration. The application can continue requesting nvidia.com/gpu: 1 while the device plugin controls how the physical GPU is exposed for sharing.

**MIG vs time-slicing?**

> MIG partitions a supported NVIDIA GPU into isolated GPU instances with stronger isolation and predictability. Time-slicing does not physically partition the GPU; multiple workloads share execution time on the same GPU. I would consider MIG where isolation and predictable resources matter and time-slicing where improving utilization for smaller/intermittent workloads is the priority.

**How is MIG configured?**

> MIG is configured at the GPU/platform layer, not by the application Pod. After the GPU is placed into the required MIG configuration, NVIDIA Device Plugin or GPU Operator exposes the MIG instances to Kubernetes. The application then requests the appropriate advertised MIG resource.
