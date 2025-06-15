# Local Kubernetes Development with KinD

This post demonstrates how to use **Kubernetes in Docker** (KinD) for local development and testing. [KinD] provides an excellent way to run local Kubernetes cluster 
using Docker containers as nodes, making it perfect for quick development and testing.

---

## 📋 Table of Contents

- [Why KinD](#why-kind)
- [Prerequisites](#prerequisites)
- [Cluster Configurations](#cluster-configurations)
- [NGINX Ingress Controller](#nginx-ingress-controller)
- [Quick Start](#quick-start)
- [Configuration Files Explained](#configuration-files-explained)
- [Example Usage Scenarios](#example-usage-scenarios)
- [Advanced Configuration](#advanced-configuration)
- [Troubleshooting](#troubleshooting)

---

## Why KinD?

As platform engineers, we often need to test configurations, validate deployments, try new operators, and experiment with different cluster topologies before 
using a real cluster. KinD offes several advantages:

- **Ligthweight**: Uses containers instead of VMs. It is very likely that you can run contianers in your OS already, for example with [Docker] or [Podma]n
- **Fast startup**: Clusters can boot in a matter of seconds
- **Reproducible**: Consistent environments across team members
- **Multi-node support**: Can simulate complex cluster topologies
- **Local development**: No cloud cost (or access to your on-prem infrastructure) for development work

## Prerequisites

Before getting started, ensure you have the following tools installed:

- **Docker**: [Install Docker](https://docs.docker.com/get-docker/)
- **KinD**: [Install KinD](https://kind.sigs.k8s.io/docs/user/quick-start/)
- **kubectl**: [Install kubectl](https://kubernetes.io/docs/tasks/tools/)
- **Helm**: [Install Helm](https:/helm.sh/docs/intro/install/)
- **Make**: Usually pre-installed on Linux/macOS

## Cluster Configurations

This example provides two different cluster configurations to demonstrate various KinD capabilities:

### Single-Node Cluster

- **Purpose**: Lightweight development and testing
- **Topology**: One node acting as both control-plane and worker
- **Ports**: 180 (HTTP) and 1443 (HTTPS)
- **Use case**: Simple application testing; learning kubernetes

### Multi-Node Cluster

- **Purpose**: Simulation product-like environments
- **Topology**: One control-plane + two worker nodes
- **Ports**: 280 (HTTP) and 2443 (HTTPS)
- **Use case**: Testing pod scheduling, node affinity, multi-node scenarios

### Port Configuration Explained

Instead of using standard ports `80/443`, we use custom ports to avoid conflicts:

- **Single-node**: `180/1443` - Easy to remember (**1**80 = **1**80 for **1** node)
- **Multi-node**: `280/2443` - Easy to remember (**2**80 = **2**80 for **2** workre nodes)

This approach allows both clusters to run simultaneously without port conflicts.

## NGINX Ingress Controller

> [!NOTE]  
> The [NGINX Ingress Controller][nginx] is included purely for demonstration purposes. It showcases how to deploy and configure applications
> in different cluster topologies. The memory limits are set conservatively (256MB) to ensure the ingress controller runs smoothly
> on local development machines.

## Quick Start

1. **Clone and and navigate the directory**:

```shell
git clone https://github.com/cmontemuino/tech-knowledge-base.git
cd tech-knowledge-base/blog-posts/local-kuberentes-cluster-kind/
```

2. **Create a single-node cluster**:

```shell
make up-single
export KUBECONFIG=$PWD/.kube/config-single
```

3. **Create a multi-node cluster** (in a new terminal):

```shell
make up-multi
export KUBECONFIG=$PWD/.kube/config-multi
```

4. **Check cluster status**:

```shell
make info-single
make info-multi
```

5. **Test your setup**:

```shell
kubectl get nodes
kubectl get pods - n ingress-nginx
```

6. **Clean up when done**:

```shell
make down-single
make down-multi
```

## Configuration Files Explained

### KinD Cluster Configurations

The KinD configuration files define cluster topology and networking:

- **Node roles**: Control-plan vs worker nodes
- **Port mappings**: Expose cluster service to localhost
- **Node labels**: Used for pod scheduling and node selection
- **kubeadm patches**: Customize cluster initialization

### NGINX Ingress Values

The Helm values files configure the NGINX Ingress Controller:

- **Resource limits**: Conservative memory allocation for local development
- **Node selection**: Different strategies for single vs multi-node
- **Tolerations**: Allow scheduling on control-plane (single-node only)
- **Anti-affinity**: Spread replicas across workers (multi-node only)

## Example Usage Scenarios

### Testing Pod Scheduling

#### Multi-node cluster

```shell
export KUBECONFIG=$PWD/.kube/config-multi
kubectl get nodes -o wide
kubectl create deployment test-app --image=nginx:1.28.0 --replicas=3
kubectl get pods -o wide # See pods distributed across workers
```

### Testing Ingress

#### Create a test service

> [!NOTE]  
> We use the Google's [Hello][hello-app] applicaiton for demo purposes. This is a Go web server
> application that does nothing particularly intesting, other than replying to all HTTP requests
> on port 8080 with a `Hello, world!` response.

```shell
kubectl create deployment hello --image=gcr.io/google-samples/hello-app:1.0
kubectl expose deployment hello --port=8080
```

### Resource Monitoring

```
kubectl top nodes 2>/dev/null || echo "Metrics server not available"
kubectl get pods -n ingress-nginx -o jsonpath='{range.items*}{.metadata.name}{"\t"}{.status.phase}{"\t"}{.spec.nodeName}{"\n"}{end}'
```

## Advanced Configuration

You can customize the setup by defining environment variables:

- **Kubernetes version**: `K8S_VERSION` (e.g., `1.31.1`)
- **NGINX version**: `NGINX_VERSION` (default is `1.12.3`)

Other customizations:

- **Resource limits**: Modify the nginx values file
- **Port mappings**: Update the KinD configuration files
- **Node count**: Add more workers to the multi-node configuration

## Troubleshooting

### Common Issues

1. **Port conflicts**: Ensure ports 180, 1443, 280, 2443 are available. Otherwise update the KinD configuration files
2. **Docker resources**: KinD requires sufficient Docker memory/CPU. A good rule-of-thumb for the examples in this post is reserving 4GB of memory and 2 CPUs

> [!NOTE]  
> I do not use Docker Desktop but [lima] + docker-cli.
> <details>
> <summary><strong>Click to expand</strong> and see the configuration I've used here</summary>
>
> ```shell
> ❯ limactl --version
> limactl version 1.1.1
> 
> ❯ docker --version
> Docker version 28.2.2, build e6534b4eb7
> 
> ❯ limactl create --name=docker \
>   --arch=aarch64 \
>   --cpus=2 \
>   --memory=4  \
>   --vm-type=vz \
>   --mount-type=virtiofs \
>   --mount=$HOME:w \
>   template://docker
> ? Creating an instance "docker" Proceed with the current configuration
> INFO[0023] Attempting to download the image              arch=aarch64 digest="sha256:c933c6932615d26c15f6e408e4b4f8c43cb3e1f73b0a98c2efa916cc9ab9549c" location="https://cloud-images.ubuntu.com/releases/noble/release-20250516/ubuntu-24.04-server-cloudimg-arm64.img"
> Downloading the image (ubuntu-24.04-server-cloudimg-arm64.img)
> 578.20 MiB / 578.20 MiB [----------------------------------] 100.00% 42.35 MiB/s
> INFO[0037] Downloaded the image from "https://cloud-images.ubuntu.com/releases/noble/release-20250516/ubuntu-24.04-server-cloudimg-arm64.img"
> INFO[0037] Converting "/Users/redacted/.lima/docker/basedisk" (qcow2) to a raw disk "/Users/redacted/.lima/docker/diffdisk"
> 3.50 GiB / 3.50 GiB [---------------------------------------] 100.00% 1.22 GiB/s
> INFO[0040] Expanding to 100GiB
> INFO[0040] Run `limactl start docker` to start the instance.
> 
> ❯ docker --version
> Docker version 28.2.2, build e6534b4eb7
> ❯ limactl start docker
> INFO[0000] Using the existing instance "docker"
> INFO[0000] Starting the instance "docker" with VM driver "vz"
> INFO[0000] [hostagent] hostagent socket created at /Users/redacted/.lima/docker/ha.sock
> INFO[0000] [hostagent] Starting VZ (hint: to watch the boot progress, see "/Users/redacted/.lima/d
> INFO[0001] SSH Local Port: 61682
> INFO[0000] [hostagent] Waiting for the essential requirement 1 of 2: "ssh"
> INFO[0000] [hostagent] [VZ] - vm state change: running
> INFO[0010] [hostagent] Waiting for the essential requirement 1 of 2: "ssh"
> INFO[0011] [hostagent] The essential requirement 1 of 2 is satisfied
> INFO[0011] [hostagent] Waiting for the essential requirement 2 of 2: "user session is ready for
> INFO[0011] [hostagent] The essential requirement 2 of 2 is satisfied
> INFO[0011] [hostagent] Waiting for the optional requirement 1 of 1: "user probe 1/1"
> INFO[0011] [hostagent] Forwarding "/run/user/501/docker.sock" (guest) to "/Users/redacted/.lima/docker/sock/docker.sock" (host)
> INFO[0011] [hostagent] Guest agent is running
> INFO[0011] [hostagent] Not forwarding TCP 127.0.0.54:53
> INFO[0011] [hostagent] Not forwarding TCP 127.0.0.53:53
> INFO[0011] [hostagent] Not forwarding TCP [::]:22
> INFO[0011] [hostagent] Not forwarding UDP 127.0.0.54:53
> INFO[0011] [hostagent] Not forwarding UDP 127.0.0.53:53
> INFO[0011] [hostagent] Not forwarding UDP 192.168.5.15:68
> INFO[0035] [hostagent] The optional requirement 1 of 1 is satisfied
> INFO[0035] [hostagent] Waiting for the guest agent to be running
> INFO[0035] [hostagent] Waiting for the final requirement 1 of 1: "boot scripts must have finished"
> INFO[0038] [hostagent] The final requirement 1 of 1 is satisfied
> INFO[0039] READY. Run `limactl shell docker` to open the shell.
> INFO[0039] Message from the instance "docker":
> To run `docker` on the host (assumes docker-cli is installed), run the following commands:
> ------
> docker context create lima-docker --docker "host=unix:///Users/redacted/.lima/docker/sock/docker.sock"
> docker context use lima-docker
> docker run hello-world
> ------
> 
> ❯ docker context create lima-docker --docker "host=unix://${HOME}/.lima/docker/sock/docker.sock"
> ❯docker context use lima-docker
>
> ```
>
> ---
> </details>

### Useful Commands

[Docker]: https://www.docker.com
[hello-app]: https://github.com/GoogleCloudPlatform/kubernetes-engine-samples/tree/main/quickstarts/hello-app
[KinD]: https://kind.sigs.k8s.io
[lima]: https://github.com/lima-vm/lima
[nginx]: https://github.com/kubernetes/ingress-nginx
[Podman]: https://podman.io
