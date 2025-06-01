# Local Kubernetes Development with KinD

This post demonstrates how to use **Kubernetes in Docker** (KinD) for local development and testing. [KinD] provides an excellent way to run local Kubernetes cluster 
using Docker containers as nodes, making it perfect for quick development and testing.

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

## Port Configuration Explained

Instead of using standard ports `80/443`, we use custom ports to avoid conflicts:

- **Single-node**: `180/1443` - Easy to remember (**1**80 = **1**80 for **1** node)
- **Multi-node**: `280/2443` - Easy to remember (**2**80 = **2**80 for **2** workre nodes)

This approach allows both clusters to run simultaneously without port conflicts.

## NGINX Ingress Controller

> [!NOTE]  
> The NGINX Ingress Controller is included purely for demonstration purposes. It showcases how to deploy and configure applications
> in different cluster topologies. The memory limits are set conservatively (256MB) to ensure the ingress controller runs smoothly
> on local development machines.

## Quick Start

## Advanced Configuration

## Troubleshooting

### Common Issues

### Useful Commands

[Docker]: https://www.docker.com
[KinD]: https://kind.sigs.k8s.io
[Podman]: https://podman.io
