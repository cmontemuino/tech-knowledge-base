# AMD MI300X for High-Performance AI Computing: Installing the Required Software

This is the second part series on AMD MI300X for High-Performance AI Computing. If you haven't read the first part[^1], I recommend you starting there to understand the hardware architecture and system topology.

## Why This Guide Matters

Installing ROCm (Radeon Open Compute)[^2] software stack is of utmost importance to unlock MI300X GPUs, especially for AI workloads. While AMD provides excellent documentation[^3], enterprise environments sometimes present unique challenges:

- **Air-gapped systems** without internet access
- **Specific kernel versions**
- **Package dependency conflicts**
- **Integration with existing monitoring and management tools**

This guide is intended to walk through a real-world installation on Oracle Linux 9.6, showing actual command outputs and troubleshooting steps.

> **Important**: This guide complements AMD's official installation documentation[^3] with practical insights from hands-on deployment. Where in doubt or if you need most current package version, **always refer to AMD's official resources**.

## Prerequisites

System requirements:
- **Operating System**: Oracle Linux 9.6
- **Kernel Version**: `5.14.0-570.x` or newer
- **Hardware**: AMD MI300X GPUs detected in firmware inventory
- **User Privileges**: Root or `sudo` access for kernel module installation

It is critical to update the system[^4] and make sure it runs a supported kernel version. In this case:

```shell
$ sudo dnf update --releasever=9.6 --exclude=\*release\*

# Kernel development packages required for DKMS compilation
$ sudo dnf -y install kernel-headers kernel-modules-core kernel-devel

# Reboot to load the updated kernel (if needed)
$ sudo reboot now
``` 

After reboot, verify the kernel version:

```shell
$ uname -srmv
Linux 5.14.0-570.30.1.0.1.el9_6.x86_64 #1 SMP PREEMPT_DYNAMIC Mon Jul 28 14:23:15 PDT 2025 x86_64
```

### Additional Dependencies (Optional)

Install Python and environment management tools:
```shell
$ sudo dnf install python3-setuptools python3-wheel
# ...
Installed:
  python3-wheel-1:0.36.2-8.el9.noarch
#
$ sudo dnf install environment-modules
# ...
Installed:
  environment-modules-5.3.0-1.el9.x86_64                                                 tcl-1:8.6.10-7.el9.x86_64
```

### Additional GRUB Configuration (Optional)

Besides not explicitely mentioned in the AMD's official installation docs[^3], it worth checking the GRUB configuration according to AMD's basic system health checks documentation[^5]:
- `pci=realloc=off`
- `intel_iommu=on` (for this specific server having Intel Xeon CPUs)
- `iommu=pt`

Current configuration:
```shell
$ cat /proc/cmdline
BOOT_IMAGE=(hd6,gpt2)/vmlinuz-5.14.0-570.30.1.0.1.el9_6.x86_64 root=/dev/mapper/os-root ro crashkernel=1G-64G:448M,64G-:512M resume=/dev/mapper/os-swap rd.lvm.lv=os/root rd.lvm.lv=os/swap selinux=0
```

Above configuration shows **none of the recommended parameters**, which means the system is running with:
- **PCI resource reallocation enabled** (default)
- **IOMMU disabled** (default)
- **No passthrough IOMMU mode**

While the system will probably work for basic ROCm operations, the missing parameters create several risks:
- **GPU Recognition Issues**: AMD speciically warns that without `pci=realloc=off`, "not all GPUs may be recognized"[^5]
- **Inefficient Memory Access**: Without IOMMU passthrough (`iommu=pt`), GPU memory operations go through additional translation layers
- High-bandwidth workloads may experience inconsistent behaviour
- **Reduced GPU-to-GPU Communication**:  xGMI Infinity Fabric links may not operate optimally
- **Memory Bandwidth Degradation** resulting from less efficient DMA operations between GPUs and system memory
- **Increased Latency** because of additional memory translation overhead

Fixing this is particularly critical of a 8-GPU configuration with its complex xGMI mesh topology.

In most cases one needs to apply the change to `/etc/default/grub` and regenerate `/boot/grub2/grub.cfg`, but the test server is **booting in UEFI mode**:

```shell
$ sudo find /boot -name "grub.cfg" -o -name "grub2.cfg"
/boot/efi/EFI/redhat/grub.cfg
/boot/grub2/grub.cfg
```

It is imperative to regenerate `/boot/efi/EFI/redhat/grub.cfg` in this case:

```shell
# Edit GRUB defaults
sudo vi /etc/default/grub

# Add missing parameters to GRUB_CMDLINE_LINUX line:
GRUB_CMDLINE_LINUX="crashkernel=1G-64G:448M,64G-:512M selinux=0 pci=realloc=off intel_iommu=on iommu=pt"

# Generate GRUB config for UEFI boot (the one for this Dell PowerEdge XE9680 server)
sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
```

Additionally, Oracle Linux 9 uses **Boot Loader Specification (BLS)**, where kernel parameters are managed separately from the main GRUB configuration. The `kernelopts` variable in GRUB needs to be synchronized with the BLS entries:

