# Governing GPU and CPU Workloads at Scale: AI/ML Policy Enforcement in Multi-Tenant Kubernetes Clusters

Modern data platforms face unprecedented challenges in managing diverse AI/ML workloads across multi-tenant Kubernetes environments. As organizations scale their data science capabilities, the complexity of governing compute resources—from CPU-intensive data processing to GPU-accelerated model training—grows exponentially.

This blog post explores how **Kyverno's new ValidatingPolicy** framework, introduced in version 1.14[^1], provides a streamlined approach to enforcing governance policies for data platform teams supporting data scientists, data analysts, data engineers, and machine learning engineers.

## The Multi-Tenant Data Platform Challenge

In enterprise data platforms, tenants create diverse workloads that support critical business functions without being directly in the payment processing path. These include:

- **Transaction Reconciliation Reports** that customers download or receive through automated data pushes for reconciliation processes
- **Machine learning models** that inform payment authorization decisions (with graceful degradation to older models during training failures)
- **Anti-money laundering models** that detect suspicious transaction patterns
- **Data pipelines** that transform and process customer transaction data

The challenge extends beyond simple resource allocation. Organizations need policies that can intelligently route workloads to appropriate hardware, enforce security standards across different compute types, and provide cost attribution while maintaining the flexibility that AI/ML teams require for rapid experimentation and production deployment.

![Multi-tenant data platform architecture with Kyverno ValidatingPolicy governance][multi-tenant-with-validation-policy-governance.png]

*Multi-tenant data platform architecture with Kyverno ValidatingPolicy governance*

### Multi-Tenant Architecture Fundamentals

Effective multi-tenancy in Kubernetes requires careful consideration of several isolation boundaries. **Namespace isolation** provides logical separation of resources, allowing teams to operate independently while sharing the same cluster infrastructure. **RBAC policies** ensure that users can only access resources within their designated tenants, preventing unauthorized cross-tenant access. **Network policies** control pod-to-pod communication, ensuring that workloads from different tenants cannot interfere with each other.

**Resource quotas** prevent any single tenant from consuming excessive cluster resources, while **Pod Security Standards** ensure consistent security posture across all tenant workloads. For data platform teams, this translates to **cost attribution** per tenant, **workload classification** for appropriate scheduling, and **compliance enforcement** for regulated industries like fintech.

### The Resource Challenge

When a data scientist's GPU training job consumes \$500 in compute resources overnight but fails due to missing runtime class configuration, you've witnessed a costly governance gap. The irony? Many of those expensive GPU resources are wasted on data preprocessing, hyperparameter tuning, and smaller model training tasks that run perfectly well on CPU-only compute[^2][^3]. Traditional Kubernetes resource management can't intelligently route workloads based on their computational requirements, creating a perfect storm of waste, security risks, and operational complexity[^5] that costs organizations millions annually.

## ValidatingPolicy: A New Approach to Data Platform Governance

Kyverno 1.14 introduces **ValidatingPolicy**, a specialized policy type that streamlines validation logic and aligns more closely with Kubernetes-native patterns. For data platform teams, this represents a significant improvement over the previous ClusterPolicy approach [^1].

### Kubernetes Version Requirements

Understanding the compatibility matrix is crucial for planning your data platform governance strategy:

- **Kubernetes 1.30+** is the recommended minimum for production ValidatingPolicy deployments [^4][^6][^8]
- **Kubernetes 1.32+** is required for NUMA-aware Memory Manager features when using Volcano
- Kyverno's ValidatingPolicy can be automatically transformed into native Kubernetes ValidatingAdmissionPolicy for improved performance [^9]

### Key Advantages for Data Platform Teams

**Simplified Policy Structure**: ValidatingPolicy eliminates the nested rule complexity of ClusterPolicy, making policies easier to write and maintain for data science workloads [^1].

**Enhanced Performance**: Optimized specifically for validation scenarios, reducing overhead in high-volume data processing environments [^1][^9].

**Better CEL Integration**: First-class support for Common Expression Language expressions provides more powerful and flexible policy definitions [^7].

**Kubernetes Alignment**: Closer compatibility with Kubernetes ValidatingAdmissionPolicy patterns ensures future-proofing and easier integration [^9].

### Data Platform Workload Classification

