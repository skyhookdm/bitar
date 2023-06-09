# bitar

[![Codacy Badge](https://api.codacy.com/project/badge/Grade/3f22cce4889a4261997ae300e8f4e595)](https://app.codacy.com/gh/ljishen/bitar?utm_source=github.com&utm_medium=referral&utm_content=ljishen/bitar&utm_campaign=Badge_Grade_Settings)
[![GitHub Super-Linter](https://github.com/ljishen/bitar/workflows/Lint%20Code%20Base/badge.svg)](https://github.com/marketplace/actions/super-linter)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![DOI](https://zenodo.org/badge/490108576.svg)](https://zenodo.org/badge/latestdoi/490108576)

<!-- markdownlint-disable-next-line no-inline-html -->
<img src="assets/logo.png" width="25%">
Bitar is a C++ library to simplify accessing hardware compression/decompression accelerators.

## Features

- Zero-copy of data input and output
- Synchronous and asynchronous operations
- Multi-core and multi-device support
- No sudo permission requirement

Bitar can run either on the host machine or on the NVIDIA BlueField DPU target
in either DPU mode or separated host mode.

## Prerequisites

- Linux (with kernel >= 4.4) or FreeBSD
- For Linux, glibc >= 2.7 (reported by `ldd --version`)
- GCC >= 9 (C++17 compliant compiler)
- [DPDK](https://github.com/DPDK/dpdk) >= v21.11 (can be installed via vcpkg)
- [Apache Arrow](https://github.com/apache/arrow) >= 11.0.0 (build automatically
  if not found)

## Supported Hardware

- [NVIDIA BLUEFIELD-2 DPU](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/documents/datasheet-nvidia-bluefield-2-dpu.pdf)

## Integration

Bitar can be easily installed and integrated via
[vcpkg](https://github.com/microsoft/vcpkg)

```bash
vcpkg install bitar
```

## Development

- The DPDK library will be built from source by vcpkg if `dpdk_ROOT` is not
  specified.

- The Arrow parquet library is required if `BITAR_BUILD_APPS` is `ON`.
  Otherwise, just having the Arrow library is sufficient.

- Loading the Arrow parquet library will create a CMake target for the arrow and
  the parquet library, respectively.

- Use `Arrow_ROOT` to specify the installation prefix of the Arrow library if it
  is not installed at the default location. By default, the system-installed
  Arrow library will be at `/usr`.

- If the Arrow library is not found, or it is found but the parquet library is
  not found when needed, the Arrow library will be built from source.

```bash
$ # Reserve hugepages
$ sudo sh -c 'echo 1024 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages'
$ # On NUMA machines, we may need
$ # sudo sh -c 'echo 1024 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages'

$ # Install prerequisites for development
$ scripts/install-deps.sh

$ CC=clang CXX=clang++ cmake -S . -B ./build-$(uname -m) -G Ninja \
[-Ddpdk_ROOT:PATH=<dpdk-install-prefix>] \
[-DArrow_ROOT:PATH=<arrow-install-prefix>] \
-DBITAR_BUILD_APPS:BOOL=ON -DBITAR_BUILD_TESTS:BOOL=ON \
-DENABLE_DEVELOPER_MODE:BOOL=ON -DCMAKE_BUILD_TYPE:BOOL=Debug

$ cmake --build ./build-$(uname -m)
$ cmake --install ./build-$(uname -m) --prefix <install-prefix>

$ ./build-$(uname -m)/apps/demo_app --in-memory --lcores 5@(0-7),6-7 \
-a <device-pci-id>,class=compress -- --file <file> \
[--bytes <size-to-read-from-file>] [--mode <file-read-mode>] [--help]
```

### Advanced CMake Configuration Options

- `BITAR_FETCHCONTENT_OVERWRITE_CONFIGURATION`: set this option to `OFF` to have
  separate debug and release builds without overwriting each others
  configurations (default: `ON`)

- `VCPKG_ROOT`: the prefix to an installed vcpkg instance (install automatically
  if not specified)

- `BITAR_BUILD_ARROW`: set this option to `ON` to force building the Arrow
  dependency from source (default: `OFF`)

- `BITAR_ARROW_GIT_REPOSITORY`: the git repository to fetch the Arrow source
  (default: the official repository)

- `BITAR_ARROW_GIT_TAG`: use the source at the git branch, tag or commit hash
  from the Arrow repository for building when needed (default
  `apache-arrow-11.0.0`)

- `BITAR_INSTALL_ARROW`: install the Arrow library as part of the cmake
  installation process if Arrow is built by this project (default: `OFF`)

- Any
  [Arrow supported CMake options](https://github.com/apache/arrow/blob/apache-arrow-11.0.0/cpp/cmake_modules/DefineOptions.cmake),
  e.g., `ARROW_WITH_LZ4`, `ARROW_WITH_ZSTD`, and `ARROW_WITH_SNAPPY`.

## Known Issues

- (July 23, 2022) DPDK mistakenly assumes the support of `aes`, `pmull`, `sha1`,
  and `sha2` CPU flags on crypto-disabled BlueField-2 DPUs (e.g.,
  `MBF2H516A-CENO_Ax`, the one on the CloudLab r7525 machine) with **LLVM
  Clang**, resulting in the following error when executing a program compiled
  with bitar
  ([relevant code](https://github.com/DPDK/dpdk/blob/v22.07/config/arm/meson.build#L652-L655)):

  ```bash
  ERROR: This system does not support "AES".
  Please check that RTE_MACHINE is set correctly.
  ```

  There is no such problem when DPDK is compiled with GCC. Note that bitar can
  still be compiled with Clang and linked with DPDK that is compiled with GCC.

- ```bash
  [Out of memory: Reserving memzone of XXXXXX bytes failed. [Error 12: Cannot allocate memory]]
  ```

  (Aug 18, 2022) If you see this error, check whether DPDK is running in the
  Virtual Addresses mode (VA). Normally, you should see
  `EAL: Selected IOVA mode 'VA'` during the program initialization. But if you
  see `EAL: Selected IOVA mode 'PA'`, it's probably that the IOMMU is disable
  either in the BIOS or in the kernel. If IOMMU is enabled,
  `/sys/kernel/iommu_groups` path will contain kernel IOMMU groups and thus is
  not be empty. To enable IOMMU in the kernel, you can add
  `intel_iommu=on iommu=pt` in GRUB command-line on x86_64 systems, or add
  `iommu.passthrough=1` on aarch64 systems.
