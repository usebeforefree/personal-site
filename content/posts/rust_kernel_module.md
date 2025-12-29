+++
title = "Rust kernel modules tutorial"
date = 2025-12-29
+++

# Rust in the kernel

Year 2025 will be remembered for the addition of Rust in the Linux kernel. It's really exciting to finally see another language in kernel, especially for the younger folks out there, who seem to be less and less interested in old fashioned programming languages such as C.

In this short tutorial I'll show you how to create a development environment for Rust kernel modules, since it's still in the experimental phase. I'll also show you an example of a kernel module in Rust, which we will load into the kernel itself.

## Choosing a distro

As I am about to learn, this isn't really that easy per se.

The first problem is that, while Rust in the kernel is supported, it's still in very early stages, and very experimental. Thus, most distros do not support it. Though some do, like the latest Ubuntu 25, or some of the latest Fedora spins.

Check if your distro supports it like so:
```bash
grep CONFIG_RUST /boot/config-$(uname -r)
```

If the output contains the `CONFIG_RUST=y` flag, it means that Rust is in fact compiled in your kernel.

Though, that doesn't mean you can compile and load your own kernel modules, which would be an essential part of the development process.

To check whether your kernel has the needed build artifacts, list the `rust` directory. If it's empty or has a `Makefile` only, you're out of luck, and have to recompile your kernel.
```bash
ls /usr/src/kernels/6.17.1-300.fc43.x86_64/rust/
```

Since I wasn't been able to find a distro that supports this out of the box, the only viable option was to pick some distro and recompile it's kernel with a proper config. My choice fell down to Fedora Server 43, since it follows the kernel development pretty fast, is light, and stable.

This is the first reason I really do not recommend doing this on your development/personal machine, since you are in a very high risk of just nuking your system while recompiling. I've opted to do all of this work on my home server, since spinning up VMs in Proxmox is fairly low effort. I highly recommend a setup like this if possible. I though I knew what I was doing, yet bricked about 6 kernels in the process of trying to compile them.

Also, you have to use VMs and not containers, since VMs have their own kernel, while containers do not, they share it with the host machine.

While we are at it, this is a perfect time to make a snapshot or backup of your kernel, just in case things go south, you can revert back without wasting effort.

The second reason I do not recommend doing this on a machine you use day to day, is because running random code in the kernel space is potentially dangerous. Especially while developing it.

## Obtaining kernel source and other dependencies

Since we are recompiling the kernel, we have to fetch the actual source. Firstly, we need to check what kernel version we are on:

```bash
uname -r
```
```text
Output:
  6.17.1-300.fc43.x86_64
```

Rust support was firstly added in 6.1, though I recommend using as newer releases as possible.

We are going to fetch source for the same version of the kernel we already have, in order to avoid potential edge cases as much as possible.

```bash
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.17.1.tar.gz
tar -xf linux-6.17.1.tar.gz
```

Before we proceed, we need to fetch our build dependencies, which are defined here `Documentation/rust/quick-start.rst`.

```bash
dnf install rust rust-src bindgen-cli rustfmt clippy
```

With the addition of three more dependencies in my case:

```bash
dnf install flex bison ncurses-devel
```

To confirm you have all needed dependencies, inside the kernel source run:

```bash
make rustavailable
```
```text
Output:
  Rust is available!
```

## Configuring the build

Now, the kernel is highly configurable, and we will use the preferences we already have on our default precompiled kernel we are currently on, with the addition of Rust support if needed. This should minimize compiling a bunch of stuff we don't need, or even worse, not compiling things we do need.

Keep in mind that kernels vary in size, Fedoras kernel is not the smallest. Smaller configs mean shorter compile times. Smaller kernels also mean less features. I tried being smart by removing some "unneeded" features, making the kernel not boot afterwards. That's why the safest way is to actually copy your current kernel's configuration. It is worth the wait, if you are not sure what you're doing.

To do so, run these commands:

```bash
cp /boot/config-$(uname -r) .config
make olddefconfig -j$(nproc)
```
```text
Output:
  . . .
  #
  # configuration written to .config
  #
```

Now to actually add our Rust support config, we should check the `General setup -> [*] Rust support` flag, if it's already not checked. This is done via this really cool TUI:
```bash
make menuconfig -j$(nproc)
```

While we are here, I also unchecked these settings, since they solved some problems I had with booting the kernel afterwards:
```text
[ ] EFI stub support
[ ] Disable signature verification sing multiple keyrings
```

When you're finished, save the changes and we can go on to compile the kernel:

```bash
make LLVM=1 -j$(nproc)
```

If all went well, we can finally install the modules, followed by the kernel itself:
```bash
make modules_install -j$(nproc)
make install -j$(nproc)
```

When finished, you can check if grub has your kernel ready for the next boot:

```bash
grubby --info=ALL | less
```
```text
Output:
  vmlinuz-6.17.1-300.fc43.x86_64
  vmlinuz-6.17.1                <----- Our new kernel
  vmlinuz-0-rescue-...
```

Restart your machine and select the new kernel inside of `grub`, if it doesn't boot by itself.

## Compiling and loading a generic Rust kernel module

I recommend starting from this super simple template/demo for Rust kernel modules: https://github.com/Rust-for-Linux/rust-out-of-tree-module/tree/00b5a8ee2bf53532d115004d7636b61a54f49802

Simply clone the repo and make it:
```bash
git clone https://github.com/Rust-for-Linux/rust-out-of-tree-module.git
cd rust-out-of-tree-module
git checkout 00b5a8ee2bf53532d115004d7636b61a54f49802
make KDIR=/usr/src/linux-6.17.1/ LLVM=1 CC=gcc
```

Load it into the kernel using:
```bash
insmod rust_out_of_tree.ko
```

Remove the module like so:
```bash
rmmod rust_out_of_tree
```

Check kernel logs:
```bash
dmesg | tail -n 20
```
```text
Output:
  . . .
  [   53.842591] rust_out_of_tree: loading out-of-tree module taints kernel.
  [   53.842604] rust_out_of_tree: module verification failed: signature and/or required key missing - tainting kernel
  [   53.843598] rust_out_of_tree: Rust out-of-tree sample (init)
  [   82.528790] rust_out_of_tree: My numbers are [72, 108, 200]
  [   82.528838] rust_out_of_tree: Rust out-of-tree sample (exit)
```

Congrats, you made it!

Other than shear curiosity, I've done all this research so I can actually implement a Rust loadable kernel module myself.

## Ending note

I am no kernel or Rust expert, this short tutorial came to see the light of day because I haven't found much info on this topic, thus having to hack around for 3 days straight.

If I said some dumb shit, either write to me `marko03kostic@protonmail.com` or literally make a PR on `https://github.com/usebeforefree/personal-site`.
