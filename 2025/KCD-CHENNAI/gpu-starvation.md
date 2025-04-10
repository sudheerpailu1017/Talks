# Scaling AI on Kubernetes: Addressing GPU Starvation, API Meltdowns, and Scheduling Nightmares

## GPU Starvation in Kubernetes: Understanding the Challenge and Emerging Solutions

AI workloads, particularly model training and inference, are exceptionally demanding on GPU resources. In Kubernetes environments where multiple workloads vie for these resources, **GPU starvation** becomes a critical issue. Pods can be left in a pending state indefinitely, waiting for GPU access, which severely impacts performance and prolongs job completion times.

The problem is exacerbated by Kubernetes' inherent treatment of GPUs as indivisible resources:

- You can request 1 GPU or 2 GPUs.
- **You cannot natively request "0.5 GPUs".**

Consequently, even pods with modest GPU requirements must reserve an entire GPU, leading to substantial resource underutilization.

### Why It Happens

Standard Kubernetes schedulers, primarily designed for CPU and memory management, struggle with the unique characteristics of GPUs:

| Aspect              | CPU                                  | GPU                                    |
|----------------------|--------------------------------------|-----------------------------------------|
| **Divisibility** | Yes (e.g., millicores)               | **No (whole device)** |
| **Shareability** | Shareable across multiple pods       | Limited, requires specific mechanisms   |
| **Runtime Enforcement** | Well-tracked and enforced by cgroups | **No native equivalent enforcement** |

Specifically, the limitations stem from:

1.  **Indivisibility:** Kubernetes treats GPUs as atomic units, lacking built-in mechanisms for fractional allocation.
2.  **Naive Scheduling:** The default scheduler only considers the *number* of available GPUs, neglecting crucial factors like:
    -   Remaining GPU memory.
    -   Current GPU utilization by other workloads.
3.  **Lack of Runtime Enforcement:** Unlike CPUs with cgroups, GPUs lack native mechanisms to enforce resource limits on memory or compute at runtime.
4.  **Memory Management Gap:** GPU memory, unlike CPU memory, cannot be overcommitted. Insufficient free GPU memory at runtime leads to pod crashes, even if a "GPU" was allocated. This often results in Kubernetes overcommitting GPUs without awareness, causing failures when multiple AI workloads are scheduled on the same node.

### Solutions and Technologies

Several emerging solutions and technologies aim to mitigate GPU starvation and improve GPU utilization in Kubernetes:

#### 1. Multi-Instance GPU (MIG)

-   **What is MIG?** A hardware-based feature available on NVIDIA A100, A30, and H100 GPUs that allows partitioning a single physical GPU into multiple isolated "mini-GPUs".
-   **Each mini-GPU has:**
    -   Dedicated compute cores
    -   Dedicated memory
    -   Independent isolation boundaries
-   **Benefits:**
    -   **Hard isolation** of GPU resources, preventing interference between workloads.
    -   Kubernetes can schedule workloads on MIG slices as if they were discrete GPUs.
-   **Limitations:**
    -   **Preconfigured:** Slice sizes must be determined beforehand.
    -   **Static:** Partitions cannot be dynamically adjusted based on workload changes.
    -   **Hardware dependent:** Only available on specific NVIDIA GPUs.
    -   **Not memory-aware scheduling:** MIG itself doesn't dynamically select nodes based on real-time memory pressure.

#### 2. Time-Slicing (via NVIDIA MPS)

-   **What is Time-Slicing?** A software-based approach using NVIDIA Multi-Process Service (MPS) that enables multiple processes to share a single GPU by rapidly switching between them, allocating "time slices" of the GPU.
-   **Benefits:**
    -   Effective for **inference workloads** with short, lightweight tasks.
    -   Improves GPU core utilization.
-   **Limitations:**
    -   **Does not manage memory sharing.** GPU memory exhaustion remains a risk if many processes load large models.
    -   Requires **application compatibility** with NVIDIA MPS.

