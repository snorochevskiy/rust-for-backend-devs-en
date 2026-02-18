# Cross-Compilation

## Different Platforms

When you build a Rust program, it compiles by default into machine code for the platform you are currently working on and links with the standard C library installed on your system. Therefore, if you are working on Windows or macOS, you cannot simply build a binary executable, copy it to a Linux server, and run it there.

Furthermore, if you are working on Linux but the GLIBC version on your system is older than the GLIBC version on the target Linux server, the binary executable built for your system will likely fail to run on the target server.

> [!TIP]
> **GLIBC** is the standard C library used by default in most Linux distributions.

There are two ways to build a Rust program for a different platform:

* Build in an environment identical to the target platform. This could be a separate physical machine, a Docker container, or a virtual machine running the same environment as the target platform.
* Perform cross-compilation for the target platform.

Building on a different machine or in a Docker container is fairly straightforward. We will delve into the details of cross-compilation later in this chapter.

## Challenges of Cross-Compilation

To build a Rust program for another platform, two things must happen:

* The source code must be compiled into an object file (machine code in a specific format) for the target platform.
* The object files must be linked with the target platform's standard C library.

While compiling code into object files for another platform is easy (you just specify the appropriate compilation option), linking with the target platform's C library is more complicated because you need:

* A linker capable of creating executable files for the target platform.
* The C library from the target platform.

For example, building a Windows x86_64 program from Linux Ubuntu is relatively easy, but only because the Ubuntu package repository contains the necessary package with the C library and a linker that knows how to build Windows executables:

```bash
# Install the MinGW C++ toolchain for Windows, which contains the linker and C library
sudo apt install gcc-mingw-w64-x86-64
# Add the Rust toolchain for cross-compiling to Windows x86_64
rustup target add x86_64-pc-windows-gnu
# Build the application for Windows x86_64 using the C library from MinGW w64
cargo build --release --target x86_64-pc-windows-gnu
```

However, building a program for Linux from Windows is more difficult: while compiling the object files is still easy, obtaining the correct linker and the required version of GLIBC is a challenge.

## Zig-build

At this point, it's worth taking a brief detour to mention the Zig programming language. Zig is a compiled language that follows the same cross-platform build rules: it requires a linker and a standard C library for the target platform. However, the unique feature of the Zig SDK is that it contains linkers and standard C libraries for all major platforms out of the box. Consequently, to cross-compile a Zig program, you don't need to install anything extra—the Zig SDK already has everything you need.

This feature did not go unnoticed by the Rust community, who soon created a utility called [zigbuild](https://crates.io/crates/cargo-zigbuild). This tool leverages the linker and C library from the Zig SDK to cross-compile Rust programs.

***

First, install the Zig SDK:

1. СDownload the latest stable version of the Zig SDK from the official website:\
   [https://ziglang.org/download/](https://ziglang.org/download/). You should receive a `.tar.xz` or `.zip` archive with a name like `zig-x86_64-linux-0.14.1.tar.xz`.
2. Unpack the archive to a location of your choice.
3. Add the path to the root Zig SDK folder (the folder containing the `zig` executable) to your system path. On Unix-like systems, this is the `PATH` variable; on Windows — `Path`.

Now, install the zigbuild utility by running:

```
cargo install --locked cargo-zigbuild
```

> [!WARNING]
> If you are on Windows and using MinGW instead of Visual C++, you might encounter linking issues with system Windows libraries when building zigbuild. In this case, it is recommended to either install Visual C++ or use a Docker image with zigbuild.

### Building for Windows

Suppose you are working on Linux or macOS and want to build a program for Windows using zigbuild. You need to:

1\) Add the Rust toolchain for Windows:

```
rustup target add x86_64-pc-windows-gnu
```

2\) Build the program using zigbuild:

```
cargo zigbuild --release --target x86_64-pc-windows-gnu
```

After this, a `target/x86_64-pc-windows-gnu/release` directory should appear, containing the executable `.exe` file.

### Building for Linux

Now let's look at an example of building a program for Linux from Windows.

1\) Add the Linux toolchain with GLIBC:

```
rustup target add x86_64-unknown-linux-gnu
```

2\) Build the program:

```
cargo zigbuild --release --target x86_64-unknown-linux-gnu
```

Keep in mind that GLIBC has different versions. The version of GLIBC you link against must not be newer than the version of GLIBC on the system where you plan to run your program.

To check the GLIBC version on a Linux system, use the `ldd` command:

```
$ ldd --version
ldd (Ubuntu GLIBC 2.39-0ubuntu8.6) 2.39
Copyright (C) 2024 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Written by Roland McGrath and Ulrich Drepper.
```

Zig versions 0.12, 0.13, 0.14, and 0.15 link with GLIBC version 2.28 by default.

To specify a specific GLIBC version during assembly, append it after a dot in the target name. For example, to link with GLIBC 2.34:

```
cargo zigbuild --release --target x86_64-unknown-linux-gnu.3.34
```

> [!TIP]
> It is also worth noting that Windows users can use WSL (Windows Subsystem for Linux) to build programs for Linux.

### zigbuild Docker

You can also use the ready-made zigbuild Docker image, which already contains Rust, the Zig SDK, and zigbuild.

For example, the following command builds a program for Windows:

```
docker run --rm -it -v $(pwd):/io -w /io ghcr.io/rust-cross/cargo-zigbuild \
  cargo zigbuild --release --target x86_64-pc-windows-gnu
```

## MUSL

Sometimes you don't know which version of GLIBC will be used on the target system. Some distributions (like Alpine Linux) don't use it at all.

For these situations, there is an alternative standard C library called MUSL. Unlike GLIBC, which programs link to dynamically, MUSL is designed for static linking. This means that during the build process, the machine code of the MUSL library is placed directly into the program's binary file, eliminating the need for an external dynamic C library.

A separate Rust toolchain handles building for Linux with MUSL. Install it with:

```
rustup target add x86_64-unknown-linux-musl
```

To build a program with MUSL, you can either use zigbuild (the Zig SDK includes MUSL) or simply install the MUSL package on your system if available.

For example, on Ubuntu Linux, you would need to install the `musl-tools` and `musl-dev` packages:

```
sudo apt install musl-tools musl-dev
```

After that, you can build your program for Linux linked with MUSL:

```
cargo build --release --target x86_64-unknown-linux-musl
```

However, using zigbuild is often more convenient as it works on any system and requires no additional package installation:

```
cargo zigbuild --target x86_64-unknown-linux-musl --release
```

***

You might wonder: "If MUSL is so convenient, why not use it for all Linux builds?"

If your application does not use any external dynamic libraries, MUSL is an excellent default choice.

However, if your application uses a dynamic library linked with GLIBC, building your application with MUSL will likely fail to produce the desired result: the application will still require GLIBC, just for that external library.

For example, at the time of writing (Rust 1.92), the Rust ecosystem lacks a driver for the Oracle database written entirely in Rust. The existing [oracle](https://crates.io/crates/oracle) crate is simply a wrapper around a library written in C and linked with GLIBC. Since the library code is closed-source, you cannot compile it yourself to link it statically with your Rust program. Therefore, if your application needs to work with an Oracle database, you will have to use GLIBC.

