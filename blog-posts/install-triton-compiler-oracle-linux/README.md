# Installing Triton Compiler on Oracle Linux: A Practical Guide

Oracle Linux is an excellent choice because of stability for production environments. However, it can present unique challenges because of its conservative approach to system library versions, with Oracle favoring stability over having the latest library versions. This is the case of modern deep learning frameworks like Triton[^1], which require newer C++ standard libraries.

The primary challenge originates from **GLIBCXX version mistmatches** between Triton's pre-compiled LLVM[^2] tools and Oracle Linux's conservative system libraries. 

This guide provides two tested approaches to successfully install Triton on Oracle Linux 9.6: using compatible Triton versions and building custom LLVM components. Therefore, you can choose the approach that best fits with your environment and requirements.

## Prerequisites and Environment Setup

Before starting, ensure your Oracle Linux meets the requirements:

### System Requirements
- Oracle Linux 9.6 (`x86_64`)
- Python 3.12+ environment
- Administrative privileges for package installation

### Essential Development Tools

```shell
# Install core development tools
sudo dnf groupinstall "Development Tools"

# Install specific compiler components
sudo dnf install gcc gcc-c++ cmake make kernel-devel

# Verify installation
gcc --version
g++ --version
cmake --version
```

### Python Environment Setup

```shell
# Install uv for efficient Python environment management
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create isolated environment
uv venv --python 3.12 --seed
source .venv/bin/activate
```

## Installation Approaches: Choose Your Strategy

Triton's latest versions (`v3.4.0+`) require `GLIBCXX_3.4.30`, which corresponds to GCC 12+ release. On the other hand, Oracle Linux 9.6 ships with and older `libstdc++` that only supports up to GLIBCXX_3.4.29`. This creates a **compatibility gap**:

```shell
# Check your system's GLIBCXX support
strings /lib64/libstdc++.so.6 | grep GLIBCXX
```

You'll see versions up to `GLIBCXX_3.4.29` but not the required `GLIBCXX_3.4.30`.

Two proven solutions are discussed in the following sections.

### Approach 1: Use Compatible Triton Version (Quick Installation)

If you do not need the latest Triton version and wnat to prioritize stability, then use **Triton v3.3.1** which works with Oracle Linux 9 native libraries:

```shell
# Clone Triton repository
mkdir ~/git/triton && cd ~/git/triton
git clone https://github.com/OpenAI/triton.git
cd triton

# Checkout version compatible with GLIBCXX_3.4.29
git checkout v3.3.1

# Install build dependencies
pip install ninja cmake wheel pybind11

# OPTIONAL: Install scientific computing libraries
pip install --upgrade numba scipy setuptools_scm
pip install --upgrade "numpy<2"

# Install Triton
pip install -e python
```

**Benefits**: Quick installation, stable, no custom compilation required.

**Limitations**: Slightly older Triton features, dpendency on specific version.

### Approach 2: Build Custom LLVM (Latest Features)

For environments requiring latest Triton features, building LLVM locally resolves all compatibility issues.

#### Step 1: Install Additional Build Tools

```shell
# Install Clang and LLD for LLVM compilation
sudo dnf install clang lld

# Verify installations
clang --version
ld.lld --version
```

#### Step 2: Prepare Triton Source

```shell
# Clone latest Triton
mkdir ~/git/triton && cd ~/git/triton
git clone https://github.com/OpenAI/triton.git
cd triton

# Set environment variables for GCC build
export CC=gcc
export CXX=g++
```

#### Step 3: Modify LLVM Build Script

The default Triton build script uses hardcoded Clang settings that may cause linker issues. Modify it for GCC compatibility:

```shell
# Backup original build script
cp scripts/build-llvm-project.sh scripts/build-llvm-project.sh.bak

# Edit the script to use GCC instead of Clang
# Replace these lines in the script:
# -DCMAKE_C_COMPILER=clang
# -DCMAKE_CXX_COMPILER=clang++
# -DLLVM_ENABLE_LLD=ON

# With these:
# -DCMAKE_C_COMPILER=gcc
# -DCMAKE_CXX_COMPILER=g++
# -DLLVM_ENABLE_LLD=OFF
# -DCMAKE_ASM_COMPILER=gcc
# -DCMAKE_BUILD_WITH_INSTALL_RPATH=ON
# -DBUILD_SHARED_LIBS=ON
# -DCMAKE_POSITION_INDEPENDENT_CODE=ON
# -DLLVM_ENABLE_PIC=ON
# -DLLVM_OPTIMIZED_TABLEGEN=ON
```

#### Step 4: Build Custom LLVM

```shell
# Build LLVM with modified configuration (it might take +5 minutes)
make dev-install-llvm

# Install Triton with custom LLVM
# The build process automatically detects the custom LLVM installation
pip install -e .
```

**Advantages**: Latest Triton features, full compatibility, optimal performance.

**Limitations**: Longer build time, requires more system resources.

### Verification Steps

#### Kernel Compilation Test

Regardless of your chosen approach, verify the installation by compiling a simple kernel:

```shell
# Create test script
cat > test_triton.py << 'EOF'
import triton
import triton.language as tl

@triton.jit
def add_kernel(x_ptr, y_ptr, output_ptr, n_elements, BLOCK_SIZE: tl.constexpr):
    pid = tl.program_id(axis=0)
    block_start = pid * BLOCK_SIZE
    offsets = block_start + tl.arange(0, BLOCK_SIZE)
    mask = offsets < n_elements
    x = tl.load(x_ptr + offsets, mask=mask)
    y = tl.load(y_ptr + offsets, mask=mask)
    output = x + y
    tl.store(output_ptr + offsets, output, mask=mask)

print("✅ Triton kernel compilation successful!")
print(f"Triton version: {triton.__version__}")
EOF

cat > test-triton2.py << 'EOF'
@triton.jit
def test_kernel(ptr, n, BLOCK: tl.constexpr):
    idx = tl.program_id(0) * BLOCK + tl.arange(0, BLOCK)
    tl.store(ptr + idx, idx, mask=idx < n)

print("✅ Triton kernel compilation successful!")
print(f"Triton version: {triton.__version__}")
EOF


python test_triton.py

# Similar expected output in case of using Triton 3.4.0:
# ✅ Triton kernel compilation successful!
# Triton version: 3.4.0
```

#### LLVM Tools Verification

```shell
# Check LLVM version and targets
~/.triton/llvm/llvm-ubuntu-x64/bin/llc --version

# Similar expcted output:
# LLVM (http://llvm.org/):
#   LLVM version 21.0.0git
#   DEBUG build with assertions.
#   Default target: x86_64-unknown-linux-gnu
#   Host CPU: emeraldrapids

#   Registered Targets:
#     amdgcn  - AMD GCN GPUs
#     nvptx   - NVIDIA PTX 32-bit
#     nvptx64 - NVIDIA PTX 64-bit
#     r600    - AMD GPUs HD2XXX-HD6XXX
#     x86     - 32-bit X86: Pentium-Pro and above
#     x86-64  - 64-bit X86: EM64T and AMD64
```

### Quick Troubleshooting

#### mlir-tblgen execution failed

This typically indicates library compatibility issues:

