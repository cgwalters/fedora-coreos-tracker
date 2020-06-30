# The initramfs

For CoreOS the initramfs is critical; a key technological pillar of CoreOS is [Ignition](https://github.com/coreos/ignition/) which e.g. handles partitioning disks that happen on the first boot.  One way to think about this is that Ignition handles a lot of the roles that a traditional "installer" program might - our initramfs contains `sgdisk`, most others don't.

# Initramfs history

See the upstream Linux kernel document: ["what is initramfs"](https://www.kernel.org/doc/html/latest/filesystems/ramfs-rootfs-initramfs.html?highlight=initramfs#what-is-initramfs).

It's basically a small filesystem that gets passed to the kernel by the bootloader, and the kernel unpacks and runs it.

The high level goal of the initramfs is to mount the root filesystem (conventionally at `/sysroot`) and switch root into it, i.e. turning `/sysroot` into `/`.

# Initramfs technologies

We use [dracut](https://github.com/dracutdevs/dracut/) the same as a number of other (but not all) distributions.  It basically gathers binaries/configuration from the real root and generates an initramfs from them.

Modern systemd has a very clean design for both the initramfs and the real boot. See the ["man bootup"](https://www.freedesktop.org/software/systemd/man/bootup.html) documentation.  The software involved implements these abstract `.target` units.

There are 3 important pieces of software involved in the initramfs:

- [ignition-dracut](https://github.com/coreos/ignition-dracut/) (i.e. Ignition)
- [ostree-prepare-root](https://github.com/ostreedev/ostree/blob/master/src/switchroot/ostree-prepare-root.c) (Part of OSTree)
- [40ignition-ostree dracut module](https://github.com/coreos/fedora-coreos-config/tree/testing-devel/overlay.d/05core/usr/lib/dracut/modules.d/40ignition-ostree) (fedora-coreos-config)

Note that Ignition and OSTree are both independent projects consumed by other distributions in addition to Fedora CoreOS.  This means that we want to support using each independently.  The `40ignition-ostree` dracut module *ties those two together* - it's the place where you will find systemd units that have direct ordering relationship around the two projects.

# First boot versus subsequent boot

Ignition runs only on the first boot.  To account for this, ignition-dracut ships two targets:

`ignition-complete.target`: Enabled on first boot
`ignition-subsequent.target`: Enabled on every boot **except** the first

`-complete` will pull in a lot of units, such as `ignition-fetch.service` and `ignition-disks.service`

We implement`ignition-subsequent.target` today by hooking in `ignition-ostree-mount-subsequent-sysroot.service` which basically just waits for a filesystem with `LABEL=root` and mounts it - very simple!  But see below around the root filesystem.

# Images and finding filesystems

An important part of the CoreOS philosophy is to make bare metal as close to cloud workflows as possible.   This follows from moving all support for e.g. filesystem provisioning into Ignition.

[coreos-assembler](https://github.com/coreos/coreos-assembler) generates a disk image with `boot` (`/boot`) and `root` (`/`) labels.  Various components of the initramfs (as well as our default GRUB config) use the `label=boot` to find the boot partition.  The label `root` is used by `ignition-ostree-mount-firstboot-sysroot.service`.

# Live versus diskful

We also ship a "Live" ISO/PXE image which uses a different filesystem (squashfs).  This caused us to introduce a separate `ignition-diskful.target` which only runs on cases where we're booted from writable persistent storage (i.e. not ISO/PXE).

To implement the "live" or "run in RAM" aspects, the `live-generator` sets up an `overlayfs` for `/etc` and a `tmpfs` for `/var`.  Everything else is part of the `squashfs` which is read-only.

The Live OS setup differs currently between the ISO and PXE: https://github.com/coreos/fedora-coreos-tracker/issues/390

Currently when generating the ISO image we inject a label onto the root filesystem, and a `coreos.liveiso` kernel argument matching it.  The initramfs knows to look for that kernel argument, which it then uses to mount the squashfs which contains the root filesystem.

In contrast for PXE the squashfs is in the `live-initramfs` directly.

# SELinux in the initramfs

SELinux policy is loaded in the real root.  This means that every file we create in the initramfs must be relabeled.  See this code: https://github.com/coreos/fedora-coreos-config/blob/testing-devel/overlay.d/05core/usr/lib/dracut/modules.d/40ignition-ostree/coreos-relabel

# Reprovisioning the root

A big recent effort is [reprovisioning the root filesystem](https://github.com/coreos/fedora-coreos-tracker/issues/94).  This will make the "subsequent" boot path work differently based on configuration.

# Debugging the initramfs

- https://fedoraproject.org/wiki/How_to_debug_Dracut_problems is generally useful
- Use `cosa buildinitramfs-fast` for fast iteration: https://github.com/coreos/coreos-assembler/pull/1433
- ignition-dracut contains code to dump the journal to a virtio channel: https://github.com/coreos/ignition-dracut/pull/146  - This is used by parts of coreos-assembler
