---
title: Will it run Nerves?
sub_title: Bringing Nerves to New or Exotic Devices
author: Marc Lainez
theme:
  name: dark
---

<!-- jump_to_middle -->

Part I
===

How a Linux system works
and where Nerves fits in

<!-- end_slide -->

The three pillars of a Linux system
===

To run Linux on **any** device, you at least these three things:

```
  +-------------------+    +-------------------+    +-------------------+
  |    Bootloader     |    |      Kernel       |    |      Rootfs       |
  +-------------------+    +-------------------+    +-------------------+
  |                   |    |                   |    |                   |
  | Initializes HW    |    | Manages processes |    | Programs,         |
  | Loads kernel into |--->| Memory mgmt       |--->| libraries,        |
  | memory, starts it |    | Device drivers    |    | init scripts,     |
  |                   |    | Filesystems       |    | config, your app  |
  +-------------------+    +-------------------+    +-------------------+
```

**Bootloader**: first code that runs, sets up just enough to load the kernel.

**Kernel**: the core of the OS -- talks to hardware so userspace doesn't have to.

**Rootfs**: the "userspace" -- everything that is not the kernel.

Without a bootloader and a kernel for your device, you won't go far.

<!-- end_slide -->

The boot sequence
===

```
  Power on
     |
     v
  +--------------+
  |  Bootloader  |  Reads kernel + DTB from storage
  +--------------+  (eMMC, SD card, SPI flash, ...)
     |
     |  Passes control + kernel command line
     v
  +--------------+
  |    Kernel    |  Initializes all hardware,
  +--------------+  mounts the root filesystem
     |
     |  root=/dev/mmcblk0p2  rootfstype=ext4
     v
  +--------------+
  |    Rootfs    |  Runs /sbin/init as PID 1
  +--------------+  (first userspace process)
     |
     v
  Init scripts, services, your application
```

The kernel command line is how the bootloader tells the kernel
where to find the rootfs: `root=/dev/mmcblk0p2`

<!-- end_slide -->

What the bootloader does
===

The bootloader's job is simple but critical:

1. Initialize the bare minimum: CPU, RAM controller, storage
2. Find the kernel (and device tree) on a known partition
3. Load them into RAM
4. Pass a command line and jump to the kernel entry point

**Common bootloaders:**

```
  +------------+   +------+   +---------+   +--------+
  |  U-Boot    |   | lk2nd|   |  Barebox|   |  GRUB  |
  +------------+   +------+   +---------+   +--------+
   Most common      Qualcomm    Modern        x86/EFI
   in embedded      Android     alternative
```

Some devices chain multiple bootloaders. On eMMC devices, the SoC boot ROM
loads a **preloader** from a dedicated `boot0` hardware partition, which
then loads the next stage. The preloader is usually **signed**.

```
  SoC ROM --> boot0 (preloader, signed) --> U-Boot/lk2nd --> Kernel
```

You usually only interact with the last stage.

<!-- end_slide -->

What the kernel does
===

The kernel is the bridge between hardware and your applications.

```
  +---------------------------------------------+
  |               Userspace                     |
  |   (your app, libraries, shell, services)    |
  +---------------------------------------------+
                      | system calls
  +---------------------------------------------+
  |                  Kernel                     |
  |                                             |
  |   Process scheduler   Memory management     |
  |   Filesystem layer    Network stack         |
  |   Device drivers      Security modules      |
  |                                             |
  +---------------------------------------------+
                      |
  +---------------------------------------------+
  |               Hardware                      |
  |   CPU, RAM, eMMC, GPIO, I2C, SPI, USB ...   |
  +---------------------------------------------+
```

The kernel needs to support your specific **SoC** and **peripherals**.
This is done through drivers (built-in or loadable modules)
and a **device tree** that describes the hardware layout.

<!-- end_slide -->

The device tree
===

The **device tree** (DTB) is a data structure that describes hardware
so the kernel doesn't need to hardcode board-specific details.

```
  / {
      model = "Pine64";
      compatible = "pine64,pine64-plus", "allwinner,sun50i-a64";

      memory@40000000 {
          reg = <0x40000000 0x40000000>;     // 1GB at this address
      };

      soc {
          uart0: serial@1c28000 {            // UART at this MMIO address
              compatible = "snps,dw-apb-uart";
              reg = <0x01c28000 0x400>;
          };

          mmc0: mmc@1c0f000 {               // SD card controller
              compatible = "allwinner,sun50i-a64-mmc";
          };
      };
  };
```

