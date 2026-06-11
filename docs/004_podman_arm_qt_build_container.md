# Podman Container for ARM Qt Cross-Compilation on Arch Linux

> **Host OS:** Arch Linux  
> **Target Architecture:** ARM (32-bit `armv7` / 64-bit `aarch64`)  
> **Qt Versions covered:** Qt5 (§1–§5) · Qt6 extension (§6)

---

## Table of Contents

1. [Host Prerequisites](#1-host-prerequisites)
2. [Project Layout](#2-project-layout)
3. [Cross-Compilation Toolchain Strategy](#3-cross-compilation-toolchain-strategy)
4. [Qt5 Containerfile](#4-qt5-containerfile)
5. [Build & Run the Qt5 Container](#5-build--run-the-qt5-container)
6. [Qt6 Extension](#6-qt6-extension)
7. [Volume Mounts & Workflow](#7-volume-mounts--workflow)
8. [Troubleshooting](#8-troubleshooting)
9. [Security & Maintenance Notes](#9-security--maintenance-notes)

---

## 1. Host Prerequisites

Install the required host packages on Arch Linux before building any container image.

```bash
# Core container runtime
sudo pacman -S podman

# QEMU user-mode emulation — needed if you want to run ARM binaries on the host
# or inside the container via binfmt_misc
sudo pacman -S qemu-user-static

# Register QEMU binfmt handlers so the kernel can transparently execute ARM ELFs
sudo systemctl enable --now systemd-binfmt

# Verify binfmt registration
ls /proc/sys/fs/binfmt_misc/ | grep -E 'aarch64|arm'
# Expected output includes: qemu-aarch64  qemu-arm

# Optional but recommended: buildah for advanced layer inspection
sudo pacman -S buildah
```

> **Why `qemu-user-static`?**  
> Without it, the ARM binaries produced inside (or run inside) the container cannot be executed on an x86_64 kernel. The static variant embeds the emulator directly into each binary's interpreter slot, making it available even inside a container namespace with no host bind-mount.

---

## 2. Project Layout

Keep everything self-contained so the `Containerfile` and helper scripts are portable.

```
arm-qt-builder/
├── qt5/
│   ├── Containerfile          # Qt5 image definition
│   ├── toolchain-arm.cmake    # CMake toolchain file for armv7
│   ├── toolchain-aarch64.cmake# CMake toolchain file for aarch64
│   └── qt5-configure.sh       # Qt5 ./configure wrapper
├── qt6/
│   ├── Containerfile          # Qt6 image definition
│   ├── toolchain-arm.cmake
│   ├── toolchain-aarch64.cmake
│   └── qt6-configure.sh
├── sysroot/                   # Populated at image build time (or pre-staged)
│   ├── armv7/
│   └── aarch64/
└── projects/                  # Bind-mounted; your application source lives here
```

---

## 3. Cross-Compilation Toolchain Strategy

Two complementary approaches are used inside the container.

**Approach A — Native ARM packages via `pacman` + QEMU binfmt (recommended for Qt)**  
The container runs an `arm64v8/archlinuxarm` base image. Podman transparently executes its ARM `pacman` binary through QEMU. Qt is compiled _for_ ARM _by_ the ARM compiler inside the image. No sysroot staging is needed.

**Approach B — Cross-compiler on x86_64 host**  
Install `aarch64-linux-gnu-gcc` / `arm-linux-gnueabihf-gcc` from `[extra]` and compile with a CMake toolchain file pointing at a staged sysroot. Faster compilation, but sysroot maintenance is more complex.

This guide uses **Approach A** as the primary path and shows the CMake toolchain files for Approach B in the appendix blocks.

---

## 4. Qt5 Containerfile

### 4.1 `qt5/Containerfile`

```dockerfile
# ─────────────────────────────────────────────────────────────────────────────
# Stage 0 – Base ARM64 Arch Linux image
# We pull the official Arch Linux ARM image; QEMU binfmt on the host makes it
# transparently executable on x86_64.
# ─────────────────────────────────────────────────────────────────────────────
FROM menci/archlinuxarm:latest AS base
# Alternative base: lopsided/archlinuxarm:aarch64  (community-maintained)
# For armv7 targets swap to:  lopsided/archlinuxarm:armv7

LABEL maintainer="your-team@example.com"
LABEL description="ARM Qt5 cross-build environment (Arch Linux)"
LABEL qt.version="5.x"
LABEL target.arch="aarch64"

# ─────────────────────────────────────────────────────────────────────────────
# Stage 1 – System update + essential build tools
# ─────────────────────────────────────────────────────────────────────────────
FROM base AS toolchain

# Initialise pacman keyring (required for Arch Linux ARM images)
RUN pacman-key --init && \
    pacman-key --populate archlinuxarm && \
    pacman -Syu --noconfirm --noprogressbar

# Core build toolchain
RUN pacman -S --noconfirm --noprogressbar \
    base-devel \
    gcc \
    g++ \
    cmake \
    ninja \
    make \
    git \
    python \
    python-pip \
    pkg-config \
    ccache \
    gdb \
    valgrind

# ─────────────────────────────────────────────────────────────────────────────
# Stage 2 – Qt5 runtime & development libraries
# ─────────────────────────────────────────────────────────────────────────────
FROM toolchain AS qt5-deps

# Qt5 base + common modules
RUN pacman -S --noconfirm --noprogressbar \
    qt5-base \
    qt5-declarative \
    qt5-multimedia \
    qt5-serialport \
    qt5-tools \
    qt5-websockets \
    qt5-svg \
    qt5-x11extras \
    qt5-charts \
    qt5-networkauth \
    qt5-remoteobjects \
    qt5-serialbus \
    qt5-datavis3d \
    qt5-gamepad

# Platform / display back-ends
RUN pacman -S --noconfirm --noprogressbar \
    libxcb \
    xcb-util \
    xcb-util-image \
    xcb-util-keysyms \
    xcb-util-renderutil \
    xcb-util-wm \
    libx11 \
    libxext \
    libxrender \
    libxi \
    libxkbcommon \
    libxkbcommon-x11 \
    mesa \
    libgl \
    libgles \
    egl-wayland \
    wayland \
    libinput \
    libevdev \
    tslib               # touchscreen support — common on ARM boards

# Audio (often needed on embedded ARM)
RUN pacman -S --noconfirm --noprogressbar \
    alsa-lib \
    pulseaudio \
    gstreamer \
    gst-plugins-base \
    gst-plugins-good

# Connectivity & crypto
RUN pacman -S --noconfirm --noprogressbar \
    openssl \
    nss \
    ca-certificates \
    curl \
    libdbus \
    dbus

# ─────────────────────────────────────────────────────────────────────────────
# Stage 3 – Environment setup & ccache configuration
# ─────────────────────────────────────────────────────────────────────────────
FROM qt5-deps AS final

# ccache speeds up repeated builds dramatically
ENV CCACHE_DIR=/ccache
ENV CCACHE_MAXSIZE=5G
ENV PATH=/usr/lib/ccache/bin:$PATH

# Qt5 environment variables
ENV QT_VERSION=5
ENV Qt5_DIR=/usr/lib/cmake/Qt5
ENV PKG_CONFIG_PATH=/usr/lib/pkgconfig

# Compiler environment
ENV CC="ccache gcc"
ENV CXX="ccache g++"
ENV CFLAGS="-O2 -march=armv8-a -mtune=cortex-a53 -fstack-protector-strong"
ENV CXXFLAGS="${CFLAGS}"

# Build output directory convention
RUN mkdir -p /workspace/build /workspace/install /ccache

WORKDIR /workspace

# Healthcheck — verify qmake is functional
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD qmake --version || exit 1

# Default: drop into bash so the developer can invoke qmake/cmake interactively
CMD ["/bin/bash"]
```

### 4.2 CMake Toolchain File (Approach B — host cross-compiler)

Save as `qt5/toolchain-aarch64.cmake` for use when compiling from an x86_64 host container.

```cmake
# toolchain-aarch64.cmake
# Use with: cmake -DCMAKE_TOOLCHAIN_FILE=toolchain-aarch64.cmake ..

set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

# Sysroot populated from the target device or from ARM packages
set(CMAKE_SYSROOT /sysroot/aarch64)

set(TOOLCHAIN_PREFIX aarch64-linux-gnu)

find_program(CMAKE_C_COMPILER   ${TOOLCHAIN_PREFIX}-gcc   REQUIRED)
find_program(CMAKE_CXX_COMPILER ${TOOLCHAIN_PREFIX}-g++   REQUIRED)
find_program(CMAKE_AR           ${TOOLCHAIN_PREFIX}-ar     REQUIRED)
find_program(CMAKE_RANLIB       ${TOOLCHAIN_PREFIX}-ranlib REQUIRED)
find_program(CMAKE_STRIP        ${TOOLCHAIN_PREFIX}-strip  REQUIRED)

set(CMAKE_FIND_ROOT_PATH ${CMAKE_SYSROOT})
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)

# Point CMake at the cross-compiled Qt5 installation
set(Qt5_DIR ${CMAKE_SYSROOT}/usr/lib/cmake/Qt5)
```

### 4.3 `qt5/qt5-configure.sh` — qmake configure helper

```bash
#!/usr/bin/env bash
# qt5-configure.sh — wrapper that calls qmake with sensible defaults for ARM
set -euo pipefail

SOURCE_DIR="${1:-.}"
BUILD_DIR="${2:-build}"

mkdir -p "${BUILD_DIR}"
cd "${BUILD_DIR}"

qmake \
    "${SOURCE_DIR}" \
    "CONFIG+=release" \
    "CONFIG+=c++17" \
    "QMAKE_CXXFLAGS+=-O2 -march=armv8-a" \
    "QMAKE_LFLAGS+=-Wl,--as-needed" \
    "$@"

make -j"$(nproc)"
```

---

## 5. Build & Run the Qt5 Container

### 5.1 Build the image

```bash
cd arm-qt-builder/qt5

# Build (first run downloads ~800 MB of ARM packages)
podman build \
    --platform linux/arm64 \
    --tag arm-qt5-builder:latest \
    --file Containerfile \
    .

# Verify the image
podman image inspect arm-qt5-builder:latest | grep -E 'Architecture|Os'
```

> **Platform flag:** `--platform linux/arm64` tells Podman/Buildah which ELF class to request. QEMU binfmt on the host transparently handles execution.

### 5.2 Run an interactive build session

```bash
podman run --rm -it \
    --platform linux/arm64 \
    --volume "$(pwd)/../projects:/workspace/source:z" \
    --volume "$(pwd)/../ccache:/ccache:z" \
    --volume "$(pwd)/../sysroot:/sysroot:z" \
    --userns=keep-id \
    --security-opt label=disable \
    arm-qt5-builder:latest \
    /bin/bash
```

Inside the container:

```bash
# CMake-based project
cmake \
    -S /workspace/source/my-app \
    -B /workspace/build \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_PREFIX_PATH=/usr/lib/cmake/Qt5
cmake --build /workspace/build -- -j"$(nproc)"

# qmake-based project
cd /workspace/source/my-app
/workspace/qt5-configure.sh . /workspace/build
```

### 5.3 Non-interactive CI / scripted build

```bash
podman run --rm \
    --platform linux/arm64 \
    --volume "$(pwd)/../projects/my-app:/workspace/source:z,ro" \
    --volume "$(pwd)/../artifacts:/workspace/install:z" \
    --volume "$(pwd)/../ccache:/ccache:z" \
    arm-qt5-builder:latest \
    bash -c "
        cmake \
            -S /workspace/source \
            -B /workspace/build \
            -G Ninja \
            -DCMAKE_BUILD_TYPE=Release \
            -DCMAKE_INSTALL_PREFIX=/workspace/install \
            -DCMAKE_PREFIX_PATH=/usr/lib/cmake/Qt5 && \
        cmake --build /workspace/build --target install -- -j\$(nproc)
    "
```

---

## 6. Qt6 Extension

Qt6 introduces several breaking changes from a build-system perspective:

| Area | Qt5 | Qt6 |
|---|---|---|
| Minimum CMake | 3.16 | **3.16** (CMake-only; qmake de-emphasised) |
| Minimum C++ standard | C++11 | **C++17** |
| QML module system | `AUTOMOC` + `.pro` | `qt_add_qml_module()` |
| Android / WASM | External | First-class |
| OpenGL | Optional | Optional; Vulkan preferred |
| Python bindings | PySide2 / Shiboken2 | **PySide6 / Shiboken6** |
| QRegExp | Available | **Removed** → use `QRegularExpression` |

### 6.1 `qt6/Containerfile`

```dockerfile
# ─────────────────────────────────────────────────────────────────────────────
# Stage 0 – Base ARM64 Arch Linux image (same as Qt5)
# ─────────────────────────────────────────────────────────────────────────────
FROM menci/archlinuxarm:latest AS base

LABEL maintainer="your-team@example.com"
LABEL description="ARM Qt6 cross-build environment (Arch Linux)"
LABEL qt.version="6.x"
LABEL target.arch="aarch64"

# ─────────────────────────────────────────────────────────────────────────────
# Stage 1 – System update + toolchain
# Qt6 requires GCC 10+ / Clang 12+ for full C++17/20 support
# ─────────────────────────────────────────────────────────────────────────────
FROM base AS toolchain

RUN pacman-key --init && \
    pacman-key --populate archlinuxarm && \
    pacman -Syu --noconfirm --noprogressbar

RUN pacman -S --noconfirm --noprogressbar \
    base-devel \
    gcc \
    g++ \
    clang \
    llvm \
    lld \
    cmake \
    ninja \
    make \
    git \
    python \
    python-pip \
    pkg-config \
    ccache \
    gdb \
    valgrind \
    perl                # required by Qt6 build system internals

# ─────────────────────────────────────────────────────────────────────────────
# Stage 2 – Qt6 runtime & development libraries
# Qt6 packages in Arch Linux ARM are named qt6-*
# ─────────────────────────────────────────────────────────────────────────────
FROM toolchain AS qt6-deps

# Qt6 core modules
RUN pacman -S --noconfirm --noprogressbar \
    qt6-base \
    qt6-declarative \
    qt6-multimedia \
    qt6-serialport \
    qt6-tools \
    qt6-websockets \
    qt6-svg \
    qt6-charts \
    qt6-networkauth \
    qt6-remoteobjects \
    qt6-serialbus \
    qt6-datavis3d \
    qt6-connectivity \
    qt6-positioning \
    qt6-sensors \
    qt6-shadertools \
    qt6-quick3d \
    qt6-5compat        # Qt5Compat module: QRegExp, QTextCodec, etc.

# Qt6 requires Vulkan headers (even if you target OpenGL/GLES at runtime)
RUN pacman -S --noconfirm --noprogressbar \
    vulkan-headers \
    vulkan-icd-loader \
    mesa \
    libgl \
    libgles \
    egl-wayland \
    wayland \
    wayland-protocols

# Display / input (same as Qt5 with additions for Qt6 Wayland compositor)
RUN pacman -S --noconfirm --noprogressbar \
    libxcb \
    xcb-util \
    xcb-util-image \
    xcb-util-keysyms \
    xcb-util-renderutil \
    xcb-util-wm \
    xcb-util-cursor \   # new dependency in Qt6
    libx11 \
    libxext \
    libxrender \
    libxi \
    libxkbcommon \
    libxkbcommon-x11 \
    libinput \
    libevdev \
    tslib

# Audio (Qt6 Multimedia uses FFmpeg or GStreamer back-end)
RUN pacman -S --noconfirm --noprogressbar \
    ffmpeg \
    gstreamer \
    gst-plugins-base \
    gst-plugins-good \
    gst-plugins-bad \
    alsa-lib \
    pipewire \
    pipewire-alsa

# Connectivity & crypto
RUN pacman -S --noconfirm --noprogressbar \
    openssl \
    nss \
    ca-certificates \
    curl \
    libdbus \
    dbus \
    bluez-libs          # Qt6 Bluetooth

# Python bindings (optional — remove if not needed)
RUN pacman -S --noconfirm --noprogressbar \
    pyside6 \
    shiboken6 || true   # best-effort; may not yet be in ARM repo

# ─────────────────────────────────────────────────────────────────────────────
# Stage 3 – Environment setup
# ─────────────────────────────────────────────────────────────────────────────
FROM qt6-deps AS final

ENV CCACHE_DIR=/ccache
ENV CCACHE_MAXSIZE=5G
ENV PATH=/usr/lib/ccache/bin:$PATH

ENV QT_VERSION=6
ENV Qt6_DIR=/usr/lib/cmake/Qt6
ENV PKG_CONFIG_PATH=/usr/lib/pkgconfig

# Qt6 recommends Clang, but GCC works fine on Arch Linux ARM
ENV CC="ccache gcc"
ENV CXX="ccache g++"

# C++17 is the minimum; C++20 is encouraged for new Qt6 projects
ENV CFLAGS="-O2 -march=armv8-a -mtune=cortex-a53 -std=c17 -fstack-protector-strong"
ENV CXXFLAGS="-O2 -march=armv8-a -mtune=cortex-a53 -std=c++20 -fstack-protector-strong"

# Qt6 feature flags
ENV QT_QPA_PLATFORM=xcb             # default platform; override at runtime
ENV QT_ENABLE_HIGHDPI_SCALING=1

RUN mkdir -p /workspace/build /workspace/install /ccache

WORKDIR /workspace

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD qmake6 --version 2>/dev/null || qmake --version || exit 1

CMD ["/bin/bash"]
```

### 6.2 CMake Toolchain File for Qt6 (Approach B)

```cmake
# toolchain-aarch64-qt6.cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)
set(CMAKE_SYSROOT /sysroot/aarch64)

set(TOOLCHAIN_PREFIX aarch64-linux-gnu)

find_program(CMAKE_C_COMPILER   ${TOOLCHAIN_PREFIX}-gcc   REQUIRED)
find_program(CMAKE_CXX_COMPILER ${TOOLCHAIN_PREFIX}-g++   REQUIRED)

# Qt6 requires C++17 minimum
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_FIND_ROOT_PATH ${CMAKE_SYSROOT})
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)

# Qt6: every module has its own CMake config
set(Qt6_DIR           ${CMAKE_SYSROOT}/usr/lib/cmake/Qt6)
set(Qt6Core_DIR       ${CMAKE_SYSROOT}/usr/lib/cmake/Qt6Core)
set(Qt6Widgets_DIR    ${CMAKE_SYSROOT}/usr/lib/cmake/Qt6Widgets)
# Add more as needed: Qt6Quick_DIR, Qt6Multimedia_DIR, etc.

# Qt6 needs the host qmake6 to generate some code at configure time
find_program(QT_HOST_PATH_CMAKE_DIR NAMES qmake6 qmake HINTS /usr/bin)
```

### 6.3 `qt6/qt6-configure.sh` — CMake configure helper for Qt6

```bash
#!/usr/bin/env bash
# qt6-configure.sh — CMake configure + build for Qt6 ARM projects
set -euo pipefail

SOURCE_DIR="${1:-.}"
BUILD_DIR="${2:-build}"
INSTALL_DIR="${3:-/workspace/install}"

mkdir -p "${BUILD_DIR}"

cmake \
    -S "${SOURCE_DIR}" \
    -B "${BUILD_DIR}" \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_CXX_STANDARD=20 \
    -DCMAKE_INSTALL_PREFIX="${INSTALL_DIR}" \
    -DCMAKE_PREFIX_PATH=/usr/lib/cmake/Qt6 \
    -DCMAKE_C_COMPILER_LAUNCHER=ccache \
    -DCMAKE_CXX_COMPILER_LAUNCHER=ccache \
    "$@"

cmake --build "${BUILD_DIR}" --target install -- -j"$(nproc)"
```

### 6.4 Build & Run the Qt6 Container

```bash
cd arm-qt-builder/qt6

podman build \
    --platform linux/arm64 \
    --tag arm-qt6-builder:latest \
    --file Containerfile \
    .

# Interactive session
podman run --rm -it \
    --platform linux/arm64 \
    --volume "$(pwd)/../projects:/workspace/source:z" \
    --volume "$(pwd)/../ccache:/ccache:z" \
    --userns=keep-id \
    --security-opt label=disable \
    arm-qt6-builder:latest \
    /bin/bash

# Inside the container — Qt6 CMake project
cmake \
    -S /workspace/source/my-qt6-app \
    -B /workspace/build \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_CXX_STANDARD=20 \
    -DCMAKE_PREFIX_PATH=/usr/lib/cmake/Qt6
cmake --build /workspace/build -- -j"$(nproc)"
```

---

## 7. Volume Mounts & Workflow

| Host path | Container path | Purpose |
|---|---|---|
| `./projects/` | `/workspace/source` | Application source (read-only in CI) |
| `./artifacts/` | `/workspace/install` | Build output / deployment artefacts |
| `./ccache/` | `/ccache` | Persistent compiler cache across runs |
| `./sysroot/` | `/sysroot` | Optional: staged ARM sysroot (Approach B) |

### Persistent ccache across sessions

On the **host**, seed the cache directory once:

```bash
mkdir -p ~/arm-build-cache/qt5 ~/arm-build-cache/qt6
```

Pass the cache on every `podman run`:

```bash
--volume "$HOME/arm-build-cache/qt5:/ccache:z"
```

After a warm cache, incremental rebuilds of a mid-size Qt5 application typically drop from ~8 min to under 60 seconds.

---

## 8. Troubleshooting

### QEMU binfmt not active

```
exec format error
```

Fix:

```bash
sudo pacman -S qemu-user-static
sudo systemctl restart systemd-binfmt
cat /proc/sys/fs/binfmt_misc/qemu-aarch64   # must say "enabled"
```

### pacman keyring failures inside container

```
error: required key missing from keyring
```

Fix: The Containerfile already runs `pacman-key --init && pacman-key --populate archlinuxarm`. If the build cache has a stale layer, force a rebuild:

```bash
podman build --no-cache --tag arm-qt5-builder:latest .
```

### Missing Qt6 packages in Arch Linux ARM repo

Some Qt6 modules (e.g. `qt6-datavis3d`, `qt6-quick3d`) may lag behind the x86_64 repo. Build from AUR inside the container:

```bash
# Inside the container
pacman -S --noconfirm git base-devel
useradd -m builder && su - builder
git clone https://aur.archlinux.org/qt6-datavis3d.git
cd qt6-datavis3d && makepkg -si --noconfirm
```

### SELinux / security label issues on volume mounts

The `:z` suffix on `--volume` triggers automatic relabelling. If you see permission errors, add `--security-opt label=disable` to the `podman run` command (appropriate for a dev workstation; review for production CI).

### Slow builds on QEMU

QEMU user-mode emulation adds ~3–5× overhead on compute-bound tasks. Mitigate by:

- Enabling ccache (already configured in the images above).
- Parallelising with `-j$(nproc)`.
- Using `--platform linux/arm64` only for the _final_ link/package stage and running compilation steps natively (Approach B cross-compiler) for maximum throughput.

---

## 9. Security & Maintenance Notes

**Image hygiene**

```bash
# Remove intermediate build layers that are no longer referenced
podman image prune -f

# Check for outdated base image
podman pull menci/archlinuxarm:latest
podman build --pull=always --tag arm-qt5-builder:latest .
```

**Secrets**  
Never embed signing keys, deploy tokens, or device credentials in the `Containerfile`. Use Podman secrets:

```bash
podman secret create my-signing-key ~/.ssh/id_ed25519
podman run --secret my-signing-key ...
```

**Rootless operation**  
The `--userns=keep-id` flag used throughout this guide maps your UID into the container without requiring `root`. This is the recommended Podman workflow on Arch Linux and avoids accidental privilege escalation.

**Regular pacman updates**  
Pin your image build date via a `LABEL build.date` and rebuild monthly. ARM security patches for `openssl`, `glibc`, and `libxcb` arrive through the normal Arch Linux ARM rolling release cycle.

```bash
# Rebuild with today's packages (no layer cache for system update)
DOCKER_BUILDKIT=1 podman build \
    --no-cache \
    --platform linux/arm64 \
    --tag "arm-qt5-builder:$(date +%Y%m%d)" \
    --tag arm-qt5-builder:latest \
    .
```


---


# ARM Qt Cross-Compilation Container - Testing & Troubleshooting Guide

## Container Setup (Reference)

```dockerfile
FROM archlinux:latest

RUN pacman -Syu --noconfirm && pacman -S --noconfirm \
    base-devel git cmake ninja \
    aarch64-linux-gnu-gcc aarch64-linux-gnu-binutils \
    qt5-base qt5-tools qt6-base qt6-tools \
    pkgconf

ENV CROSS_TRIPLE=aarch64-linux-gnu
ENV PATH="/opt/cross/bin:${PATH}"

WORKDIR /build
```

Build & run:
```bash
podman build -t arm-qt-cross .
podman run -it --rm -v $(pwd)/src:/build/src arm-qt-cross bash
```

---

## Test 1: Qt5 "Hello World" (Widgets)

**main.cpp**
```cpp
#include <QApplication>
#include <QLabel>

int main(int argc, char *argv[]) {
    QApplication app(argc, argv);
    QLabel label("Hello from Qt5 ARM!");
    label.resize(300, 100);
    label.show();
    return app.exec();
}
```

**CMakeLists.txt**
```cmake
cmake_minimum_required(VERSION 3.16)
project(Qt5HelloARM)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_AUTOMOC ON)

find_package(Qt5 COMPONENTS Widgets REQUIRED)

add_executable(qt5hello main.cpp)
target_link_libraries(qt5hello Qt5::Widgets)
```

**Toolchain file (toolchain-arm-qt5.cmake)**
```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(CMAKE_C_COMPILER aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)

set(CMAKE_FIND_ROOT_PATH /usr/aarch64-linux-gnu/qt5)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

**Build commands**
```bash
mkdir build-qt5 && cd build-qt5
cmake -G Ninja -DCMAKE_TOOLCHAIN_FILE=../toolchain-arm-qt5.cmake ..
ninja
file qt5hello   # should report ARM aarch64 ELF
```

---

## Test 2: Qt6 "Hello World" (Widgets + QML)

**main.cpp (Widgets variant)**
```cpp
#include <QApplication>
#include <QPushButton>

int main(int argc, char *argv[]) {
    QApplication app(argc, argv);
    QPushButton button("Hello from Qt6 ARM!");
    button.resize(300, 100);
    button.show();
    return app.exec();
}
```

**main_qml.cpp (QML variant)**
```cpp
#include <QGuiApplication>
#include <QQmlApplicationEngine>

int main(int argc, char *argv[]) {
    QGuiApplication app(argc, argv);
    QQmlApplicationEngine engine;
    engine.loadFromModule("HelloARM", "Main");
    if (engine.rootObjects().isEmpty())
        return -1;
    return app.exec();
}
```

**Main.qml**
```qml
import QtQuick
import QtQuick.Controls

ApplicationWindow {
    visible: true
    width: 320; height: 240
    title: "Qt6 ARM QML Test"

    Label {
        anchors.centerIn: parent
        text: "Hello from Qt6 QML on ARM!"
    }
}
```

**CMakeLists.txt (Qt6, QML)**
```cmake
cmake_minimum_required(VERSION 3.16)
project(Qt6HelloARM)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_AUTOMOC ON)

find_package(Qt6 COMPONENTS Quick REQUIRED)
qt_standard_project_setup()

qt_add_executable(qt6hello main_qml.cpp)

qt_add_qml_module(qt6hello
    URI HelloARM
    VERSION 1.0
    QML_FILES Main.qml
)

target_link_libraries(qt6hello PRIVATE Qt6::Quick)
```

**Toolchain file (toolchain-arm-qt6.cmake)**
```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(CMAKE_C_COMPILER aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)

set(CMAKE_FIND_ROOT_PATH /usr/aarch64-linux-gnu/qt6)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)

# Important for Qt6: host tools (qmlcachegen, moc) must run on the build host
set(QT_HOST_PATH /usr)
```

**Build commands**
```bash
mkdir build-qt6 && cd build-qt6
cmake -G Ninja -DCMAKE_TOOLCHAIN_FILE=../toolchain-arm-qt6.cmake ..
ninja
file qt6hello
```

---

## Verification on Target / Emulation

Without real hardware, use QEMU:
```bash
podman run --rm -v $(pwd)/build-qt5:/app \
    --platform linux/arm64 \
    arm64v8/archlinux qemu-aarch64-static /app/qt5hello -platform offscreen
```

Check binary architecture and dynamic dependencies:
```bash
aarch64-linux-gnu-readelf -h qt5hello | grep Machine
aarch64-linux-gnu-objdump -p qt5hello | grep NEEDED
```

---

## Troubleshooting Guide

### Container-Level Issues

**Issue: `podman build` fails on package install**
- Cause: stale Arch mirrors or `pacman` keyring issues.
- Fix:
```bash
pacman-key --init
pacman-key --populate archlinux
pacman -Syyu --noconfirm
```

**Issue: Cross compiler not found (`aarch64-linux-gnu-gcc: command not found`)**
- Confirm package installed: `pacman -Qs aarch64-linux-gnu`
- If using AUR toolchains, ensure an AUR helper (yay/paru) ran during build, or pull prebuilt binaries from a toolchain repo.
- Verify `PATH` includes the cross-toolchain bin directory.

**Issue: Permission denied on mounted volumes**
- Cause: SELinux context or UID mismatch with Podman rootless mode.
- Fix: add `:Z` to volume mounts:
```bash
podman run -v $(pwd)/src:/build/src:Z ...
```
- Or run with `--userns=keep-id`.

**Issue: Container can't resolve DNS / package downloads fail**
- Check `/etc/resolv.conf` inside container.
- Try `podman run --dns=8.8.8.8 ...`
- For rootless Podman, ensure `slirp4netns` is installed.

---

### Qt5/Qt6 Build Issues

**Issue: `Could not find a package configuration file provided by "Qt5"`**
- `CMAKE_FIND_ROOT_PATH` is wrong or Qt ARM libraries aren't installed.
- Check: `find / -name "Qt5Config.cmake" 2>/dev/null`
- Ensure ARM Qt is under `/usr/aarch64-linux-gnu/qt5/lib/cmake/Qt5`.

**Issue: `moc`/`uic`/`rcc` errors — "wrong ELF class" or host tool not executed**
- Cause: CMake is trying to run the **target** moc binary on the host.
- Fix: ensure host Qt tools (x86_64) are on PATH and `CMAKE_FIND_ROOT_PATH_MODE_PROGRAM` is `NEVER`, so CMake uses host moc/uic/rcc instead of cross ones.

**Issue: Qt6-specific — `qt_add_qml_module` fails / `qmlcachegen` errors**
- Qt6 requires a **host build of Qt** for code generators (`qmlcachegen`, `moc`).
- Set `QT_HOST_PATH` to the location of the native Qt6 install (the x86_64 `qt6-base`/`qt6-declarative` packages).
- If host Qt6 isn't installed, `pacman -S qt6-base qt6-declarative` (native packages, separate from ARM sysroot ones).

**Issue: Linker errors `cannot find -lQt5Core` / `-lQt6Core`**
- ARM Qt libraries missing or not in linker search path.
- Verify: `aarch64-linux-gnu-pkg-config --libs Qt5Core`
- Ensure `PKG_CONFIG_PATH`/`PKG_CONFIG_LIBDIR` points to the ARM sysroot's pkgconfig directory, e.g.:
```bash
export PKG_CONFIG_LIBDIR=/usr/aarch64-linux-gnu/qt5/lib/pkgconfig
```

**Issue: Binary builds but `file` shows x86-64 instead of ARM**
- Toolchain file wasn't applied — check CMake cache:
```bash
grep CMAKE_CXX_COMPILER build-qt5/CMakeCache.txt
```
- Delete build dir and re-run cmake with `-DCMAKE_TOOLCHAIN_FILE=...`.

**Issue: Plugin loading errors at runtime (`This application failed to start because no Qt platform plugin could be initialized`)**
- ARM `platforms/` plugin directory not deployed alongside the binary.
- Set `QT_QPA_PLATFORM_PLUGIN_PATH` and copy `plugins/platforms/libqxcb.so` (or `libqeglfs.so`/`libqlinuxfb.so` for embedded targets) to the deployment.
- For headless testing use `-platform offscreen`.

**Issue: QML modules not found at runtime (Qt6)**
- `QML2_IMPORT_PATH` / `QML_IMPORT_PATH` not set on target.
- Deploy the `qml/` directory from the ARM Qt6 install alongside the app, and set:
```bash
export QML2_IMPORT_PATH=/path/to/deployed/qml
```

---

### Runtime/Emulation Issues

**Issue: `qemu-aarch64-static: Could not open '/lib/ld-linux-aarch64.so.1'`**
- Missing ARM dynamic linker/libs in the rootfs being used for emulation — need a proper aarch64 sysroot or run inside an `arm64v8` container, not the build container.

**Issue: Segfault only under QEMU, works natively**
- Often floating-point ABI mismatch (`-mfloat-abi=hard` vs `soft`) or QEMU version too old for newer ARM instructions — try `qemu-aarch64-static -cpu max`.

---

## Quick Sanity Checklist
1. `aarch64-linux-gnu-gcc --version` works inside container
2. Both **host** (x86_64) and **target** (ARM) Qt installs exist separately
3. `CMAKE_FIND_ROOT_PATH` points only to ARM sysroot, programs mode = `NEVER`
4. Qt6 only: `QT_HOST_PATH` set to native Qt6
5. `file <binary>` confirms `ELF 64-bit ... ARM aarch64`
6. `ldd`/`objdump -p` shows expected ARM `.so` dependencies (or none, if static)