```shell
# Update with grubby
sudo grubby --update-kernel=ALL --args="pci=realloc=off intel_iommu=on iommu=pt"

# Verify the parameters were added
sudo grubby --info=ALL | grep args

# Regenerate GRUB configuration to sync with BLS
sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg

# Reboot to apply changes
sudo reboot
```

After rebooting, verify the configuration:
```shell
# Should now show all three AMD parameters
cat /proc/cmdline | grep -o "pci=realloc=off\|intel_iommu=on\|iommu=pt"
pci=realloc=off
intel_iommu=on
iommu=pt
```

## ROCm Installation Methods

### Method 1: Online Installation

If the target server has internet access, use AMD's package manager approach[^6]. This is definitively the simplest installation.

```shell
# Add AMD's ROCm repository
sudo tee /etc/yum.repos.d/rocm.repo <<EOF
[ROCm-6.4.2]
name=ROCm6.4.2
baseurl=https://repo.radeon.com/rocm/el9/6.4.2/main
enabled=1
priority=50
gpgcheck=1
gpgkey=https://repo.radeon.com/rocm/rocm.gpg.key
EOF

# Install ROCm metapackage (includes all essential components)
sudo dnf update
sudo dnf install rocm
```

**Advantages**: Automatic dependency resolution, official support, and easy updates

**Requirements**: Internet access

### Method 2: Offline Installation

Enterprise environment often requires offline installation due to security policies or network restrictions. AMD provides the ROCm offline installer creator [^7], which works great if you have a server **with the same Linux distribution, release version, and Linux kernel version**, with internet connection.

If that is not the case, then only option available is to manually download and install all required packages.

#### When to use this method

- **Air-gapped environments** without Internet access
- **Custom kernel versions** not supported by AMD's offline installer
- **Lack of similar sever configuration with intenet access** to run the AMD's offiline installer
- **Corporate security policies** requiring manual package validation

#### Prerequisites

- **Download machine** with Internet access to download packages
- **Transfer method**, such as SCP

> **Important**: this is definitively a much more complex setup, with manual dependency management, but gives you full control over package versions and installation process

## Package Download and Organization

### Repository Sources

Required packages are distributd across two main repositories:
- **AMDGPU Drivers**: `https://repo.radeon.com/amdgpu/6.4.2/el/9.6/main/x86_64/`
- **ROCm Software**: `https://repo.radeon.com/rocm/el9/rpm/`

### Package Categories

#### 1. Essential Syste Components