- Usually comes with the kernel source or in a separate partition
- Without the right DTB for your board, the kernel boots but **nothing works**

<!-- end_slide -->

What is a rootfs?
===

The root filesystem is a directory tree containing everything
the kernel needs to start userspace:

```
  /
  +-- /sbin/init         <-- First process (PID 1)
  +-- /bin/              <-- Essential binaries (sh, ls, mount...)
  +-- /lib/              <-- Shared libraries (libc, ...)
  +-- /etc/              <-- Configuration files
  |     +-- inittab      <-- What init should start
  |     +-- fstab        <-- What to mount
  +-- /dev/              <-- Device nodes
  +-- /proc/             <-- Kernel process info (mount point)
  +-- /sys/              <-- Kernel device info (mount point)
  +-- /tmp/              <-- Temporary files
```

PID 1 is special: it is the **ancestor of all processes**.
If PID 1 dies, the kernel panics.

Traditional init systems (systemd, OpenRC, BusyBox init) run
**shell scripts** to bring up the system: mount filesystems,
configure network, start services...

<!-- end_slide -->

Build systems vs distributions
===

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

**Build system**
*(from source)*

```
  Source code
      |
      v
  Cross-compile
  everything
      |
      v
  +----------+
  | Minimal  |
  | image    |
  | ~20-60MB |
  +----------+
```

- Buildroot, Yocto, OpenWrt
- Full control over content
- Reproducible builds
- Purpose-built for one device

<!-- column: 1 -->

**Distribution**
*(prebuilt packages)*

```
  Package repos
      |
      v
  apt install
  everything
      |
      v
  +----------+
  | General  |
  | purpose  |
  | ~500MB+  |
  +----------+
```

- Debian, Ubuntu, Alpine
- Package manager, updates
- Broad hardware support
- Large footprint

<!-- reset_layout -->

For embedded, you want a build system.
**Buildroot** is what Nerves uses underneath.

<!-- end_slide -->

The toolchain
===

You develop on an x86_64 machine but target an ARM board.
You need a **cross-compilation toolchain**: a compiler that
runs on your host but produces binaries for the target.

```
  Your laptop (x86_64)                Target (ARM)
  +---------------------+             +------------------+
  |  arm-linux-gcc      | ==========> | ELF binary       |
  |  arm-linux-ld       |  compiles   | (runs on target) |
  |  arm-linux-libc     |  for ARM    |                  |
  +---------------------+             +------------------+
```



A toolchain includes:

- **Compiler** (GCC or LLVM) configured for the target architecture
- **C library** (glibc, musl, uClibc) -- must match what's on the rootfs
- **Binutils** (linker, assembler, objdump)
- **Kernel headers** -- defines the kernel API available to userspace



The toolchain must be **consistent** across everything you build:
kernel, kernel modules, rootfs packages, and any C code compiled
as part of your application.

Buildroot can build its own toolchain or use an external one.
Nerves provides **prebuilt toolchain packages** for supported architectures.

<!-- end_slide -->

Buildroot in practice
===

Buildroot takes a **defconfig** and produces a complete image.

```
  defconfig                    make                  output/images/
  +-----------------+                                +-----------------+
  | BR2_arm=y       |                                | rootfs.ext4     |
  | BR2_TOOLCHAIN.. | =============================> | zImage          |
  | BR2_PACKAGE_..  |                                | board.dtb       |
  | BR2_LINUX_..    |                                | sdcard.img      |
  +-----------------+                                +-----------------+
```

For custom boards, you use an **external tree** (BR2_EXTERNAL):

```
  my-external/
    board/my_device/
      linux.config               <-- kernel config
      genimage.cfg               <-- image layout
    configs/
      my_device_defconfig        <-- your board config
    package/
      my-custom-package/         <-- extra packages
```

```bash
make BR2_EXTERNAL=../my-external my_device_defconfig && make
```

This is the standard workflow. Nerves builds on top of it.

<!-- end_slide -->

Where Nerves comes in
===

Nerves is a **layer on top of Buildroot** that replaces
the traditional Linux init system with the BEAM VM.

