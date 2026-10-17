# retrohandle_toolchain

**用于在 [PocketAIS](https://www.aislpc.com) 上构建原生 Linux 应用的官方交叉编译工具链。**

PocketAIS 是一台基于 RK3562 的手持设备，运行 **AisOS** —— 一款衍生自 Rocknix 的 Linux 发行版。本仓库提供预编译的 C/C++ 交叉工具链，可用于编译能够直接在设备上运行的原生 ARM 二进制文件（应用程序、库与服务），无需模拟器、也无需容器。

借助本工具链，你可以构建从简单的命令行小工具，到完整的 SDL2/OpenGL ES 游戏，再到 Python 扩展模块——其构建方式与 AisOS 系统原生库完全一致。

---

## 本仓库包含什么

| 目录             | 目标三元组                      | ABI        | 用途                                    |
| ---------------- | ------------------------------- | ---------- | --------------------------------------- |
| `toolchain_r`    | `aarch64-rocknix-linux-gnueabi` | 64 位 ARM  | 现代 PocketAIS 应用（推荐默认选择）     |
| `toolchain_r_32` | `armv8a-rocknix-linux-gnueabi`  | 32 位 ARM  | 仅 32 位目标 / 遗留二进制文件           |

每个目录都是一个**自包含、可重定位**的工具链（含编译器、binutils、头文件，以及一套包含设备系统库的完整 sysroot）。除了工具链本身，你的构建机器上无需安装其他任何东西。

> **关于仓库体积的说明。** 两个工具链目录包含完整的编译器 + sysroot，因此体积很大。它们以普通文件形式提交入库，以保证仓库自包含；如果你希望保持 Git 仓库精简，也可以将它们发布为发布压缩包 / Git LFS，并在此指向下载说明供开发者获取。

### 工具链版本

| 组件            | 版本（两个工具链一致）                  |
| --------------- | --------------------------------------- |
| GCC             | 14.2.0                                  |
| LLVM / Clang    | 19                                      |
| C 库            | glibc（sysroot）                        |
| binutils / ld   | GNU linker（bfd + gold）                |
| 语言            | C、C++、Objective-C、Fortran（视情况）  |

---

## 宿主环境要求

交叉编译器本身是 **Linux `x86_64` 二进制文件**。要运行它们，你的构建机器需要满足：

- 64 位 Linux 发行版（x86_64 / amd64）。
- 构建宿主上的 **glibc ≥ 2.32**（例如 Ubuntu 20.04+、Debian 11+、Fedora 33+）。
  编译器二进制文件本身链接于宿主 glibc，因此宿主 glibc 过旧时会报错
  `version 'GLIBC_2.32' not found`。
- `ccache`（可选但推荐——见*故障排查*）。

Windows / macOS 用户：请在 WSL2、Linux 虚拟机或 Linux 容器中构建。这些工具链并非原生的 Windows/macOS 二进制文件。

---

## 快速开始

### 1. 获取工具链

```bash
git clone https://github.com/gamely-code/retrohandle_toolchain.git
cd retrohandle_toolchain
```

### 2. 将编译器加入 `PATH`

对于 **64 位** 工具链（推荐）：

```bash
export TC=$PWD/toolchain_r
export PATH=$TC/bin:$PATH
export SYSROOT=$TC/aarch64-rocknix-linux-gnueabi/sysroot
```

对于 **32 位** 工具链：

```bash
export TC=$PWD/toolchain_r_32
export PATH=$TC/bin:$PATH
export SYSROOT=$TC            # 32 位构建的头文件/库位于工具链根目录
```

（如需随时查询实际配置的 sysroot，可运行
`aarch64-rocknix-linux-gnueabi-gcc -print-sysroot`。）

### 3. 编译一个 "Hello, PocketAIS"

`hello.c`：

```c
#include <stdio.h>

int main(void) {
    printf("Hello from PocketAIS!\n");
    return 0;
}
```

```bash
aarch64-rocknix-linux-gnueabi-gcc --sysroot=$SYSROOT hello.c -o hello
file hello            # => ELF 64-bit LSB executable, ARM aarch64
```

将 `hello` 复制到设备（通过 `scp`/ADB/SD 卡），再从 shell 中运行它。

C++ 示例用法相同，只需使用对应的 `g++`：

```bash
aarch64-rocknix-linux-gnueabi-g++ --sysroot=$SYSROOT hello.cpp -o hello
```

---

## 使用 CMake 构建

创建一个工具链文件（示例：`aarch64-rocknix.cmake`）：

```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

# 指向你克隆的 toolchain_r 目录。
set(TOOLCHAIN_DIR "$ENV{TOOLCHAIN_DIR}")

set(CMAKE_C_COMPILER   ${TOOLCHAIN_DIR}/bin/aarch64-rocknix-linux-gnueabi-gcc)
set(CMAKE_CXX_COMPILER ${TOOLCHAIN_DIR}/bin/aarch64-rocknix-linux-gnueabi-g++)

set(CMAKE_SYSROOT ${TOOLCHAIN_DIR}/aarch64-rocknix-linux-gnueabi/sysroot)

# 在宿主上搜索程序，但只在 sysroot 中搜索库/头文件。
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

随后配置并构建：

```bash
export TOOLCHAIN_DIR=$PWD/toolchain_r
cmake -B build -DCMAKE_TOOLCHAIN_FILE=aarch64-rocknix.cmake
cmake --build build
```

对于 32 位目标，请使用 `armv8a-rocknix-linux-gnueabi-gcc`/`g++`，并将
`CMAKE_SYSROOT` 设为 `$TOOLCHAIN_DIR`（32 位 sysroot 位于工具链根目录）。

---

## sysroot 中预置的库

sysroot 中已自带 AisOS 所使用的系统库，因此你可以开箱即用地链接它们（头文件搜索与库搜索默认指向 sysroot）：

- **图形 / 显示：** SDL2、Wayland（`libwayland-client/-cursor/-server` + 协议）、GBM、libdrm、OpenGL ES（`libGLESv2`）、EGL（Mesa）、Vulkan（Panfrost 驱动）
- **音频：** OpenAL
- **脚本：** Python 3.11（头文件 + `libpython3.11`）
- **网络 / 加密：** OpenSSL（`libssl` / `libcrypto`）、cURL
- **压缩 / 数据：** zlib、zstd、liblzma（xz）、libxml2、expat
- **系统 / 基础：** glib / GObject / GIO、gettext、pcre2、ncurses、X11 / xcb、GMP / MPFR

使用某个库时，只需 `#include` 其头文件，并在链接时加上 `-l<名称>`，例如：

```bash
aarch64-rocknix-linux-gnueabi-gcc --sysroot=$SYSROOT app.c -lSDL2 -lGLESv2 -o app
```

---

## 64 位还是 32 位——该选哪个工具链？

除非你有特定理由不这么做，否则请使用 **`toolchain_r`（aarch64，64 位）**。
它是 RK3562 的原生字长，也是 AisOS 用户空间构建所针对的架构。

仅当你必须产出 32 位 ARM 二进制文件时才使用 **`toolchain_r_32`（armv8a，32 位）**——
例如需要与仅支持 32 位的组件互操作，或需要复用已有的 32 位 ARM 代码/库。

---

## 目录结构

```
retrohandle_toolchain/
├── README.md                 # 本文件
├── toolchain_r/              # aarch64（64 位）工具链
│   ├── bin/                  # 带前缀的编译器与工具（aarch64-rocknix-linux-gnueabi-*）
│   ├── include/              # 工具链头文件
│   └── aarch64-rocknix-linux-gnueabi/
│       └── sysroot/          # 目标 sysroot（usr/lib、usr/include 等）
└── toolchain_r_32/           # armv8a（32 位）工具链
    ├── bin/                  # 带前缀的编译器与工具（armv8a-rocknix-linux-gnueabi-*）
    ├── include/              # sysroot 头文件（32 位构建位于工具链根目录）
    └── lib/                  # sysroot 库
```

---

## 故障排查

**`aarch64-rocknix-linux-gnueabi-gcc: .../ccache: No such file or directory`**
`gcc`/`g++` 入口是精简的 `ccache` 包装脚本，其内部路径在构建时即已写死。在你的机器上可二选一：
- 安装 `ccache` 并重新创建包装脚本，使其从你的 `PATH` 中调用 `ccache`；或者
- 直接调用真正的编译器：
  `aarch64-rocknix-linux-gnueabi-gcc-14.2.0` /
  `armv8a-rocknix-linux-gnueabi-gcc-14.2.0`。

**`version 'GLIBC_2.32' not found`**
你的构建宿主版本过旧。请使用 glibc ≥ 2.32 的 Linux 发行版，或基于当前发行版在容器 / WSL2 内构建。

**找不到头文件 / 库**
请确保传入了 `--sysroot=$SYSROOT`（或在 CMake 中设置 `CMAKE_SYSROOT`），并且你使用的是**带前缀的**编译器，而非宿主上的 `gcc`。

---

## 许可证

工具链二进制文件依据各自上游项目（GCC、LLVM、glibc、binutils，以及随附的系统库）的许可证进行再分发。
具体条款请参阅工具链目录内各组件的 `share/doc/*/COPYING` / `share/licenses`。

> ⚠️ 本仓库自身的文档/脚本尚未添加顶级的 `LICENSE` 文件——在公开发布前请选定并补提交一个。

## 贡献

本仓库分发的是预构建工具链，并非在此从源码构建。如需报告问题或申请更新/更全面的工具链，请到项目追踪页面提交 issue。

---

