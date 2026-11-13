# Alpine Linux - Chromium cross-compilation

This document describes how to cross-compile Chromium for Alpine Linux.

This version covers x86_64-to-armv7 cross-compilation on Alpine Linux
3.24.4.

## Overview

To cross-compile Chromium for armv7 from x86_64, three toolchains are
required:

- x86_64 host toolchain (CBUILD): used for host tools
- x86_64 -> armv7 default toolchain (CHOST): used for target binaries
- x86_64 -> x86 V8 snapshot toolchain: used to generate the V8 memory
  snapshot

Note: in abuild, the host toolchain is called CBUILD, while the default
toolchain is called CHOST. Do not confuse CHOST with the host
toolchain.

The critical requirement is that the default toolchain and the V8
snapshot toolchain must have the same pointer size (both 32-bit), and
that the binaries produced by the V8 toolchain must be executable on
the host system. This is the main reason that cross-compiling Chromium
on Alpine Linux is non-trivial:

- Alpine Linux generally does not support executing x86 binaries on an
  x86_64 system without a chroot.
- Alpine Linux's build system does not allow cross-compilation with
  three different toolchains at the same time.

This must be handled manually. Once these steps are complete, the build
process proceeds largely as expected.

The build also requires compiler-rt libraries for each target
architecture and patches for the Chromium toolchain setup. After these
requirements are satisfied, the build should complete successfully.

## Prerequisites

All steps in this guide are performed inside a Docker container, so
Docker must be installed.

A substantial amount of free disk space is also required.

## Step 1. Prepare the abuild environment

Create a working directory and copy the patch files into it.

```sh
WORKSPACE_DIR=./workspace  # you can use your own path
mkdir -p "$WORKSPACE_DIR"
cp *.patch "$WORKSPACE_DIR"
```

Start a new Docker container and mount the working directory.

```sh
docker run --rm -it -v "$WORKSPACE_DIR":/workspace -w /workspace alpine:3.24
```

Update the system.

```sh
apk upgrade
```

Install the required packages:

- `alpine-sdk` - for building Alpine Linux packages
- `doas` - for privilege elevation
- `nano` - text editor
- `bash` - shell

```sh
apk add alpine-sdk doas nano bash
```

Configure abuild. Pay close attention to the following settings:

- `USE_CCACHE` - recommended to be enabled; it speeds up rebuilds but
  increases disk usage
- `PACKAGER` - set to your name and email address
- `MAINTAINER` - set to the same value

```sh
nano /etc/abuild.conf
```

Enable `doas` for the `wheel` group.

```sh
sed -i "/:wheel/s/^# //" /etc/doas.conf
```

Create a new build user.

```sh
addgroup -g 1000 build
adduser -G build -u 1000 -h /workspace -H -D -s /bin/bash build
echo -e "build\nbuild" | passwd build
adduser build abuild
adduser build wheel
```

Log in as the build user.

```sh
su - build
```

Generate your development keys. The user password is `build`.

```sh
abuild-keygen -i -a
```

Clone aports.

```sh
git clone https://github.com/alpinelinux/aports.git -b 3.24-stable --depth 1
```

Apply the patches.

```sh
(cd aports; git apply ../*.patch)
```

The abuild environment is now ready.

## Step 2. Prepare the cross-compilation tools

Set up the x86 sysroot. This command may fail while building fakeroot;
this can be ignored.

```sh
CBUILDROOT="$HOME/sysroot-x86" ./aports/scripts/bootstrap.sh x86
```

Set up the armv7 sysroot. This command may fail while building
binutils; this can be ignored.

```sh
CBUILDROOT="$HOME/sysroot-armv7" ./aports/scripts/bootstrap.sh armv7
```

Remove the target-architecture packages and replace them by fetching
the required packages from the official repositories.

```sh
rm -rf packages/main/{x86,armv7}
cp /etc/apk/repositories sysroot-armv7/etc/apk/
doas apk --arch armv7 --root sysroot-armv7 add --allow-untrusted alpine-keys
cp /etc/apk/repositories sysroot-x86/etc/apk/
doas apk --arch x86 --root sysroot-x86 add --allow-untrusted alpine-keys
```

Install Clang and configure it to support the required targets.

```sh
doas apk --repository packages/main add clang build-base-x86 build-base-armv7
doas apk --arch armv7 --root sysroot-armv7 add build-base
doas cp /etc/clang22/x86_64-alpine-linux-musl.cfg /etc/clang22/armv7-alpine-linux-musleabihf.cfg
echo --sysroot="$HOME/sysroot-armv7" | doas tee -a /etc/clang22/armv7-alpine-linux-musleabihf.cfg
doas apk --arch x86 --root sysroot-x86 add build-base
doas cp /etc/clang22/x86_64-alpine-linux-musl.cfg /etc/clang22/i586-alpine-linux-musl.cfg
echo --sysroot="$HOME/sysroot-x86" | doas tee -a /etc/clang22/i586-alpine-linux-musl.cfg
```

Check that Clang works.

```sh
cat > hello.cc << "EOF"
#include <iostream>
int main() {
    std::cout << "Hello World" << std::endl;
    return 0;
}
EOF
clang++ --target=armv7-alpine-linux-musleabihf -o hello.armv7 hello.cc
clang++ --target=i586-alpine-linux-musl -o hello.x86 hello.cc
```

Ensure that x86 binaries can be executed on the host system.

```sh
doas ln -sv "$HOME/sysroot-x86/lib/ld-musl-i386.so.1" /lib
echo "$HOME/sysroot-x86/lib:$HOME/sysroot-x86/usr/lib" | doas tee /etc/ld-musl-i386.path
```

Verify that the configuration works.

```sh
./hello.x86
```

## Step 3. Build Chromium

First, build the patched compiler-rt packages.

```sh
cd aports/main/llvm-runtimes/
abuild -r
BOOTSTRAP=1 CHOST=x86 abuild -r
BOOTSTRAP=1 CHOST=armv7 abuild -r
cd -
```

Install the Chromium dependencies for the x86 toolchain.

```sh
doas apk --arch x86 --root sysroot-x86 add \
  $(sed -n '/makedepends_host="/,/"/p' \
   aports/community/chromium//APKBUILD | tr -d '\n' | cut -d\" -f2 | \
   sed 's/$_llvmver/22/g')
```

`libpulse` uses absolute runpaths, so they must be replaced.

```sh
doas apk add patchelf
doas patchelf --set-rpath "$HOME/sysroot-x86/usr/lib/pulseaudio" sysroot-x86/usr/lib/libpulse*.so
```

Now the Chromium package can be built.

```sh
cd aports/community/chromium/
CHOST=armv7 abuild -r
```

You may now exit the Docker container.

The resulting packages are located in`$WORKSPACE_DIR/packages/community/armv7`.