```
  Traditional Linux                 Nerves
  =================                 ======

  +------------------+              +------------------+
  | /sbin/init       |              | erlinit (PID 1)  |
  |  (systemd, etc.) |              |                  |
  +--------+---------+              +--------+---------+
           |                                 |
  +--------v---------+              +--------v---------+
  | Shell scripts    |              | BEAM VM          |
  | - mount fs       |              |                  |
  | - setup network  |              | +------+-------+ |
  | - start services |              | | Your Elixir  | |
  |                  |              | | application  | |
  +------------------+              | +--------------+ |
                                    +------------------+
```

**erlinit** runs as PID 1 instead of systemd/busybox init.
It starts the BEAM, which starts your Elixir application.

There are no shell scripts. No package manager.
No sshd, no networkmanager, no systemctl.

<!-- end_slide -->

What erlinit replaces
===

Everything that init scripts normally do must now happen
in Elixir, because the BEAM is the only thing running.

```
  Traditional Linux              Nerves equivalent
  =====================          =====================

  NetworkManager / ifupdown  --> VintageNet
  systemd services           --> OTP Supervisors
  sshd (OpenSSH)             --> NervesSSH
  /etc/resolv.conf           --> VintageNet
  logging (syslog/journald)  --> RingLogger
  NTP (chrony/ntpd)          --> NervesTimeSync
  ...
```

This is a **fundamental shift**: you don't configure Linux,
you write Elixir code.

The upside: your entire system is one supervised OTP application.
Fault tolerance, hot code upgrades, remote shell -- built in.

<!-- end_slide -->

Anatomy of a Nerves system
===

A Nerves system is a project that defines everything about
the target hardware: kernel, bootloader config, partition layout.

```
  nerves_system_my_board/
    mix.exs                     <-- Elixir project, depends on nerves_system_br
    nerves_defconfig            <-- Buildroot config with Nerves additions
    linux-X.Y.defconfig         <-- Kernel config
    fwup.conf                   <-- How to create/flash/upgrade firmware
    fwup-ops.conf               <-- Runtime ops (revert, factory-reset)
    fwup_include/
      fwup-common.conf          <-- Partition layout (A/B scheme)
    post-build.sh               <-- Buildroot post-build hook
    rootfs_overlay/
      etc/
        erlinit.config          <-- PID 1 config (console, mounts)
        fw_env.config           <-- U-Boot env access
    uboot/
      boot.cmd                  <-- U-Boot boot script
```

The **nerves_defconfig** is a Buildroot defconfig with Nerves-specific
options: Nerves toolchain, squashfs rootfs, erlinit, nbtty, ...

<!-- end_slide -->

From Buildroot defconfig to Nerves system
===

Creating all these files by hand is tedious and error-prone.

**nerves_system_bootstrap** automates it:

```
  +---------------------+     mix nerves.system     +---------------------+
  | Buildroot defconfig |     .bootstrap            | Complete Nerves     |
  | (e.g. pine64)       | ========================> | system project      |
  +---------------------+                           +---------------------+
                                                       |
                          Analyzes defconfig           +-- nerves_defconfig
                          Detects architecture         +-- linux.defconfig
                          Resolves toolchain           +-- fwup.conf
                          Generates fwup config        +-- fwup_include/
                          Creates rootfs overlay       +-- rootfs_overlay/
                          Sets up mix project          +-- mix.exs
                                                       +-- post-build.sh
```

It detects the platform (RPi, Allwinner, x86_64, RISC-V, generic ARM),
chooses the right partition scheme (MBR/GPT), boot mechanism,
and wires up the correct Nerves toolchain automatically.

