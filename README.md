# retrohandle_toolchain

**The official cross-compilation toolchain for building native Linux applications on [PocketAIS](https://www.aislpc.com).**

PocketAIS is an RK3562-based handheld device running **AisOS**, a Rocknix-derived
Linux distribution. This repository ships the prebuilt C/C++ cross toolchains that
let you compile native ARM binaries (applications, libraries, and services) that
run directly on the device — no emulator, no container required.

With this toolchain you can build everything from a small command-line utility to a
full SDL2/OpenGL ES game or a Python extension module, exactly the way the stock
AisOS system libraries are built.

---

## What's in this repository

| Directory        | Target triple                     | ABI        | Use it for                                  |
| ---------------- | --------------------------------- | ---------- | ------------------------------------------- |
| `toolchain_r`    | `aarch64-rocknix-linux-gnueabi`   | 64-bit ARM | Modern PocketAIS apps (recommended default) |
| `toolchain_r_32` | `armv8a-rocknix-linux-gnueabi`    | 32-bit ARM | 32-bit-only targets / legacy binaries       |

Each directory is a **self-contained, relocatable** toolchain (compiler, binutils,
headers and a full sysroot with the device's system libraries). You do not need to
install anything else on your build machine besides the toolchain itself.

> **Note on repository size.** The two toolchain directories contain a complete
> compiler + sysroot and are large. They are committed as ordinary files so the
> repo is self-contained; if you prefer to keep the Git repo light, you can instead
> publish them as release tarballs / Git LFS and point developers here for the
> download instructions.

### Toolchain versions

| Component      | Version (both toolchains) |
| -------------- | ------------------------- |
| GCC            | 14.2.0                    |
| LLVM / Clang   | 19                        |
| C library      | glibc (sysroot)           |
| binutils / ld  | GNU linker (bfd + gold)   |
| Languages      | C, C++, Objective-C, Fortran (where applicable) |

---

## Host requirements

The cross-compilers are **Linux `x86_64` binaries**. To run them your build machine needs:

- A 64-bit Linux distribution (x86_64 / amd64).
- **glibc ≥ 2.32** on the build host (e.g. Ubuntu 20.04+, Debian 11+, Fedora 33+).
  The compiler binaries themselves link against the host glibc, so an older host
  glibc will fail with `version 'GLIBC_2.32' not found`.
- `ccache` (optional but recommended — see *Troubleshooting*).

Windows / macOS users: build inside WSL2, a Linux VM, or a Linux container. The
toolchains are not native Windows/macOS binaries.

---

## Quick start

### 1. Get the toolchain

```bash
git clone https://github.com/gamely-code/retrohandle_toolchain.git
cd retrohandle_toolchain
```

### 2. Put the compiler on your `PATH`

For the **64-bit** toolchain (recommended):

```bash
export TC=$PWD/toolchain_r
export PATH=$TC/bin:$PATH
export SYSROOT=$TC/aarch64-rocknix-linux-gnueabi/sysroot
```

For the **32-bit** toolchain:

```bash
export TC=$PWD/toolchain_r_32
export PATH=$TC/bin:$PATH
export SYSROOT=$TC            # headers/libs live at the toolchain root for the 32-bit build
```

(To find the exact configured sysroot at any time, run
`aarch64-rocknix-linux-gnueabi-gcc -print-sysroot`.)

### 3. Compile a "Hello, PocketAIS"

`hello.c`:

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

Copy `hello` to the device (via `scp`/ADB/SD card) and run it from a shell.

A C++ example works the same way with the matching `g++`:

```bash
aarch64-rocknix-linux-gnueabi-g++ --sysroot=$SYSROOT hello.cpp -o hello
```

---

## Building with CMake

Create a toolchain file (example: `aarch64-rocknix.cmake`):

```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

# Point this at the toolchain_r directory you cloned.
set(TOOLCHAIN_DIR "$ENV{TOOLCHAIN_DIR}")

set(CMAKE_C_COMPILER   ${TOOLCHAIN_DIR}/bin/aarch64-rocknix-linux-gnueabi-gcc)
set(CMAKE_CXX_COMPILER ${TOOLCHAIN_DIR}/bin/aarch64-rocknix-linux-gnueabi-g++)

set(CMAKE_SYSROOT ${TOOLCHAIN_DIR}/aarch64-rocknix-linux-gnueabi/sysroot)

# Search for programs on the host, but libraries/headers only in the sysroot.
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

Then configure and build:

```bash
export TOOLCHAIN_DIR=$PWD/toolchain_r
cmake -B build -DCMAKE_TOOLCHAIN_FILE=aarch64-rocknix.cmake
cmake --build build
```

For the 32-bit target, use `armv8a-rocknix-linux-gnueabi-gcc`/`g++` and set
`CMAKE_SYSROOT` to `$TOOLCHAIN_DIR` (the 32-bit sysroot is at the toolchain root).

---

## Prebuilt libraries in the sysroot

The sysroot already ships the system libraries AisOS uses, so you can link against
them out of the box (header search and library search default into the sysroot):

- **Graphics / display:** SDL2, Wayland (`libwayland-client/-cursor/-server` +
  protocols), GBM, libdrm, OpenGL ES (`libGLESv2`), EGL (Mesa), Vulkan (Panfrost driver)
- **Audio:** OpenAL
- **Scripting:** Python 3.11 (headers + `libpython3.11`)
- **Networking / crypto:** OpenSSL (`libssl` / `libcrypto`), cURL
- **Compression / data:** zlib, zstd, liblzma (xz), libxml2, expat
- **System / base:** glib / GObject / GIO, gettext, pcre2, ncurses, X11 / xcb, GMP / MPFR

To use a library, just `#include` its header and add `-l<name>` at link time, e.g.:

```bash
aarch64-rocknix-linux-gnueabi-gcc --sysroot=$SYSROOT app.c -lSDL2 -lGLESv2 -o app
```

---

## 64-bit vs 32-bit — which toolchain?

Use **`toolchain_r` (aarch64, 64-bit)** unless you have a specific reason not to.
It is the native word size of the RK3562 and what AisOS user-space is built for.

Use **`toolchain_r_32` (armv8a, 32-bit)** only when you must produce 32-bit ARM
binaries — for example to interoperate with a 32-bit-only component, or to reuse
existing 32-bit ARM code/libraries.

---

## Directory layout

```
retrohandle_toolchain/
├── README.md                 # this file
├── toolchain_r/              # aarch64 (64-bit) toolchain
│   ├── bin/                  # prefixed compilers & tools (aarch64-rocknix-linux-gnueabi-*)
│   ├── include/              # toolchain headers
│   └── aarch64-rocknix-linux-gnueabi/
│       └── sysroot/          # target sysroot (usr/lib, usr/include, ...)
└── toolchain_r_32/           # armv8a (32-bit) toolchain
    ├── bin/                  # prefixed compilers & tools (armv8a-rocknix-linux-gnueabi-*)
    ├── include/              # sysroot headers (at toolchain root for the 32-bit build)
    └── lib/                  # sysroot libraries
```

---

## Troubleshooting

**`aarch64-rocknix-linux-gnueabi-gcc: .../ccache: No such file or directory`**
The `gcc`/`g++` entries are thin `ccache` wrappers whose path was baked in at build
time. On your machine either:
- install `ccache` and recreate the wrapper so it calls `ccache` from your `PATH`, or
- simply invoke the real compiler directly:
  `aarch64-rocknix-linux-gnueabi-gcc-14.2.0` /
  `armv8a-rocknix-linux-gnueabi-gcc-14.2.0`.

**`version 'GLIBC_2.32' not found`**
Your build host is too old. Use a Linux distribution with glibc ≥ 2.32, or build
inside a container / WSL2 based on a current distro.

**Header / library not found**
Make sure you pass `--sysroot=$SYSROOT` (or set `CMAKE_SYSROOT` in CMake) and that
you are using the **prefixed** compiler, never your host `gcc`.

---

## License

The toolchain binaries are redistributed under the licenses of their respective
upstream projects (GCC, LLVM, glibc, binutils, and the bundled system libraries).
See each component's `share/doc/*/COPYING` / `share/licenses` inside the toolchain
directories for the exact terms.

> ⚠️ A top-level `LICENSE` file for this repository's own documentation/scripts has
> not been added yet — please choose and commit one before the public release.

## Contributing

This repository distributes prebuilt toolchains; it is not built from source here.
To report an issue or request an updated/broader toolchain, please open an issue on
the project tracker.

---

