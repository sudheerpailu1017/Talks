# Scaling AI on Kubernetes: Lessons from Uber on Addressing GPU Starvation and Scheduling Nightmares

Uber's experience scaling AI workloads on Kubernetes offers valuable insights into tackling the persistent challenges of GPU starvation and inefficient scheduling. Their journey highlights the power of extending Kubernetes with custom solutions tailored to the unique demands of AI. Here's a breakdown of their key innovations and their implications:

## Uber's Strategies for Efficient GPU Management on Kubernetes

**1. Elastic Resource Management: Dynamic GPU Sharing**

* **The Problem:** Inefficient GPU utilization in multi-tenant clusters due to static resource allocation.
* **Uber's Solution:** Hierarchical resource pools with fairness guarantees, enabling borrowing of idle resources through a dynamic preemption system. Non-critical "preemptible" pods are evicted when the original resource owner needs them back.
* **The Workflow:** Pods queue, undergo entitlement checks, and then are placed by the default Kubernetes scheduler.
* **The Impact:** Near-100% GPU utilization during peak demand without compromising stability, effectively turning wasted resources into available capacity.

**2. Gang Scheduling for Distributed Training: Ensuring Atomic Placement**

* **The Problem:** Partial pod placement in distributed training leading to deadlocks and significantly longer training times.
* **Uber's Solution:** Gang scheduling by grouping pods of the same training job using labels and annotations, enforced by a custom admission controller to ensure atomic scheduling of all pods in the gang.
* **The Impact:** Elimination of a 300% training time penalty caused by partial pod placements, drastically speeding up AI model training.

**3. GPU-Specific Optimizations: Hardware-Aware Orchestration**

* **The Problem:** Inefficient GPU usage in mixed hardware environments, with non-GPU workloads potentially occupying premium GPU nodes.
* **Uber's Solution:** Heterogeneous cluster management (combining different GPU and CPU nodes), workload segmentation (e.g., Ray data loaders on CPU, training on GPU), a GPU filter plugin to block non-GPU workloads from GPU nodes, and node tagging with specific GPU models allowing pods to request compatible hardware.
* **The Impact:** Reservation of over $250k worth of GPUs for priority jobs while maintaining 85% utilization across their GPU fleet, ensuring the right hardware is used for the right tasks.

**4. Metrics-Driven Optimization: Data-Informed Resource Allocation**

* **The Problem:** Wasted GPU memory in inference pods due to over-allocation.
* **Uber's Solution:** A monitoring stack (cAdvisor, custom agents) to collect detailed pod-level GPU metrics, aggregated in Grafana. This insight revealed significant memory wastage in inference pods, leading to the implementation of memory slicing to reduce this waste.
* **The Impact:** Better resource utilization through data-driven insights and targeted optimizations like memory slicing.

**5. Control Plane Innovations: Enhancing Job Lifecycle Management**

* **The Problem:** Resource leaks from completed or failed jobs and idle clusters.
* **Uber's Solution:** Automatic termination of Ray clusters upon job completion or failure with cascading deletion of resources, and idle detection to automatically kill unused clusters after a period of inactivity.
* **The Impact:** Efficient resource cleanup and freeing up valuable resources automatically.

## Implications for AI Infrastructure: Key Takeaways

Uber's journey demonstrates that:

* **Dynamic resource fluidity** can be achieved in Kubernetes for GPUs, offering cloud-like elasticity through custom scheduling and preemption.
* **Hardware-aware orchestration** is crucial for efficient GPU utilization and preventing costly mismatches between workloads and hardware.
* **Custom admission control** layers, like Uber's kube-resource-manager-scheduler, can help prevent API overloads common in large-scale AI inference.

## Conclusion: Extending Kubernetes for AI Success

Uber's experience with Ray on Kubernetes underscores the potential of extending Kubernetes with custom schedulers and intelligent resource management strategies. Their innovations have led to significant improvements in efficiency, training speed, and hardware utilization, providing a real-world blueprint for tackling GPU starvation and scheduling nightmares in the demanding landscape of AI workloads.