These packages provide GPU detection and ROCm core functionality:
- Core ROCm runtime (must install first):
    - `rocm-core`: Provides fundamental ROCm system services
    - [`hsa-rocr`](https://github.com/ROCm/ROCR-Runtime): HSA (Heterogeneous System Architecture) runtime for GPU compute operations
    - [`rocm-device-libs`](https://github.com/ROCm/llvm-project/tree/amd-staging/amd/device-libs): GPU-specific optimized libraries
- AMDGPU kernel drivers (critical for hardware detection):
  - [`amdgpu-core`](https://github.com/ROCm/amdgpu)
  - `amdgpu-dkms`
  - `amdgpu-dkms-firmware`

```shell
# Core ROCm runtime
wget https://repo.radeon.com/rocm/el9/6.4.2/main/rocm-core-6.4.2.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/6.4.2/main/hsa-rocr-1.15.0.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/6.4.2/main/rocm-device-libs-1.0.0.60402-120.el9.x86_64.rpm

# AMDGPU kernel drivers
wget https://repo.radeon.com/amdgpu/6.4.2/el/9.6/main/x86_64/amdgpu-core-6.4.60402-2187269.el9.noarch.rpm
wget https://repo.radeon.com/amdgpu/6.4.2/el/9.6/main/x86_64/amdgpu-dkms-6.12.12-2187269.el9.noarch.rpm
wget https://repo.radeon.com/amdgpu/6.4.2/el/9.6/main/x86_64/amdgpu-dkms-firmware-6.12.12-2187269.el9.noarch.rpm
```

#### 2. GPU Compute Runtime

Required for running GPU applications:
- HIP runtime:
  - `hip-runtime-amd`: CUDA-like programming model for AMD GPUs
  - `rocm-hip-runtime`
  - `rocm-language-runtime`: Support for various programming languages on AMD GPUs
- System management:
  - `rocm-smi-lib`: System management interface for monitoring GPU status
  - [`rocminfo`](https://github.com/ROCm/rocminfo): Gives information about the HSA system attributes and agents
  - [`amd-smi`](https://rocm.blogs.amd.com/software-tools-optimization/amd-smi-overview/README.html): Future replacement of `rocm-smi` 

```shell
# HIP runtime
wget https://repo.radeon.com/rocm/el9/6.4.2/main/hip-runtime-amd-6.4.43484.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/6.4.2/main/rocm-hip-runtime-6.4.2.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/6.4.2/main/rocm-language-runtime-6.4.2.60402-120.el9.x86_64.rpm

# System management
wget https://repo.radeon.com/rocm/el9/6.4.2/main/rocm-smi-lib-7.5.0.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/6.4.2/main/rocminfo-1.0.0.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/rpm/amd-smi-lib-25.5.1.60402-120.el9.x86_64.rpm
```

#### 3. Development and Compilation Tools

For building GPU applications and custom kernels:
- [`comgr`](https://github.com/ROCm/llvm-project/tree/amd-staging/amd/comgr): Code object manager for GPU kernels
- [`hipcc`](https://github.com/ROCm/llvm-project/tree/amd-staging/amd/hipcc): [HIP](https://rocm.docs.amd.com/projects/HIP/en/docs-develop/what_is_hip.html) (Heterogeneous-computing Interface for Portability) compiler wrapper for GPU code compilation
- `rocm-llvm`: Compiler infrastructure optimized for AMD GPUs
- `hip-devel`

```shell
wget https://repo.radeon.com/rocm/el9/6.4.2/main/comgr-3.0.0.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/6.4.2/main/hipcc-1.1.1.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/rpm/rocm-llvm-19.0.0.25224.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/6.4.2/main/hip-devel-6.4.43484.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/rpm/hip-doc-6.4.43484.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/rpm/hip-samples-6.4.43484.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/rpm/hipify-clang-19.0.0.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/rpm/rocm-cmake-0.14.0.60402-120.el9.x86_64.rpm
```

#### 4. AI/ML Libraries

Optimized libraries for AI/ML workloads:
- [`rocblas`](https://github.com/ROCm/rocm-libraries/tree/develop/projects/rocblas): High-performance BLAS (Basic Linear Algebra Subprograms) operations on GPU
- [`hipblaslt`](https://github.com/ROCm/rocm-libraries/tree/develop/projects/hipblaslt): Optimized GEMM (General Matrix-Multiply) operations for AI workloads
- [`miopen-hip`](https://github.com/ROCm/rocm-libraries/tree/develop/projects/miopen): Deep learning primitives library (like cuDNN)
- [`rccl`](https://github.com/ROCm/rccl): Collective communications for multi-GPU training
- [`rocrand`](https://github.com/ROCm/rocm-libraries/tree/develop/projects/rocrand): Used to generate pseudorandom and quasirandom numbers
- [`rocsolver`](https://github.com/ROCm/rocsolver): Implementation of a subset of [LAPACK](https://www.netlib.org/lapack/) 

```shell
# Linear algebra and deep learning
wget https://repo.radeon.com/rocm/el9/rpm/rocblas-4.4.1.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/rpm/hipblaslt-0.12.1.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/6.4.2/main/miopen-hip-3.4.0.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/rpm/rccl-2.22.3.60402-120.el9.x86_64.rpm

# Additional libraries
wget https://repo.radeon.com/rocm/el9/rpm/rocrand-3.3.0.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/rpm/rocsolver-3.28.2.60402-120.el9.x86_64.rpm
```

#### 5. Performance Analysis Tooling

For benchmarking and profiling:
- Benchmarking utilities:
  - [`rocm-bandwidth-test`](https://github.com/ROCm/rocm_bandwidth_test): Used to capture the performance characteristics of buffer copying and kernel read and write operations
  - [`transferbench-devel`](https://github.com/ROCm/TransferBench): Utility for benchmarking simultaneous copies between user-specified CPU and GPU devices
- Profiling and tracing:
  - [`rocprofiler-sdk`](https://github.com/ROCm/rocprofiler-sdk):  Tooling for hardware-specific low-level performance analysis
  - [`roctracer`](https://github.com/ROCm/roctracer): Libraries to help tracing applications in the runtime
  - [`hsa-amd-aqlprofile`](https://github.com/ROCm/aqlprofile): Library for advanced GPU profiling and tracing on AMD platforms
  - `hiprand`: requirement for the package `rocm-validation-suite`
  - [`rocm-validation-suite`](https://github.com/ROCm/ROCmValidationSuite): System validation and diagnostics tool for monitoring and stress testing  

```shell
# Benchmarking utilities
wget https://repo.radeon.com/rocm/el9/6.4.2/main/rocm-bandwidth-test-1.4.0.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/6.4.2/main/transferbench-devel-1.62.0.60402-120.el9.x86_64.rpm

# Profiling and tracing
wget https://repo.radeon.com/rocm/el9/6.4.2/main/rocprofiler-sdk-0.6.0.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/rpm/roctracer-4.1.60402.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/rpm/hsa-amd-aqlprofile-1.0.0.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/rpm/rocprofiler-sdk-roctx-0.6.0.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/rpm/hiprand-2.12.0.60402-120.el9.x86_64.rpm
wget https://repo.radeon.com/rocm/el9/rpm/rocm-validation-suite-1.1.0.60402-120.el9.x86_64.rpm
```

### Package Transfer

Transfer all downloaded packages to the target server:

```shell
ssh $username@$target_server "mkdir -p /tmp/rocm-packages"
scp -p *.rpm $username@$target_server:/tmp/rocm-packages/
```

## Installation Sequence

The installation must follow a specific order:
1. **Core first**: ROCm services must exist before GPU drivers register with them
2. **Drivers require reboot**: Kenel modules need system restart to load properly
3. **Runtime depends on drivers**: HIP and other compute layers require working GPU drivers

### Phase 1: Core ROCm Components

```shell
cd /tmp/rocm-packages

sudo dnf install -y \
  rocprofiler-register-0.4.0.60402-120.el9.x86_64.rpm \
  rocm-core-6.4.2.60402-120.el9.x86_64.rpm \
  hsa-rocr-1.15.0.60402-120.el9.x86_64.rpm \
  rocm-device-libs-1.0.0.60402-120.el9.x86_64.rpm
```

### Phase 2: GPU Kernel Drivers

These drivers enable hardware-level GPU access:

```shell
# Install AMDGPU package only - THIS IS ESSENTIAL for MI300X recognition
sudo dnf install -y amdgpu-install-6.4.60402-1.el9.noarch.rpm

# Install Kernel Drivers (compilation occurs here)
sudo dnf install -y \
  amdgpu-core-6.4.60402-2187269.el9.noarch.rpm \
  amdgpu-dkms-6.12.12-2187269.el9.noarch.rpm \
  amdgpu-dkms-firmware-6.12.12-2187269.el9.noarch.rpm
```

At this point the installation process should have completed these critical steps:
1. **Kernel Module Compilation**: The AMDGPU drivers built for the current kernel (`5.14.0-570.30.1.0.1.el9_6.x86_64`)
2. **Module Signing**: All kernel modules properly signed with a self-generated MOK (Machine Owner Key)
3. **Driver Installation**: Seven essential modules installed:
  - `amdgpu.ko` - Main GPU driver
  - `amdttm.ko` - GPU memory management
  - `amdkcl.ko` - Kernel compatibility layer
  - `amd-sched.ko` - GPU scheduler
  - `amddrm_ttm_helper.ko` - DRM helper functions
  - `amddrm_buddy.ko` - Memory buddy allocator
  - `amdxcp.ko` - Cross-partition communication

Verification steps before rebooting:
```shell
# Check if the modules were properly installed
$ ls -la /lib/modules/$(uname -r)/extra/amd*
-rw-r--r-- 1 root root    9156 Jul 30 16:54 /lib/modules/5.14.0-570.30.1.0.1.el9_6.x86_64/extra/amddrm_buddy.ko.xz
-rw-r--r-- 1 root root    2768 Jul 30 16:54 /lib/modules/5.14.0-570.30.1.0.1.el9_6.x86_64/extra/amddrm_ttm_helper.ko.xz
-rw-r--r-- 1 root root 3747708 Jul 30 16:54 /lib/modules/5.14.0-570.30.1.0.1.el9_6.x86_64/extra/amdgpu.ko.xz
-rw-r--r-- 1 root root   11380 Jul 30 16:54 /lib/modules/5.14.0-570.30.1.0.1.el9_6.x86_64/extra/amdkcl.ko.xz
-rw-r--r-- 1 root root   20996 Jul 30 16:54 /lib/modules/5.14.0-570.30.1.0.1.el9_6.x86_64/extra/amd-sched.ko.xz
-rw-r--r-- 1 root root   36212 Jul 30 16:54 /lib/modules/5.14.0-570.30.1.0.1.el9_6.x86_64/extra/amdttm.ko.xz
-rw-r--r-- 1 root root    2732 Jul 30 16:54 /lib/modules/5.14.0-570.30.1.0.1.el9_6.x86_64/extra/amdxcp.ko.xz

# Verify module dependencies were generated
$ ls -la /lib/modules/$(uname -r)/modules.dep
-rw-r--r-- 1 root root 114523 Jul 30 16:55 /lib/modules/5.14.0-570.30.1.0.1.el9_6.x86_64/modules.dep

# Check DKMS status
$ sudo dkms status
amdgpu/6.12.12-2187269.el9, 5.14.0-570.30.1.0.1.el9_6.x86_64, x86_64: installed
```

Now go and **reboot** the server to load the new kernel modules.

After reboot, verify AMD modules are loaded:
```shell
# Check if AMDGPU drivers are loaded
$ lsmod | grep amdgpu
amdgpu              15712256  0
amddrm_ttm_helper      12288  1 amdgpu
amdttm                114688  2 amdgpu,amddrm_ttm_helper
amddrm_buddy           24576  1 amdgpu
amdxcp                 12288  1 amdgpu
i2c_algo_bit           16384  1 amdgpu
drm_ttm_helper         16384  1 amdgpu
drm_exec               16384  1 amdgpu
drm_suballoc_helper    12288  1 amdgpu
video                  77824  1 amdgpu
amd_sched              69632  1 amdgpu
amdkcl                 36864  3 amd_sched,amdttm,amdgpu
drm_display_helper    299008  1 amdgpu
drm_kms_helper        266240  4 drm_display_helper,amdgpu,drm_ttm_helper,amdkcl
cec                    69632  2 drm_display_helper,amdgpu
drm                   811008  13 drm_kms_helper,drm_exec,amd_sched,amdttm,drm_suballoc_helper,drm_display_helper,amdgpu,drm_ttm_helper,amddrm_buddy,ttm,amddrm_ttm_helper,amdxcp

# Check for GPU devices
$ lspci | grep -i MI300X
1b:00.0 Processing accelerators: Advanced Micro Devices, Inc. [AMD/ATI] Aqua Vanjaram [Instinct MI300X]
3d:00.0 Processing accelerators: Advanced Micro Devices, Inc. [AMD/ATI] Aqua Vanjaram [Instinct MI300X]
4e:00.0 Processing accelerators: Advanced Micro Devices, Inc. [AMD/ATI] Aqua Vanjaram [Instinct MI300X]
5f:00.0 Processing accelerators: Advanced Micro Devices, Inc. [AMD/ATI] Aqua Vanjaram [Instinct MI300X]
9d:00.0 Processing accelerators: Advanced Micro Devices, Inc. [AMD/ATI] Aqua Vanjaram [Instinct MI300X]
bd:00.0 Processing accelerators: Advanced Micro Devices, Inc. [AMD/ATI] Aqua Vanjaram [Instinct MI300X]
cd:00.0 Processing accelerators: Advanced Micro Devices, Inc. [AMD/ATI] Aqua Vanjaram [Instinct MI300X]
dd:00.0 Processing accelerators: Advanced Micro Devices, Inc. [AMD/ATI] Aqua Vanjaram [Instinct MI300X]
```

The above output indicates the **AMDGPU kernel driver installation was completely successful** and the MI300X GPUs are now properly detected at the hardware level. 

### Phase 3: Runtime and Development Tools

After reboot, install remaining components:

```shell
sudo dnf install -y \
  comgr-3.0.0.60402-120.el9.x86_64.rpm \
  rocminfo-1.0.0.60402-120.el9.x86_64.rpm \
  rocm-smi-lib-7.5.0.60402-120.el9.x86_64.rpm \
  amd-smi-lib-25.5.1.60402-120.el9.x86_64.rpm \
  rocm-language-runtime-6.4.2.60402-120.el9.x86_64.rpm \
  hip-runtime-amd-6.4.43484.60402-120.el9.x86_64.rpm \
  rocm-hip-runtime-6.4.2.60402-120.el9.x86_64.rpm \
  rocprofiler-sdk-roctx-0.6.0.60402-120.el9.x86_64.rpm \
  hsa-amd-aqlprofile-1.0.0.60402-120.el9.x86_64.rpm \
  rocprofiler-sdk-0.6.0.60402-120.el9.x86_64.rpm \
  rocm-bandwidth-test-1.4.0.60402-120.el9.x86_64.rpm \
  transferbench-devel-1.62.0.60402-120.el9.x86_64.rpm \
  roctracer-4.1.60402.60402-120.el9.x86_64.rpm \ 
  hipblaslt-0.12.1.60402-120.el9.x86_64.rpm \
  rocblas-4.4.1.60402-120.el9.x86_64.rpm \
  rocrand-3.3.0.60402-120.el9.x86_64.rpm \
  miopen-hip-3.4.0.60402-120.el9.x86_64.rpm \
  rocm-llvm-19.0.0.25224.60402-120.el9.x86_64.rpm \
  hipcc-1.1.1.60402-120.el9.x86_64.rpm \
  hip-devel-6.4.43484.60402-120.el9.x86_64.rpm \
  hip-doc-6.4.43484.60402-120.el9.x86_64.rpm \
  hip-samples-6.4.43484.60402-120.el9.x86_64.rpm \
  hipify-clang-19.0.0.60402-120.el9.x86_64.rpm \
  rocm-cmake-0.14.0.60402-120.el9.x86_64.rpm \
  rocm-hip-runtime-devel-6.4.2.60402-120.el9.x86_64.rpm \
  miopen-hip-devel-3.4.0.60402-120.el9.x86_64.rpm \
  rccl-2.22.3.60402-120.el9.x86_64.rpm \
  hiprand-2.12.0.60402-120.el9.x86_64.rpm \
  rocm-validation-suite-1.1.0.60402-120.el9.x86_64.rpm
```

### 4. Environment Setup

Last, ROCm binaries need to be added to the PATH:

```shell
echo 'export PATH=/opt/rocm/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

## Verification and Testing

### Quick Checks

Verify all 8 MI300X GPUs are detected:

```shell
# Check what ROCm detects
$ rocminfo | grep -A 20 "Agent"
# Output shows 10 agents in total:
# - Agent 1 & 2: Intel CPUs
# - Agent 3-10: 8 MI300X GPUs
HSA Agents
==========
*******
Agent 1
*******
  Name:                    INTEL(R) XEON(R) PLATINUM 8562Y+
# ... (2 CPU agents)
--
Agent 3
*******
  Name:                    gfx942
  Uuid:                    GPU-72be95cb49b27a64
  Marketing Name:          AMD Instinct MI300X
  Vendor Name:             AMD
  Feature:                 KERNEL_DISPATCH
  Profile:                 BASE_PROFILE
  Float Round Mode:        NEAR
  Max Queue Number:        128(0x80)
  Queue Min Size:          64(0x40)
  Queue Max Size:          131072(0x20000)
  Queue Type:              MULTI
  Node:                    2
  Device Type:             GPU
  Cache Info:
    L1:                      32(0x20) KB
    L2:                      4096(0x1000) KB
    L3:                      262144(0x40000) KB
  Chip ID:                 29857(0x74a1)
  ASIC Revision:           1(0x1)
# ... (8 GPU agents)

# Check GPUs are detected
$ rvs -g

ROCm Validation Suite (version 1.1.0)

Supported GPUs available:
0000:1b:00.0 - GPU[ 2 - 55354] AMD Instinct MI300X (Device 29857)
0000:3d:00.0 - GPU[ 3 - 41632] AMD Instinct MI300X (Device 29857)
0000:4e:00.0 - GPU[ 4 - 47045] AMD Instinct MI300X (Device 29857)
0000:5f:00.0 - GPU[ 5 - 60169] AMD Instinct MI300X (Device 29857)
0000:9d:00.0 - GPU[ 6 - 56024] AMD Instinct MI300X (Device 29857)
0000:bd:00.0 - GPU[ 7 -   705] AMD Instinct MI300X (Device 29857)
0000:cd:00.0 - GPU[ 8 - 59108] AMD Instinct MI300X (Device 29857)
0000:dd:00.0 - GPU[ 9 - 10985] AMD Instinct MI300X (Device 29857)

# Same but with rocm-smi, which provides more information
$ rocm-smi | head -20

============================================ ROCm System Management Interface ============================================
====================================================== Concise Info ======================================================
Device  Node  IDs              Temp        Power     Partitions          SCLK    MCLK    Fan  Perf  PwrCap  VRAM%  GPU%
              (DID,     GUID)  (Junction)  (Socket)  (Mem, Compute, ID)
==========================================================================================================================
0       2     0x74a1,   55354  51.0°C      152.0W    NPS1, SPX, 0        134Mhz  900Mhz  0%   auto  750.0W  0%     0%
1       3     0x74a1,   41632  48.0°C      146.0W    NPS1, SPX, 0        134Mhz  900Mhz  0%   auto  750.0W  0%     0%
2       4     0x74a1,   47045  48.0°C      147.0W    NPS1, SPX, 0        134Mhz  900Mhz  0%   auto  750.0W  0%     0%
3       5     0x74a1,   60169  53.0°C      151.0W    NPS1, SPX, 0        133Mhz  900Mhz  0%   auto  750.0W  0%     0%
4       6     0x74a1,   56024  50.0°C      153.0W    NPS1, SPX, 0        133Mhz  900Mhz  0%   auto  750.0W  0%     0%
5       7     0x74a1,   705    46.0°C      141.0W    NPS1, SPX, 0        133Mhz  900Mhz  0%   auto  750.0W  0%     0%
6       8     0x74a1,   59108  50.0°C      149.0W    NPS1, SPX, 0        134Mhz  900Mhz  0%   auto  750.0W  0%     0%
7       9     0x74a1,   10985  46.0°C      145.0W    NPS1, SPX, 0        134Mhz  900Mhz  0%   auto  750.0W  0%     0%
==========================================================================================================================
================================================== End of ROCm SMI Log ===================================================
```

### Check GPU Link Speed and Width on PCIe Bus

Another interesting check is confirming PCIe links to each of the GPUs are running at full speed and width[^8]:
```shell
$ sudo lspci -d 1002:74a1 -vvv | grep -e DevSta -e LnkSta
                DevSta: CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
                LnkSta: Speed 32GT/s (ok), Width x16 (ok)
                LnkSta2: Current De-emphasis Level: -3.5dB, EqualizationComplete- EqualizationPhase1-
                DevSta: CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
                LnkSta: Speed 32GT/s (ok), Width x16 (ok)
                LnkSta2: Current De-emphasis Level: -3.5dB, EqualizationComplete- EqualizationPhase1-
                DevSta: CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
                LnkSta: Speed 32GT/s (ok), Width x16 (ok)
                LnkSta2: Current De-emphasis Level: -3.5dB, EqualizationComplete- EqualizationPhase1-
                DevSta: CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
                LnkSta: Speed 32GT/s (ok), Width x16 (ok)
                LnkSta2: Current De-emphasis Level: -3.5dB, EqualizationComplete- EqualizationPhase1-
                DevSta: CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
                LnkSta: Speed 32GT/s (ok), Width x16 (ok)
                LnkSta2: Current De-emphasis Level: -3.5dB, EqualizationComplete- EqualizationPhase1-
                DevSta: CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
                LnkSta: Speed 32GT/s (ok), Width x16 (ok)
                LnkSta2: Current De-emphasis Level: -3.5dB, EqualizationComplete- EqualizationPhase1-
                DevSta: CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
                LnkSta: Speed 32GT/s (ok), Width x16 (ok)
                LnkSta2: Current De-emphasis Level: -3.5dB, EqualizationComplete- EqualizationPhase1-
                DevSta: CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
                LnkSta: Speed 32GT/s (ok), Width x16 (ok)
                LnkSta2: Current De-emphasis Level: -3.5dB, EqualizationComplete- EqualizationPhase1-
```

Output shows `LinkSta` displays link speed is `32GT/s` and width is `x16`. This indicates GPUs are running at **PCIe Gen 5.0 specification**, which is exactly what MI300X accelerators expect.

> **Note**: Device ID `0x74a1` is shown in both `rocminfo` and `rocm-smi` outputs from previous section.

### System Validations

It worth running the system validation checks[^9] to determine if the GPUs have optimal performance:

```shell
$ export RVS_CONF=/opt/rocm/share/rocm-validation-suite/conf

# This is validation check that shouldn't fail at this point, but still worth trying
$ rvs -c ${RVS_CONF}/gpup_single.conf

# Stress test for GPU FLOPS performance for SGEMM, DGEMM and HGEMM operations and computes and displays peak GFLOPs/s
$ rvs -c ${RVS_CONF}/MI300X/gst_single.conf
# Truncated output follows. Showing one test only
[RESULT] [  2455.547500] Action name :gst-639Tflops-4K4K8K-rand-fp16
[RESULT] [  2455.547517] Module name :gst
[RESULT] [  2456.47741 ] [gst-639Tflops-4K4K8K-rand-fp16] [GPU:: 55354] Start of GPU ramp up
[RESULT] [  2461.658482] [gst-639Tflops-4K4K8K-rand-fp16] [GPU:: 55354] GFLOPS 738501
[RESULT] [  2462.659172] [gst-639Tflops-4K4K8K-rand-fp16] [GPU:: 55354] End of GPU ramp up
[RESULT] [  2465.759602] [gst-639Tflops-4K4K8K-rand-fp16] [GPU:: 55354] GFLOPS 736974
[RESULT] [  2468.853149] [gst-639Tflops-4K4K8K-rand-fp16] [GPU:: 55354] GFLOPS 738615
[RESULT] [  2471.968716] [gst-639Tflops-4K4K8K-rand-fp16] [GPU:: 55354] GFLOPS 733394
[RESULT] [  2475.79689 ] [gst-639Tflops-4K4K8K-rand-fp16] [GPU:: 55354] GFLOPS 734481
[RESULT] [  2477.764871] [gst-639Tflops-4K4K8K-rand-fp16] [GPU:: 55354] GFLOPS 738615
[RESULT] [  2477.764883] [gst-639Tflops-4K4K8K-rand-fp16] [GPU:: 55354] GFLOPS 738615 Target GFLOPS: 639000 met: TRUE
```

This is the output of `and-smi` showing how a GPU was at peak during stress testing:

```shell
$ amd-smi monitor --watch 5
'CTRL' + 'C' to stop watching output:
 TIMESTAMP  GPU  POWER   GPU_T   MEM_T   GFX_CLK   GFX%   MEM%   ENC%   DEC%      VRAM_USAGE
1753969973    0  750 W   69 °C   50 °C  1278 MHz  100 %    1 %    N/A    0 %    2.0/192.0 GB
1753969973    1  178 W   47 °C   38 °C  2105 MHz    0 %    0 %    N/A    0 %    1.7/192.0 GB
1753969973    2  178 W   47 °C   41 °C  2102 MHz    0 %    0 %    N/A    0 %    1.7/192.0 GB
1753969973    3  182 W   51 °C   43 °C  2104 MHz    0 %    0 %    N/A    0 %    1.7/192.0 GB
1753969973    4  188 W   50 °C   45 °C  2102 MHz    0 %    0 %    N/A    0 %    1.7/192.0 GB
1753969973    5  173 W   47 °C   40 °C  2099 MHz    0 %    0 %    N/A    0 %    1.7/192.0 GB
1753969973    6  184 W   52 °C   46 °C  2108 MHz    0 %    0 %    N/A    0 %    1.7/192.0 GB
1753969973    7  181 W   48 °C   42 °C  2101 MHz    0 %    0 %    N/A    0 %    1.7/192.0 GB
```

We can observe stress test was compute-only from above output.

### xGMI Throughput Performance

```shell
$ rvs -c ${RVS_CONF}/gm_single.conf
[RESULT] [  3349.119787] Action name :metrics_monitor
[RESULT] [  3349.185433] Module name :gm
json log file is /var/tmp/rvs_1753970507889.json
# (truncated output)
# Temperature violations: 0 across all 8 GPUs ✅
# Clock violations: 0 across all 8 GPUs ✅
# Memory clock violations: 0 across all 8 GPUs ✅
# Fan violations: 0 across all 8 GPUs ✅
# Power violations: 0 across all 8 GPUs ✅

$ tail var/tmp/rvs_1753970507889.json
# (truncated output)
{
    "srcgpu" : "10985",
    "dstgpu" : "41632",
    "intf" : "xGMI",
    "throughput" : "91.832 GBps",
    "pass" : "true"
  },
{
    "srcgpu" : "10985",
    "dstgpu" : "47045",
    "intf" : "xGMI",
    "throughput" : "93.274 GBps",
    "pass" : "true"
  },
{
    "srcgpu" : "10985",
    "dstgpu" : "60169",
    "intf" : "xGMI",
    "throughput" : "92.394 GBps",
    "pass" : "true"
  }
```

From **Appendix D - xGMI Implementation Details**[^1]:
- **Practical xGMI bandwidth**: 45-48 GB/s per direction
- **Bidirectional capability**: 128GB/s theoretical per link
- **315-336 GB/s aggregate bandwidth per GPU** (7 links × 45-48 GB/s each)

The ~92 GB/s average xGMI throughput represents excellent real-world performance:
- **Theoretical maximum**: 128 GB/s bidirectional per link[8]
- **Measured performance**: 92.38 GB/s (**72% efficiency**)
- **Industry typical**: 60-80 GB/s practical throughput

This performance validates that system system optimization (GRUB configuration, ROCm installation) achieved optimal hardware utilization.

### Verify multi-GPU communication

TODO

## Understanding the MI300X Installation

After sucessful installation, the system creates specific device interfaces for GPU access. Understanding these interfaces is paramount for troubleshooting and optimization.

### Device Interface Overview

```shell
$ ls -la /dev/dri/
crw-rw----  1 root video  226,   0 Jul 30 17:07 card0
crw-rw----  1 root video  226,  63 Jul 30 17:07 card1
# ... (64 total card devices)
crw-rw-rw-  1 root render 226, 128 Jul 30 17:07 renderD128
crw-rw-rw-  1 root render 226, 191 Jul 30 17:07 renderD129
# ... (64 total render devices)
```

### Card Devices (Management Interface)

The **64 card devices** (`card0` through `card63`) provide **management and monitoring** access. These are used by `roc-smi` for system management interface.

**Why 64 cards if there are 8 MI300X GPUs?** As explained in the first part[^1], each MI300X contains **8 XCDs (**Accelerated Compute Dies**), and each XCD can present itself as a separete card device for certain management operations. That gives a total of **8 MI300X GPUs × 8 XCDs per GPU = 64 card devices**.

### Render Nodes (Compute Interface)

The **64 render devices** (`renderD128` through `renderD191`) provide compute access. These are used by `rocminfo`, HIP, ROCm applications, and AI frameworks.

## Troubleshooting Common Issues

1. **Missing dependencies when installing a package**: search for the package in ROCm repository, download, transfer, and install it first
2. **Cannot access devices**: verify `groups $USER` include `video` and `render`
3. **Not showing all GPUs**: check your GRUB configuration. See section [Additional Grub Configuration](#additional-grub-configuration-optional)

## Next Steps

With ROCm successfully installed and validated, the system is now ready for comprehensive performance evaluation. Here are some possible continuations:
- **Performance Benchmarking**
  - **Memory bandwidth analysis** using `rocm-bandwidth-test` to measure HBM3 performance across all 8 GPUs
  - **Inter-GPU communication testing** with `TransferBench` to validate the fully connected xGMI mesh topology
  - **Multi-GPU scaling analysis** using RCCL (ROCm Collective Communications Library) for distributed workloads
- **AI Framework Integration**
    - **PyTorch ROCm installation** and optimization for large language model training
    - **vLLM deployment** for high-throughput LLM inference serving
    - **Performance validation** using AMD's optimized Docker containers and benchmarking suites
- **Advanced System Optimization**
    - **Compute and memory partitioning** using CPX and NPS4 modes for specific workload optimization
    - **NUMA affinity tuning** for optimal CPU-GPU data locality
    - **Power and thermal monitoring** during sustained AI workloads
- **Real-World Workload Validation**
    - **Large language model training** with models exceeding 70B parameters
    - **Multi-node communication** setup for distributed training scenarios
    - **Production deployment patterns** for enterprise AI inference serving

## References

[^1]: https://github.com/cmontemuino/tech-knowledge-base/tree/main/blog-posts/amd-mi300x-hardware-exploration
[^2]: https://rocm.docs.amd.com/en/latest/what-is-rocm.html
[^3]: https://rocm.docs.amd.com/projects/install-on-linux/en/latest/install/quick-start.html
[^4]: https://rocm.docs.amd.com/projects/install-on-linux/en/latest/install/prerequisites.html#update-your-enterprise-linux
[^5]: https://instinct.docs.amd.com/projects/system-acceptance/en/latest/mi300x/health-checks.html#check-kernel-boot-arguments
[^6]: https://rocm.docs.amd.com/projects/install-on-linux/en/latest/install/install-methods/package-manager-index.html
[^7]: https://rocm.docs.amd.com/projects/install-on-linux/en/latest/install/rocm-offline-installer.html
[^8]: https://instinct.docs.amd.com/projects/system-acceptance/en/latest/mi300x/health-checks.html#check-gpu-link-speed-and-width-on-pcie-bus
[^9]: https://instinct.docs.amd.com/projects/system-acceptance/en/latest/mi300x/system-validation.html