The foundation of effective governance starts with intelligent workload classification:

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: ValidatingPolicy
metadata:
  name: data-platform-workload-classification
spec:
  validations:
  - expression: |
      has(object.metadata.labels) &&
      has(object.metadata.labels.team) &&
      has(object.metadata.labels["cost-center"]) &&
      has(object.metadata.labels["workload-type"]) &&
      object.metadata.labels["workload-type"] in [
        "jupyter-notebook",
        "ml-training", 
        "data-processing",
        "model-inference",
        "report-generation"
      ] &&
      has(object.metadata.labels["compute-type"]) &&
      object.metadata.labels["compute-type"] in ["gpu", "cpu", "mixed"]
    message: "Data platform workloads must have required labels for governance and cost attribution"
```

This policy ensures that all workloads created by data scientists and ML engineers include the necessary metadata for proper governance, cost attribution, and resource management.

## Ray Cluster Governance for Distributed Training

Ray has become the de facto standard for distributed ML training in Kubernetes environments. With KubeRay, data platform teams can provide self-service distributed computing capabilities while maintaining governance and cost control.

![Overview of the Ray distributed cluster architecture, highlighting the head node and worker nodes with their key components.][ray-architecture.png]

*Overview of the Ray distributed cluster architecture, highlighting the head node and worker nodes with their key components | credited: Benedat LLC on [Ray: Core Architecture][benedat]*

### Ray Cluster Policy Enforcement

Ray clusters require specialized governance policies that account for their distributed nature and resource requirements.

> **Note**: The values used in the following examples (max replicas, GPU limits, etc.) are for demonstration purposes only and should be adapted to your specific requirements and infrastructure constraints.

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: ValidatingPolicy
metadata:
  name: govern-ray-clusters
spec:
  validations:
  - expression: |
      object.spec.workerGroupSpecs.all(group,
        has(group.template.spec.containers[^0].resources.limits) &&
        (
          (has(group.template.spec.containers[^0].resources.limits["nvidia.com/gpu"]) &&
           int(group.template.spec.containers[^0].resources.limits["nvidia.com/gpu"]) <= 4 &&
           group.maxReplicas <= 8) ||
          (!has(group.template.spec.containers[^0].resources.limits["nvidia.com/gpu"]) &&
           group.maxReplicas <= 20)
        )
      )
    message: "Ray worker groups limited based on resource type: GPU workers max 8 replicas with 4 GPUs each, CPU workers max 20 replicas (adjust limits based on your infrastructure)"
  - expression: |
      has(object.metadata.labels) &&
      has(object.metadata.labels.team) &&
      has(object.metadata.labels["cost-center"]) &&
      has(object.metadata.labels["workload-type"]) &&
      object.metadata.labels["workload-type"] in ["training", "inference", "development"] &&
      has(object.metadata.labels["compute-type"]) &&
      object.metadata.labels["compute-type"] in ["gpu", "cpu", "mixed"]
    message: "Ray clusters must have required labels: team, cost-center, workload-type, compute-type"
```

### Heterogeneous Ray Cluster Management

Ray's enhanced support for heterogeneous clusters allows sophisticated resource management:

```yaml
apiVersion: ray.io/v1
kind: RayCluster
metadata:
  name: mixed-compute-cluster
  labels:
    team: "ml-platform"
    cost-center: "data-science"
    workload-type: "training"
    compute-type: "mixed"
spec:
  rayVersion: "2.47.1"
  headGroupSpec:
    rayStartParams:
      dashboard-host: "0.0.0.0"
    template:
      spec:
        containers:
        - name: ray-head
          image: rayproject/ray:2.47.1-py312-cpu
          resources:
            limits:
              cpu: 4
              memory: 8Gi
  workerGroupSpecs:
  - groupName: cpu-workers
    replicas: 4
    minReplicas: 2
    maxReplicas: 10
    rayStartParams:
      num-cpus: "4"
    template:
      spec:
        containers:
        - name: ray-worker
          image: rayproject/2.47.1-py312-cpu
          resources:
            limits:
              cpu: 4
              memory: 8Gi
  - groupName: gpu-workers
    replicas: 2
    minReplicas: 0
    maxReplicas: 4
    rayStartParams:
      num-gpus: "1"
    template:
      spec:
        containers:
        - name: ray-worker
          image: rayproject/ray:2.47.1-py312-gpu
          resources:
            limits:
              nvidia.com/gpu: 1
              cpu: 8
              memory: 32Gi
```