#### 3. NVIDIA KAI Scheduler (Experimental)

-   **What is NVIDIA KAI Scheduler?** An experimental, intelligent Kubernetes scheduler developed by NVIDIA specifically to optimize GPU resource utilization for AI/ML workloads. It addresses the shortcomings of the default Kubernetes scheduler for GPU-intensive tasks.

-   **Core Features:**
    -   **Better GPU Bin-Packing:** Intelligently places multiple smaller AI jobs onto the same GPU (or MIG slice) to maximize utilization.
    -   **Gang Scheduling:** Schedules all pods of a distributed training job together, avoiding partial deployments.
    -   **Preemption Support:** Allows higher-priority jobs to preempt lower-priority jobs when GPU resources are scarce.
    -   **Fractional GPU Scheduling:** Enables pods to request a fraction of a GPU (e.g., `0.5`), improving sharing even without MIG.
    -   **Awareness of MIG and Time-Slicing:** Adapts scheduling decisions based on MIG configurations and MPS usage.
    -   **GPU Sharing Across Namespaces:** Allows pods from different namespaces to share the same physical GPU.
    -   **Job Affinity and Anti-Affinity:** Supports rules for co-locating related jobs or spreading them for better performance isolation.
    -   **Workload Priority Classes:** Enables prioritizing different types of AI workloads.

-   **GPU Sharing in KAI Scheduler:**
    -   **How it Works:** KAI enables logical GPU sharing by allowing pods to specify their GPU memory requirements through annotations at creation time:
        -   `gpu-fraction: "0.5"` (request 50% of the GPU)
        -   `gpu-memory: "2000"` (reserve 2000 MiB of GPU memory)
    -   During scheduling, KAI groups pods such that their combined declared usage doesn't exceed the total GPU memory. These pods are then scheduled onto the same physical GPU.
    -   **Crucially, there is no hardware-level enforcement of these limits at runtime.** It operates on a trust-based model where pods are expected to adhere to their declared memory usage.
    -   **Key Points About GPU Sharing:**
        | Aspect                   | Behavior                      |
        |--------------------------|-------------------------------|
        | GPU Hardware Division    | ❌ No, GPU remains atomic     |
        | Enforcement of Memory Use | ❌ No, trust model only      |
        | Monitoring at Runtime    | ❌ No automatic throttling   |
        | Cross-Namespace Sharing  | ✅ Yes, supported            |
        | Reservation Pods         | ✅ Yes, for conflict avoidance |

    -   **Risk:** If a pod exceeds its declared memory usage, it can lead to GPU memory exhaustion and crash other co-located pods.

    -   **Reservation Pods:** KAI deploys "reservation pods" in a special namespace (`runai-reservation`) to logically track and manage GPU sharing, preventing conflicts.

    -   **Enabling GPU Sharing in KAI Scheduler:**
        -   **Globally:** Add `--set "global.gpuSharing=true"` during KAI Scheduler installation via Helm.
        -   **Per Pod (Fraction Sharing):**
            ```yaml
            apiVersion: v1
            kind: Pod
            metadata:
              name: gpu-sharing-fraction
              annotations:
                gpu-fraction: "0.5"
            spec:
              containers:
              - name: my-container
                image: your-image
                resources:
                  limits:
                    [nvidia.com/gpu](https://nvidia.com/gpu): 1 # Still request a full GPU for scheduling
            ```
        -   **Per Pod (Fixed Memory Sharing):**
            ```yaml
            apiVersion: v1
            kind: Pod
            metadata:
              name: gpu-sharing-memory
              annotations:
                gpu-memory: "2000"
            spec:
              containers:
              - name: my-container
                image: your-image
                resources:
                  limits:
                    [nvidia.com/gpu](https://nvidia.com/gpu): 1 # Still request a full GPU for scheduling
            ```

    -   **How KAI Scheduler Logically Shares Atomic GPUs:** Because GPUs are hardware-level atomic, KAI's sharing mechanism is a logical agreement enforced at the scheduling layer. It ensures that the total requested memory or fraction of pods on the same GPU does not exceed the physical GPU's capacity. Pods mount the same GPU device and are expected to self-regulate their memory allocation. There is no runtime enforcement by the Linux kernel or GPU drivers.

