# CUDA 12.8 + glibc 2.40 + GCC 15 Compatibility Fix

**Repository**: https://github.com/supere989/llama-cpp-python  
**Branch**: `fix/cuda-12.8-glibc-compatibility`  
**Status**: 🔧 **Work In Progress**

## Problem

Building `llama-cpp-python` with CUDA support fails on modern Linux systems with:
- **CUDA 12.8+**
- **glibc 2.40+** (Arch Linux, Fedora 41+, etc.)
- **GCC 15+**

### Error Symptoms

```
/usr/include/cuda/std/detail/libcxx/include/cmath(536): error: the global scope has no "ldexpl"
/usr/include/c++/15/type_traits(554): error: type name is not allowed
/usr/include/cuda_fp16.hpp(2864): error: identifier "__fmaf_rn" is undefined
```

### Root Causes

1. **GCC 15 Incompatibility**: CUDA 12.8 doesn't officially support GCC 15
2. **glibc 2.40 Changes**: New glibc removed some long double functions that CUDA headers expect
3. **CUDA Header Issues**: CUDA intrinsic functions not properly detected during CMake configuration

## Solution Approach

### 1. Force GCC 14 for CUDA Compilation

Modified `vendor/llama.cpp/ggml/src/ggml-cuda/CMakeLists.txt`:

```cmake
# Fix for CUDA 12.8+ with glibc 2.40+ and GCC 15+ compatibility issues
# Must be set BEFORE enable_language(CUDA) to affect compiler detection
if (CUDAToolkit_VERSION VERSION_GREATER_EQUAL "12.8")
    message(STATUS "Applying CUDA 12.8+ compatibility fixes for glibc 2.40+ and GCC 15+")
    
    # CUDA 12.8 doesn't support GCC 15, use GCC 14 if available
    find_program(GCC14_PATH g++-14)
    if(GCC14_PATH)
        message(STATUS "Using g++-14 for CUDA compatibility: ${GCC14_PATH}")
        set(CMAKE_CUDA_HOST_COMPILER "${GCC14_PATH}" CACHE STRING "" FORCE)
        set(CMAKE_CXX_COMPILER "${GCC14_PATH}" CACHE STRING "" FORCE)
    else()
        message(WARNING "g++-14 not found, CUDA compilation may fail with GCC 15+")
        set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} --allow-unsupported-compiler" CACHE STRING "" FORCE)
    endif()
endif()
```

### 2. Current Status

✅ **Working**:
- Fork created: https://github.com/supere989/llama-cpp-python
- Branch pushed: `fix/cuda-12.8-glibc-compatibility`
- GCC 14 detection and selection working
- CMake properly uses g++-14 for CUDA compilation

❌ **Still Failing**:
- CUDA intrinsic functions not detected during CMake's CUDA compiler identification
- Need to investigate CMake's CUDA language detection process
- May need to patch CMake's CUDA detection or provide pre-compiled identification

## Installation (When Fixed)

```bash
# Clone the fork
git clone https://github.com/supere989/llama-cpp-python.git
cd llama-cpp-python
git checkout fix/cuda-12.8-glibc-compatibility
git submodule update --init --recursive

# Install with CUDA support
CMAKE_ARGS="-DGGML_CUDA=on" pip3 install -e . --no-cache-dir
```

## System Requirements

- **CUDA**: 12.8+
- **GCC**: 14 (will be auto-detected and used)
- **Python**: 3.11+ (3.13 works)
- **CMake**: 3.18+

## Workarounds (Until Fixed)

### Option 1: Use Docker with CUDA 12.1

```bash
# Use older CUDA version in Docker
docker run --gpus all -it nvidia/cuda:12.1.0-devel-ubuntu22.04
```

### Option 2: Downgrade CUDA

```bash
# Install CUDA 12.1 instead of 12.8
# Download from: https://developer.nvidia.com/cuda-12-1-0-download-archive
```

### Option 3: Use Python 3.11/3.12 with Pre-built Wheels

```bash
# Create Python 3.11 environment
conda create -n llama python=3.11
conda activate llama

# Install pre-built wheel (if available for your CUDA version)
pip install llama-cpp-python --extra-index-url https://abetlen.github.io/llama-cpp-python/whl/cu121
```

### Option 4: CPU-Only Mode

```bash
# Install without CUDA support
pip install llama-cpp-python
```

## Next Steps

### Immediate Tasks

1. **Investigate CMake CUDA Detection**
   - Study CMake's `CMakeDetermineCUDACompiler.cmake`
   - Understand why CUDA intrinsics fail during identification
   - Possible solution: Skip compiler ID test for CUDA 12.8+

2. **Test Alternative Approaches**
   - Try setting `CMAKE_CUDA_COMPILER_ID` manually
   - Investigate `CMAKE_CUDA_COMPILER_FORCED` option
   - Consider patching CMake's CUDA detection

3. **Upstream Contribution**
   - Once working, create PR to llama.cpp
   - Document the fix for other users
   - Coordinate with llama-cpp-python maintainers

### Long-term Solution

The proper fix requires one of:
1. **NVIDIA**: Update CUDA 12.8 to support GCC 15
2. **llama.cpp**: Add workarounds for modern toolchains
3. **CMake**: Improve CUDA compiler detection robustness

## Testing

Once the fix is complete, verify with:

```bash
# Check CUDA support
python3 -c "from llama_cpp import Llama; import os; \
    model = Llama(model_path='model.gguf', n_gpu_layers=1); \
    print('GPU support:', model.n_gpu_layers)"

# Run inference test
python3 -c "from llama_cpp import Llama; \
    model = Llama(model_path='model.gguf', n_gpu_layers=-1); \
    result = model('Hello', max_tokens=10); \
    print(result)"

# Check GPU usage
nvidia-smi
```

## Related Issues

- **llama.cpp**: https://github.com/ggerganov/llama.cpp/issues/
- **llama-cpp-python**: https://github.com/abetlen/llama-cpp-python/issues/
- **CUDA GCC Support**: https://docs.nvidia.com/cuda/cuda-installation-guide-linux/

## Contributing

This is an active work in progress. Contributions welcome:

1. Test the current fix on your system
2. Suggest alternative approaches
3. Help debug CMake CUDA detection
4. Document workarounds that work for you

## Contact

- **GitHub**: https://github.com/supere989/llama-cpp-python
- **Branch**: `fix/cuda-12.8-glibc-compatibility`
- **Issues**: Open an issue on the fork

---

**Last Updated**: November 13, 2025  
**Status**: Partial fix implemented, still debugging CMake CUDA detection