## Multi-Compute Governance Framework

Data platforms must handle both GPU-accelerated and CPU-only workloads seamlessly. The governance framework addresses several key areas:

![Multi-tenant data platform governance framework components and relationships][framework-components-and-relationships.png]

*Multi-tenant data platform governance framework components and relationships*

### 1. Intelligent Runtime Classification

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: ValidatingPolicy
metadata:
  name: workloads-resource-governance
spec:
  validations:
  - expression: |
      !object.spec.containers.exists(c, 
        has(c.resources.limits) && 
        has(c.resources.limits["nvidia.com/gpu"])
      ) || (
        has(object.spec.runtimeClassName) &&
        object.spec.runtimeClassName == "nvidia" &&
        object.spec.tolerations.exists(t,
          t.key == "nvidia.com/gpu" && t.operator == "Exists"
        )
      )
    message: "GPU workloads must use nvidia runtime class and appropriate tolerations"
  - expression: |
      !(has(object.metadata.labels) && 
        has(object.metadata.labels["workload-type"]) &&
        object.metadata.labels["workload-type"] == "cpu-training") || (
        !object.spec.containers.exists(c,
          has(c.resources.limits) && 
          has(c.resources.limits["nvidia.com/gpu"])
        )
      )
    message: "CPU training workloads should not request GPU resources"
```

### 2. Dynamic Resource Quota Management

Rather than static quotas that lead to either resource waste or starvation, implement dynamic allocation based on actual usage patterns and workload characteristics using **ValidatingPolicy** with external data sources:

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: ValidatingPolicy
metadata:
  name: validate-resource-requests
spec:
  context:
  - name: gpuUsage
    variable:
      globalReference:
        jmesPath: |
          prometheus_query("avg_over_time(DCGM_FI_DEV_GPU_UTIL{exported_namespace='{{request.namespace}}'}[7d]) / 100") || '0'
  - name: cpuUsage
    variable:
      globalReference:
        jmesPath: |
          prometheus_query("avg_over_time(container_cpu_usage_seconds_total{namespace='{{request.namespace}}'}[7d])") || '0'
  validations:
  - expression: |
      !has(object.spec.containers[^0].resources.limits["nvidia.com/gpu"]) ||
      int(object.spec.containers[^0].resources.limits["nvidia.com/gpu"]) <= 
      max(ceil(float(variables.gpuUsage) * 1.2), 1)
    message: "GPU request exceeds recommended allocation based on usage history"
  - expression: |
      !has(object.spec.containers[^0].resources.requests.cpu) ||
      quantity(object.spec.containers[^0].resources.requests.cpu).asApproximateFloat() <= 
      max(ceil(float(variables.cpuUsage) * 1.5), 4.0)
    message: "CPU request exceeds recommended allocation based on usage history"
```

### 3. CPU-First Training Policies

Support CPU-optimized training governance:

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: ValidatingPolicy
metadata:
  name: cpu-training-optimization
spec:
  validations:
  - expression: |
      !(has(object.metadata.labels) && 
        object.metadata.labels["workload-type"] == "cpu-training") ||
      has(object.spec.affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution) ||
      has(object.spec.nodeSelector)
    message: "CPU training workloads should specify node preferences for optimal placement"
```

### 4. Cost Attribution and Billing

For accurate cost tracking across data science teams:

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: ValidatingPolicy
metadata:
  name: cost-attribution-validation
spec:
  validations:
  - expression: |
      has(object.metadata.labels) &&
      has(object.metadata.labels["opencost.io/team"]) &&
      has(object.metadata.labels["opencost.io/workload-category"]) &&
      object.metadata.labels["opencost.io/workload-category"] in [
        "training", "inference", "development", "data-processing"
      ]
    message: "All workloads must include OpenCost labels for accurate billing attribution"
```

### 5. Security and Compliance

Data platforms handling financial data require strict security policies:

```yaml
apiVersion: policies.kyverno.io/v1alpha1
kind: ValidatingPolicy
metadata:
  name: security-compliance-validation
spec:
  validations:
  - expression: |
      object.spec.securityContext.runAsNonRoot == true &&
      !object.spec.containers.exists(c, 
        has(c.securityContext.privileged) && 
        c.securityContext.privileged == true
      )
    message: "Data platform workloads must run as non-root and cannot use privileged containers"
```

## Getting Started

### Prerequisites

- **Kubernetes 1.30+** cluster (1.32+ for NUMA-aware Memory Manager support)
- **Kyverno 1.14+** with ValidatingPolicy support
- **KubeRay operator** for distributed training workloads
- **Optional**: NVIDIA GPU Operator for GPU workloads
- **Optional**: Volcano or Kueue for advanced scheduling


### Quick Setup Guide

1. **Install Kyverno 1.14**:
```bash
kubectl apply -f https://github.com/kyverno/kyverno/releases/download/v1.14.0/install.yaml
```

2. **Deploy KubeRay for distributed training**:
```bash
helm repo add kuberay https://ray-project.github.io/kuberay-helm/
helm install kuberay-operator kuberay/kuberay-operator
```

3. **Apply data platform governance policies**:
```bash
kubectl apply -f governance-policies/workload-classification.yaml
kubectl apply -f governance-policies/ray-governance.yaml
kubectl apply -f governance-policies/cost-attribution.yaml
```

4. **Optional: Deploy Volcano for gang scheduling and NUMA optimization**:
```bash
helm repo add volcano-sh https://volcano-sh.github.io/helm-charts
helm install volcano volcano-sh/volcano
```


## Looking Ahead: The Future of Data Platform Governance

As AI/ML workloads continue to evolve, governance frameworks must adapt to support:

- **Automated policy generation** based on workload characteristics and historical performance
- **Cross-cluster governance** for distributed data processing pipelines
- **Enhanced cost optimization** through predictive resource allocation
- **Compliance automation** for regulatory requirements in financial services

The combination of Kyverno's **ValidatingPolicy** with Ray cluster management and advanced scheduling (Volcano or Kueue) provides a robust foundation for these future enhancements while maintaining the flexibility that data science teams require for innovation.

## Conclusion

Effective governance of AI/ML workloads in multi-tenant data platforms requires a thoughtful balance between control and flexibility. Kyverno 1.14's **ValidatingPolicy** framework, combined with intelligent workload classification, Ray cluster governance, and optional advanced scheduling, enables platform teams to support diverse data science workflows while maintaining operational excellence.

This approach represents a practical path forward for scaling data platform governance in fintech, but can easily be adapted to a broader use case. By starting with core **ValidatingPolicy** patterns and gradually adding advanced features like Volcano scheduling based on specific needs, teams can build sustainable governance frameworks that grow with their AI/ML capabilities.

The key is not choosing between control and innovation, but creating governance patterns that enable both—allowing data scientists, analysts, and ML engineers to focus on delivering business value while ensuring the platform remains secure, cost-effective, and compliant with enterprise requirements. Through proper multi-tenant isolation, Ray cluster governance, and advanced scheduling capabilities, organizations can build data platforms that scale from development environments to enterprise-grade production clusters.

## Appendix A: Advanced Scheduling - Volcano and Kueue Integration

Modern AI/ML workloads often require sophisticated scheduling capabilities beyond what the default Kubernetes scheduler provides. Two primary solutions address these needs: **Volcano** for comprehensive batch scheduling with NUMA awareness, and **Kueue** for job queuing and resource management.

### Volcano: Gang Scheduling and NUMA Optimization

Volcano provides two key capabilities essential for AI/ML workloads:

- **Gang Scheduling**: Ensures that distributed training jobs either get all required resources simultaneously or remain queued. This prevents partial scheduling that wastes resources and degrades performance.
- **NUMA-Aware Scheduling**: For organizations with NUMA architectures, Volcano optimizes memory locality for GPU workloads, reducing cross-NUMA memory access overhead.

#### Volcano Integration for Gang Scheduling

Gang scheduling is particularly critical for distributed training workloads because it prevents resource waste and training failures that occur when only some workers in a distributed job are scheduled [^10][^11][^12]. Without gang scheduling, a distributed training job might have some workers running while others wait for resources, leading to timeouts, failed communications between workers, and ultimately wasted GPU cycles. This is especially costly for expensive hardware accelerators where partial resource allocation provides no training value [^12][^13].

