# Installation on Linux

Installing on Linux, like installing on Windows, consists of two steps:

1) Installing the C++ toolchain
2) Installing the Rust toolchain

## Installing the C++ Toolchain

We will use GCC as the C++ toolchain.

On different Linux distributions, GCC is installed differently, but typically it is installed using the system’s package manager.

On Linux distributions that use the `apt` package manager (Ubuntu, Debian, Linux Mint), GCC can be installed with the following command:

```
sudo apt install gcc
```

On distributions that use the `yum` and/or `dnf` package managers (CentOS, Fedora, Oracle Linux), use:

```
sudo yum install gcc
```

или 

```
sudo dnf install gcc
```

## Installing Rust

After GCC is installed, we need to install the `rustup` utility, which manages the Rust environment.

To install `rustup`, run the following command:

```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

This command downloads and runs the `rustup.sh` script, which in turn downloads the Rust toolchain and places it in the `~/.rustup` directory.

> [!TIP]
> If you encounter issues while installing rustup, refer to the official guide: https://www.rust-lang.org/tools/install

Run the command `rustc --version`, which prints the version of the currently installed Rust compiler. If everything was successful, the output should look similar to this:

```
$ rustc --version
rustc 1.89.0 (29483883e 2025-08-04)
```