[Nerves System Bootstrap](https://github.com/mlainez/nerves_system_bootstrap)

<!-- end_slide -->

<!-- jump_to_middle -->

Live demo: nerves_system_bootstrap
===

<!-- end_slide -->

<!-- jump_to_middle -->

Part II
===

When things get weird -- porting Nerves to exotic devices

<!-- end_slide -->

The challenge with exotic devices
===

The "happy path" assumes you have:

```
  +-------------------+     +------------------+     +-----------------+
  | Open bootloader   |     | Mainline kernel  |     | Standard flash  |
  | (U-Boot, etc.)    |     | with DTB support |     | (SD, fastboot)  |
  +-------------------+     +------------------+     +-----------------+
          |                          |                        |
          v                          v                        v
       Replaceable              Well supported           dd / fastboot
       Configurable             Documented               Simple layout
```

Exotic devices break these assumptions:

- **Locked bootloaders** with secure boot (signed images only)
- **Vendor Android kernels** (old, patched, no mainline support)
- **Fixed partition tables** you cannot modify
- **No SD card slot** -- eMMC only, sometimes no fastboot
- **Undocumented hardware** -- no public datasheet for the SoC

<!-- end_slide -->

Android devices: what's different
===

Android devices are Linux underneath, but with a very specific structure:

```
  Android partition layout (typical)
  +------------------+
  | bootloader (BL)  |  Often multi-stage, signed
  +------------------+
  | boot             |  Kernel + ramdisk (Android boot image)
  +------------------+
  | recovery         |  Recovery kernel + ramdisk
  +------------------+
  | system           |  Android OS (/system)
  +------------------+
  | vendor           |  Vendor HAL libraries
  +------------------+
  | userdata         |  User data (largest partition, often 10-25GB)
  +------------------+
```

Key differences from standard embedded Linux:

- Partition table is **fixed** -- some bootloaders refuse to boot if changed
- Kernels are **Android boot images**, not raw zImage/Image files
- Bootloader chain is often **locked and signed**
- The `userdata` partition is your best bet for extra storage

<!-- end_slide -->

Firmware files
===

Many peripherals have their own microprocessor that needs software
to run. The kernel driver loads this software -- a **firmware file** --
into the chip at boot or when the device is first used.

```
  Kernel driver                  Peripheral chip
  +----------------------+        +------------------+
  |  requests firmware   | -----> | Microcontroller  |
  |  from /lib/firmware/ |        | (WiFi, BT, GPU,  |
  +----------------------+        |  modem, DSP...)  |
        |                         +------------------+
        v
  /lib/firmware/device_fw.bin
```



On standard Linux (Debian, Fedora), these come from the
`linux-firmware` package -- a curated open collection.



On **Android devices**, most firmware files are **proprietary**
and not in `linux-firmware`. They live on the `vendor` partition
and are specific to the exact SoC and board revision.



This is where **mainline forks** do critical work: they write
**open-source kernel drivers** that speak the same protocol as
the vendor's proprietary ones, loading the same firmware blobs
but through clean, upstream-style code. Without these reverse-
engineered drivers, you'd be stuck with the vendor's kernel.

Without the right firmware, the kernel driver loads but the
hardware stays silent. No error -- just nothing works.

<!-- end_slide -->

Finding firmware for Android devices
===

Where to find proprietary firmware blobs:



- **On the device itself** -- `adb pull /vendor/firmware/` or
  copy from a running Android/stock system



- **Stock ROM images** -- extract from the vendor's factory image
  (`system.img`, `vendor.img`)



- **Alternative ROMs** -- LineageOS, CalyxOS, /e/OS ship firmware
  in Git repos (e.g. `TheMuppets/proprietary_vendor_*` on GitHub)



You need to include these blobs in your Nerves rootfs overlay
at `/lib/firmware/` so kernel drivers can find them at runtime.



Tip: check which firmware files the kernel actually requests
by watching `dmesg` for lines like:

```
  wmt_drv: Direct firmware load for wmt_soc_patch.bin failed
```

<!-- end_slide -->

The reconnaissance phase
===

Before writing any config, you need answers:

<!-- incremental_lists: false -->

- **What SoC is in the device?** (ARM? which variant?)
- **Is there an existing Linux port?** (PostmarketOS, Mobian, ...)
- **Can you get a shell?** (adb, SSH, UART serial port)
- **Can you extract the kernel config?** (`zcat /proc/config.gz`)
- **Is there a device tree for your board?** (mainline `arch/arm*/boot/dts/`)
- **What does the partition table look like?** (`fdisk`, `by-partlabel`)
- **Is secure boot enabled?** (check bootloader logs, FIT signatures)
- **Can you load unsigned kernel modules?** (`CONFIG_MODULE_SIG`)

<!-- end_slide -->

Mainline vs downstream kernels
===

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

**Mainline** (Linus's tree)

- `git.kernel.org/torvalds/linux`
- Actively maintained, reviewed
- Security patches within days
- Device support added upstream
- DTB in `arch/arm*/boot/dts/`
- You build your own config

<!-- column: 1 -->

**Downstream** (vendor fork)

- Vendor's private fork, often Android
- Based on an old LTS (4.x, 3.x)
- Thousands of out-of-tree patches
- May never get security updates
- DTB baked into boot image
- You use what you're given

<!-- reset_layout -->



Most SoC vendors start downstream and **some** patches make it
upstream over time. The gap can be years -- or permanent.



Community forks (PostmarketOS, Mobian) sit in between: they take
a mainline kernel and add device-specific patches not yet upstream.
Close to mainline, but not truly mainline.



Nerves requires **kernel headers >= 5.4** to compile the BEAM and
its dependencies. Building against a 4.9 vendor kernel *is* possible
but expect to patch packages that assume modern kernel APIs.

<!-- end_slide -->

Finding a kernel for your device
===

You have three scenarios, from easy to hard:

```
  +--------------------------------------------------------------+
  |                                                              |
  |  Mainline       Community fork       Vendor Android kernel   |
  |  support        (PostmarketOS)       (last resort)           |
  |                                                              |
  |  * Upstream     * Dedicated fork     * Old (4.x, 3.x)        |
  |  * Well tested  * HW-specific        * Full of patches       |
  |  * DTB included * Usually works      * DTB baked into boot   |
  |                                      * image (FIT, Android)  |
  |  Nexus 7        Fairphone 2          Kobo Clara              |
  |  (Tegra 3)      (msm8974-mainline)   (MT8113, 4.9.77)        |
  |                                                              |
  +--------------------------------------------------------------+
```



With mainline or community forks, you get a **separate DTB** you can
pass to the bootloader. With vendor kernels, the device tree is often
**embedded in a signed boot image** -- you're stuck with it as-is.

When stuck with a vendor kernel, check:

```
CONFIG_MODULES=y              <-- can you load modules?
# CONFIG_MODULE_SIG is not set  <-- are unsigned modules accepted?
```

If yes, you can extend the kernel with F2FS, squashfs, USB gadget
modules -- everything Nerves needs -- without replacing the kernel.

<!-- end_slide -->

The Fairphone 2 story
===

**Device**: Snapdragon 801, 4 cores @ 2.26GHz, 2GB RAM, LTE modem

**Kernel**: a mainline fork exists (`msm8974-mainline/linux`)
with a proper device tree for the FP2. PostmarketOS runs on it.

**Bootloader**: the stock Android bootloader can launch **lk2nd**,
a second-level bootloader that scans for `extlinux.conf`:

```
  Stock bootloader
       |
       v
  +----------+
  |  lk2nd   |  Flashed on the "boot" partition
  +----------+  Scans partitions for extlinux.conf
       |
       v
  Our kernel + DTB + rootfs
```

**Flashing**: build the image, flash via fastboot:

```bash
mix firmware.image
fastboot flash userdata nerves_system_fairphone2.img
```

After the first flash, regular `mix upload` over USB works.

<!-- end_slide -->

The Fairphone 2: partition layout
===

Android's partition table is fixed, so Nerves is flashed
as a **sub-image** inside the `userdata` partition:

```
  Android eMMC layout
  +------------------------------------+
  | boot      <-- lk2nd                |
  +------------------------------------+
  | system    (unused by Nerves)       |
  +------------------------------------+
  | userdata  (20+ GB)                 |
  |  +-------------------------------+ |
  |  | MBR                           | |
  |  +-------------------------------+ |
  |  | Firmware config (uboot env)   | |
  |  +-------------------------------+ |
  |  | Boot A        | Boot B        | |
  |  +-------------------------------+ |
  |  | Rootfs A      | Rootfs B      | |
  |  | (squashfs)    | (squashfs)    | |
  |  +-------------------------------+ |
  |  | Application data (f2fs)       | |
  |  +-------------------------------+ |
  +------------------------------------+
```

An **initramfs** uses `kpartx` to map these sub-partitions
to real device files, then mounts them for Nerves.

<!-- end_slide -->

The Kobo Clara story: reconnaissance
===

**Device**: Kobo Clara Colour, MediaTek MT8113, e-ink display

Step 1: find a way in. The Kobo exposes a USB mass storage partition
with a file called `ssh-disabled`:

```
To enable ssh:
- Rename this file to ssh-enabled
- Reboot the device
- Connect via: ssh root@<device_ip>
```

Step 2: gather information once inside:

```bash
[root@kobo ~]# cat /proc/cpuinfo | grep Hardware
Hardware : MediaTek MT8110 board

[root@kobo ~]# cat /proc/cmdline
console=ttyS0,921600n1 root=/dev/mmcblk0p10 ...

[root@kobo ~]# zcat /proc/config.gz > /mnt/onboard/kernel.defconfig
```

Step 3: discover the bad news -- the FIT image is **signed**:

```
Sign algo:    sha256,rsa2048:dev
```

We cannot replace the bootloader, the kernel, or the device tree --
they're all locked inside the signed FIT image.

<!-- end_slide -->

The Kobo Clara: working around secure boot
===

Secure boot means the kernel and bootloader are untouchable.
But two things save us:

**1. Unsigned kernel modules are accepted**

```
CONFIG_MODULES=y
# CONFIG_MODULE_SIG is not set
```

We can cross-compile F2FS, squashfs, USB gadget modules
with the vendor toolchain and `insmod` them at boot.

**2. We control the rootfs init script**

The existing Kobo OS on `system_a` runs `/sbin/init` (BusyBox init).
We can modify it to detect a touch event at boot and
**pivot_root** to an alternative system stored on `userdata`.

```
  Kobo boots normally
       |
  Modified /sbin/init
       |
       +-- Touch detected? --yes--> Mount image, pivot_root
       |                                                |
       +-- No touch -------> Boot Kobo OS               v
                                                    Run Nerves
```

<!-- end_slide -->

The Kobo Clara: the pivot_root trick
===

Our Nerves image lives as a regular file on the Kobo's `userdata` partition:

```
  /mnt/onboard/.nerves/nerves-kobo.img
```

The modified init script does this:

```
  1. Load kernel modules (squashfs, f2fs)
              |
              v
  2. losetup -Pf image.img         <-- create /dev/loop0
              |
              v
  3. mount /dev/loop0p1 /newroot   <-- mount squashfs rootfs
              |
              v
  4. pivot_root . .oldroot         <-- swap root filesystems
              |
              v
  5. mount --move /dev, /proc, /sys
              |
              v
  6. umount .oldroot
              |
              v
  7. exec /sbin/init               <-- erlinit takes over as PID 1
```

The Kobo can still be used as a normal e-reader -- just don't
touch the screen during boot and the original OS starts.

<!-- end_slide -->

The Kobo Clara: the toolchain problem
===

The toolchain matters in **two places**:



**1. Buildroot** uses it to cross-compile the kernel modules
(squashfs, f2fs, USB gadget) that extend the on-device vendor kernel.
Modules must be built with the **exact same toolchain** as the kernel.



**2. Elixir NIFs** -- any Hex dependency with C code (like `fbink_nif`)
is compiled on your dev machine during `mix firmware`. It needs a
matching cross-compiler. In Nerves, this is a **toolchain package**:
an Elixir library that wraps a GCC sysroot.



Nerves provides toolchain packages for common architectures
(`nerves_toolchain_aarch64_nerves_linux_gnu`, etc.) but they
ship modern GCC. For the Kobo's 4.9 vendor kernel, we had to
**create a custom toolchain package** wrapping GCC 4.9.

<!-- end_slide -->

The Kobo Clara: GCC 4.9 in practice
===

The Kobo's vendor toolchain (GCC 4.9) breaks modern code:



**C99 errors**: Buildroot packages assume modern C standards

```
error: 'for' loop initial declarations are only allowed in C99 mode
```

Fix: override package makefiles to add `-std=gnu99`.



**Missing headers**: no `sys/random.h` with `getrandom()`

Fix: write a stub that returns `-ENOSYS` (erlinit falls back to `/dev/random`).



**Compiler bugs**: GCC 4.9 with `-O3` generates invalid ARM instructions
for specific code paths in Cairo.

Fix: patch the problematic file to compile at `-O0`.



**Year 2038**: 32-bit time overflow in 2038, causes compilation errors.

Fix: `--disable-year2038` when configuring Erlang.



Every vendor toolchain will have its own set of surprises.

<!-- end_slide -->

Adapting Nerves for unusual setups
===

When you can't follow the standard layout, you need to adapt:

**No boot partition (Kobo):**

```
  Standard Nerves image            Kobo Nerves image
  +---------------------+          +---------------------+
  | MBR                 |          | MBR                 |
  +---------------------+          +---------------------+
  | Boot A  | Boot B    |          | Firmware env        |
  +---------------------+          +---------------------+
  | Rootfs A | Rootfs B |          | Rootfs A | Rootfs B |
  +---------------------+          +---------------------+
  | Application (f2fs)  |          | Application (f2fs)  |
  +---------------------+          +---------------------+
```

**fwup.conf** must be adapted for the actual device paths:

```
# Standard
define(NERVES_FW_DEVPATH, "/dev/mmcblk0")

# Kobo (loop device from image file)
define(NERVES_FW_DEVPATH, "/dev/loop0")
define(NERVES_FW_APPLICATION_PART0_DEVPATH, "/dev/loop0p2")
```

And **erlinit.config** must mount from the right places.

<!-- end_slide -->

Getting cellular to work (Fairphone 2)
===

The Fairphone 2 has an LTE modem -- but making it work in Nerves
means replacing Linux userspace services with Elixir code.

In buildroot, shell scripts and daemons handle the modem:

```
  S39sim-selector:
  1. Check SIM slot 1 and 2
  2. Find which has a card inserted
  3. Reconfigure modem via qmicli
  4. Set raw_ip mode

  rmtfs daemon:
  Provides remote filesystem access to the modem's
  remoteproc -- required for the modem to boot at all
```

In Nerves, all of this becomes Elixir:

```
  vintage_net_qmi (modified)
  + ExRmtfs (new library -- replaces the rmtfs daemon)
  + QMI library (modified for older modems)
```

The modem required forcing `raw_ip` via QMI messages
because the usual `/sys/class` interface didn't work
on older Qualcomm modems.

Every init script from buildroot becomes an Elixir library in Nerves.

<!-- end_slide -->

Replacing init scripts: dealing with proprietary binaries (Kobo)
===

The WiFi stack requires `wmt_loader` -- a proprietary binary.
It always exits 255 because it tries to set an Android system property
that doesn't exist on our Linux-only Nerves system:

```elixir
defp run_wmt_loader do
  case KmodLoader.System.safe_cmd("/usr/bin/wmt_loader", []) do
    {:ok, output, exit_code} ->
      if wmt_loader_success?(exit_code, output) do
        KmodLoader.System.wait_for_device("/dev/stpwmt", timeout())
      else
        {:error, {:wmt_loader_failed, exit_code}}
      end

    {:error, :enoent} ->
      # Binary exists but can't execute -- missing ELF dynamic linker
      {:error, {:exec_failed, :enoent}}
  end
end

# Exit code 255 is "normal" -- detect success from stdout instead
defp wmt_loader_success?(0, _output), do: true
defp wmt_loader_success?(_exit_code, output) do
  String.contains?(output, "do kernel module init succeed: 0") or
    String.contains?(output, "Success to insmod")
end
```

You can't trust exit codes from vendor binaries.
Parse their output instead.

<!-- end_slide -->

Replacing init scripts: enabling WiFi via sysfs (Kobo)
===

WiFi is enabled by writing `"1"` to a device node, then
polling sysfs until the kernel creates the network interface:

```elixir
defp do_enable_wifi(wmt_wifi, ifname, attempts_left, retry_delay)
     when attempts_left > 0 do
  case File.write(wmt_wifi, "1") do
    :ok ->
      if wait_for_interface(ifname, retry_delay) do
        :ok
      else
        File.write(wmt_wifi, "0")    # Reset before retrying
        Process.sleep(500)
        do_enable_wifi(wmt_wifi, ifname, attempts_left - 1, retry_delay)
      end

    {:error, reason} ->
      Process.sleep(retry_delay)
      do_enable_wifi(wmt_wifi, ifname, attempts_left - 1, retry_delay)
  end
end
```

<!-- end_slide -->

Replacing init scripts: e-ink display init (Kobo)
===

The e-ink display needs device nodes created from sysfs at runtime.
This replaces a stock shell script that used `awk` and `mknod`:

```elixir
@regal_devices ~w(regal_wb regal_tmp regal_img regal_cinfo regal_waveform)

def run_regal_check do
  Enum.each(@regal_devices, fn dev_name ->
    ensure_regal_device(dev_name)
  end)
end

defp ensure_regal_device(dev_name) do
  sysfs_dev = Path.join(["/sys/class/regal_class", dev_name, "dev"])
  dev_path = Path.join("/dev", dev_name)

  with {:ok, content} <- File.read(sysfs_dev),
       [major_str, minor_str] <- content |> String.trim() |> String.split(":"),
       {major, ""} <- Integer.parse(major_str),
       {minor, ""} <- Integer.parse(minor_str) do
    System.cmd("mknod", [dev_path, "c", to_string(major), to_string(minor)])
  end
end
```

Device nodes in `/dev` are how userspace talks to kernel drivers.
Each one is identified by a **major:minor** number pair that the
kernel assigns dynamically. `mknod` creates the file, but without
the right numbers it points nowhere -- so we read them from sysfs first.

<!-- end_slide -->

The general strategy
===

```
  +-----------------------------------------------------+
  |  1. Reconnaissance                                  |
  |     Get a shell, extract kernel config, map         |
  |     partitions, check secure boot                   |
  +---------------------------+-------------------------+
                              |
  +---------------------------v-------------------------+
  |  2. Get Linux running                               |
  |     Find/build a kernel, get a device tree,         |
  |     understand the bootloader chain                 |
  +---------------------------+-------------------------+
                              |
  +---------------------------v-------------------------+
  |  3. Build a minimal Buildroot system                |
  |     Cross-compile a rootfs, get SSH access,         |
  |     prove the hardware works                        |
  +---------------------------+-------------------------+
                              |
  +---------------------------v-------------------------+
  |  4. Create the Nerves system                        |
  |     nerves_defconfig, fwup.conf, erlinit.config,    |
  |     adapt for device quirks                         |
  +---------------------------+-------------------------+
                              |
  +---------------------------v-------------------------+
  |  5. Replace init scripts with Elixir                |
  |     VintageNet, NervesSSH, custom libraries         |
  |     for device-specific hardware                    |
  +-----------------------------------------------------+
```

<!-- end_slide -->

Where to find existing Linux support
===

Before starting from scratch, check these resources:

**Mobile Linux distributions**

PostmarketOS, Mobian, Droidian -- huge databases of phone/tablet
hardware with kernel configs, device trees, and bootloader instructions.

**Board-focused distributions**

Armbian, Buildroot defconfigs, OpenWrt -- great for SBCs and routers.
If your SoC is supported, you already have a working kernel config.

**Vendor source releases**

Some vendors publish kernel + bootloader source (Kobo does this).
Even if it's an old Android kernel, it's a starting point.

**Mainline kernel**

Check `arch/arm/boot/dts/` and `arch/arm64/boot/dts/`
in the Linux source tree. Your SoC might already be supported.

**SoC-specific community forks**

msm8974-mainline, sdm845-mainline, linux-sunxi, etc.
Often exist for popular SoC families.

<!-- end_slide -->

Devices that already run Nerves
===

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

**Official Nerves systems**

- Raspberry Pi (0, 1, 2, 3, 4, 5)
- BeagleBone (Black, Green)
- Generic x86_64
- MangoPi MQ-Pro (RISC-V)
- OSD32MP1 (STM32MP1)
- Grisp2

<!-- column: 1 -->

**Community ports**

- Fairphone 2 (Snapdragon 801)
- Google Nexus 7 2012 (Tegra 3)
- Kobo Clara Colour (MT8113)
- Spotify Car Thing
- ...and many more

Search **nerves_system** on GitHub for the full list.

<!-- reset_layout -->

The PostmarketOS wiki lists hundreds of devices that boot Linux.
Each one is a potential Nerves target.

**If it runs Linux, it can probably run Nerves.**

<!-- end_slide -->

What's next
===

**More exotic ports in progress:**

- An Android smartwatch
- Fairphone 3
- A gaming console (TBD)

**From PostmarketOS to Nerves in two commands:**

I'm working on a tool that converts any PostmarketOS device
configuration into a Buildroot external tree automatically.

```
  PostmarketOS device config
         |
         v
  pmos-to-buildroot pine64     --> BR2_EXTERNAL tree
         |
         v
  mix nerves.system.bootstrap  --> Complete Nerves system
```

PostmarketOS supports **hundreds** of devices. Combined with
`nerves_system_bootstrap`, this could turn any of them into
a Nerves target with minimal manual work.

<!-- end_slide -->

<!-- jump_to_middle -->

Thank you!
===

Questions?

[My github](https://www.github.com/mlainez)

[My company's github](https://www.github.com/Spin42)

[My Elixirforum profile](https://www.elixirforum.com/u/mlainez)

**The articles:**

[Running Nerves on Android E-Waster posts](https://elixirforum.com/t/running-nerves-on-android-e-waste/70187)

[The Journey to running Nerves on a Kobo Clara Colour ereader](https://elixirforum.com/t/the-journey-to-running-nerves-on-a-kobo-clara-e-reader/73331)