Gang scheduling ensures that distributed training jobs only start when all required resources are available simultaneously, maximizing resource utilization and preventing cascading failures in multi-worker training scenarios [^10].

**Gang Scheduling Example**:

```yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: distributed-training-job
spec:
  minAvailable: 8  # All 8 workers must be available
  schedulerName: volcano
  plugins:
    gang: {}
  tasks:
  - replicas: 8
    name: worker
    template:
      spec:
        containers:
        - name: training-container
          image: pytorch/pytorch:latest
          resources:
            limits:
              nvidia.com/gpu: 1
              cpu: 4
              memory: 16Gi
```


#### Volcano Integration for NUMA-Aware Scheduling

**Optional Enhancement**: For organizations with NUMA architectures, Volcano scheduler provides significant performance benefits for GPU workloads through NUMA-aware scheduling. This is particularly valuable for memory-intensive ML training workloads where cross-NUMA memory access can create bottlenecks.

![NUMA-aware GPU scheduling with Volcano scheduler for optimal performance][numa-aware-gpu-scheduling-with-volcano.png]

*NUMA-aware GPU scheduling with Volcano scheduler for optimal performance*

**Example Volcano Queue Configuration**:

```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  name: ml-training-queue
spec:
  reclaimable: true
  weight: 100
  capability:
    nvidia.com/gpu: "16"
    cpu: "64"
    memory: "256Gi"
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: node.kubernetes.io/numa-topology
            operator: Exists
```

**Important Note**: Volcano integration is completely optional. Organizations using Kueue, the default Kubernetes scheduler, or other scheduling systems achieve excellent results with the core ValidatingPolicy approach. Volcano's NUMA awareness provides additional optimization for specific hardware configurations but is not required for effective data platform governance.

### Kueue: Alternative for Job Queuing

For teams preferring a Kubernetes-native approach, **Kueue** provides gang scheduling and resource management without NUMA optimization [^10][^12]:

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: ml-cluster-queue
spec:
  preemption:
    withinClusterQueue: LowerPriority
  resourceGroups:
  - coveredResources: ["cpu", "memory", "nvidia.com/gpu"]
    flavors:
    - name: gpu-flavor
      resources:
      - name: nvidia.com/gpu
        nominalQuota: 16
      - name: cpu
        nominalQuota: 64
      - name: memory
        nominalQuota: 256Gi
```


## References

[^1]: https://main.kyverno.io/blog/2025/04/25/announcing-kyverno-release-1.14/

[^2]: https://arxiv.org/abs/2303.17727

[^3]: https://arxiv.org/html/2309.02521v3

[^4]: https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/

[^5]: https://eajournals.org/ejcsit/vol13-issue15-2025/dynamic-gpu-aware-scheduling-for-distributed-data-science-workloads-in-kubernetes/

[^6]: https://kubernetes.io/blog/2024/04/17/kubernetes-v1-30-release/

[^7]: https://kyverno.io/blog/2023/11/13/using-cel-expressions-in-kyverno-policies/

[^8]: https://kubernetes.io/blog/2024/04/24/validating-admission-policy-ga/

[^9]: https://kyverno.io/docs/policy-types/validating-policy/

[^10]: https://cloud.google.com/blog/products/containers-kubernetes/using-kuberay-and-kueue-to-orchestrate-ray-applications-in-gke

[^11]: https://www.numberanalytics.com/blog/mastering-gang-scheduling

[^12]: https://docs.ray.io/en/latest/cluster/kubernetes/examples/rayjob-kueue-gang-scheduling.html

[^13]: https://www.numberanalytics.com/blog/gang-scheduling-essentials

[benedat]: https://www.benedat.com/blog-ray-2020-3/

[multi-tenant-with-validation-policy-governance.png]: assets/multi-tenant-with-validation-policy-governance.png
[ray-architecture.png]: assets/ray-architecture.png
[framework-components-and-relationships.png]: assets/framework-components-and-relationships.png
[numa-aware-gpu-scheduling-with-volcano.png]: assets/numa-aware-gpu-scheduling-with-volcano.png