#### 4. Ongoing Research: Memory-Aware Pod Placement

-   A recent research paper (2024) proposes a lightweight strategy to address GPU memory starvation by making pod placement decisions based on real-time GPU memory usage.
-   **Core Idea:** Monitor the actual GPU memory usage and GPU usage frequency on each node and prioritize nodes with more available memory for new pod placements.
-   **How It Works:**
    -   **Monitoring Agents:** Run on each node to track:
        -   GPU memory used.
        -   Number of running GPU tasks.
        -   GPU idle time.
    -   **Control Node:** Maintains a live list of nodes ranked by available GPU memory and prioritizes nodes with lower GPU usage frequency.
    -   **Scheduling Interception:** When a new pod is created, the control node selects the best node based on real-time GPU memory availability instead of relying solely on the number of available GPUs.
    -   **Pod Placement Algorithm:** Scores each node using a metric like: `(1 - GPU Memory Usage) * (1 - GPU Idle Time)` and selects the node with the highest score.
-   **Advantages:**
    -   **Dynamic decision-making** at scheduling time based on actual resource utilization.
    -   **No changes required in Pods or Helm charts.**
    -   **No dependency on MIG or MPS.**
    -   Helps **avoid runtime crashes** due to GPU memory exhaustion.
-   **Limitations:**
    -   Most effective for **small to medium GPU workloads** (e.g., inference, transfer learning).
    -   May not fully address issues for massive training jobs requiring exclusive GPU access.
    -   Requires a **minor extension of Kubernetes scheduling logic** (e.g., a scheduler extender).

### Final Comparison

| Solution                             | Problem Addressed                       | Limitations                                      |
|--------------------------------------|-----------------------------------------|--------------------------------------------------|
| MIG                                  | Hard resource slicing, isolation        | Static, hardware-dependent                         |
| Time-Slicing                         | Core sharing, improved core utilization | No memory management, application compatibility |
| NVIDIA KAI Scheduler (Experimental)  | Intelligent GPU scheduling, fractional sharing | Experimental, trust-based memory sharing          |
| Memory-Aware Pod Placement (Research) | Live monitoring, smarter pod placement   | Heuristic, not full enforcement                   |

### Conclusion

GPU starvation remains a significant challenge in Kubernetes, particularly for AI/ML workloads. While technologies like MIG, Time-Slicing, and the NVIDIA Operator offer improvements in GPU management, they do not entirely resolve the issue of dynamic GPU memory exhaustion during pod scheduling.

The memory-aware pod placement strategy from ongoing research presents a promising lightweight solution by:

-   Dynamically monitoring GPU memory usage.
-   Selecting optimal nodes in real-time.
-   Mitigating job failures without requiring extensive system modifications.

The future of efficient AI-optimized Kubernetes GPU clusters likely involves a combination of these approaches: leveraging MIG for hardware-level slicing, Time-Slicing for core sharing, and intelligent, memory-aware scheduling to achieve optimal resource utilization and stability.

### Extra: CPU Scheduling vs GPU Scheduling

| Aspect              | CPU                                  | GPU                                     |
|----------------------|--------------------------------------|------------------------------------------|
| **Divisible** | Yes (millicores)               | **No (whole device)** |
| **Scheduling** | Tracks requests and availability natively | Only tracks device count                 |
| **Runtime Enforcement** | Cgroups for CPU throttling           | **No equivalent enforcement** |
| **Starvation Protection** | Strong                                 | Weak                                     |

✅ CPUs are managed effectively within Kubernetes.
🛑 GPUs require more sophisticated handling, especially concerning memory-aware scheduling.