```shell
# Verifyty LLVM tools can execute
ldd ~/.triton/llvm/llvm-ubuntu-x64/bin/mlir-tblgen
# Similar expected output (redacted):
        # linux-vdso.so.1 (0x00007ffd51c99000)
        # libLLVMCodeGenTypes.so.21.0git => ~/.triton/llvm/llvm-ubuntu-x64/bin/../lib/libLLVMCodeGenTypes.so.21.0git (0x00007f2882024000)
        # libLLVMTableGen.so.21.0git => ~/.triton/llvm/llvm-ubuntu-x64/bin/../lib/libLLVMTableGen.so.21.0git (0x00007f2881c00000)
        # libLLVMSupport.so.21.0git => ~/.triton/llvm/llvm-ubuntu-x64/bin/../lib/libLLVMSupport.so.21.0git (0x00007f2881400000)
        # libLLVMDemangle.so.21.0git => ~/.triton/llvm/llvm-ubuntu-x64/bin/../lib/libLLVMDemangle.so.21.0git (0x00007f2881f77000)
        # libstdc++.so.6 => /lib64/libstdc++.so.6 (0x00007f2881000000)
        # libm.so.6 => /lib64/libm.so.6 (0x00007f2881325000)
        # libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007f2881f51000)
        # libc.so.6 => /lib64/libc.so.6 (0x00007f2880c00000)
        # libz.so.1 => /lib64/libz.so.1 (0x00007f2881f37000)
        # libzstd.so.1 => /lib64/libzstd.so.1 (0x00007f288126e000)
        # /lib64/ld-linux-x86-64.so.2 (0x00007f288202f000)
```

#### Build Process Memory Issues

For systems with limited memory during LLVM compilation:

```shell
# Limit parallel build jobs
export MAKEFLAGS="-j4"  # Adjust based on available memory
export CMAKE_BUILD_PARALLEL_LEVEL=2 # Limit ninja build parallelism
make dev-install-llvm
```

## Conclusions

Both approaches resolve the Oracle Linux 9.6 compatibility gap with Triton. **Choose the compatible version approach** for quick deployment and stable environments. **Choose the custom LLVM build** when you need latest Triton features.

## Appendix: Deep Dive into Triton Build Challenges

> **This appendix is completely optional and intended only for readers interested in understanting the low-level technical challenges encountered during Triton installation. Feel free to skip this section.**

### Understanding the GLIBCXX Challenge

The core challenge originates from **C++ ABI (Application Binary Interface) compatibility**[^3]. Triton's pre-compiled LLVM tools (specifically `mlir-tblgen`) require `GLIBCXX_3.4.30` symbols, which corresponds to GCC 12+ runtime libraries. Oracle Linux 9.6 ships with `libstdc++.so.6.0.29`, providing only up to `GLIBCXX_3.4.29`.

The **dual ABI nature of libstdc++**[^4] compounds this issue - newer GCC toolsets in Oracle Linux exist in isolation (`/opt/rh/gcc-toolset-*`) but don't update the system-wide library that applications link against at runtime.

### LLVM Build Configuration Modifications

When building LLVM locally, the default Triton build script (`build-llvm-project.sh`) requires **significant modifications**:
- **Compiler override**: Change from hardcoded Clang to GCC for Oracle Linux compatibility
- **Linker configuration**: Disable LLD (`-DLLVM_ENABLE_LLD=OFF`) because of integration issues
- **Position Independent Code (PIC)**: Enable shared library compilation (`-DBUILD_SHARED_LIBS=ON`, `-DCMAKE_POSITION_INDEPENDENT_CODE=ON`)
- **Runtime path resolution**: `-DCMAKE_BUILD_WITH_INSTALL_RPATH=ON` eliminates dependency conflicts

These modifications prevent **static-to-shared linking problems** where LLVM attempts to create shared libraries by linking static compiled without PIC

## References

[^1]: Tillet, P., Kung, H. T., & Cox, D. (2019). Triton: An intermediate language and compiler for tiled neural network computations. Proceedings of the 3rd MLSys Conference. https://github.com/triton-lang/triton
[^2]: Lattner, C., & Adve, V. (2004). LLVM: A compilation framework for lifelong program analysis & transformation. Proceedings of the International Symposium on Code Generation and Optimization: Feedback-directed and Runtime Optimization, 75-86. IEEE Computer Society. https://llvm.org/
[^3]: Majid, O. (2020, July 8). What is this GLIBCXX error? Omair Majid. https://omairmajid.com/posts/2020-07-08-what-is-glibcxx-error/
[^4]: Free Software Foundation. (n.d.). Dual ABI. In The GNU C++ Library manual. GCC, the GNU Compiler Collection. https://gcc.gnu.org/onlinedocs/libstdc++/manual/using_dual_abi.html