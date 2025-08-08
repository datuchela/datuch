+++
date = '2025-08-07T15:27:09+04:00'
draft = false
title = 'Ephemeral VMs with tmpfs'
+++

## Motivation

I use VMs a lot. Most of the time, I just need to create one, use it, and then throw it away. Doing all of this on an SSD can shorten its lifespan, and repeatedly recreating disk images and installing the OS every time feels redundant.

## Getting Rid of Redundancy

To avoid doing the same setup work repeatedly, the obvious solution is to have a single base image like `win11-base.qcow2`, and clone it every time you need a new VM.

## tmpfs

But this doesn’t solve the SSD wear problem... That’s where `tmpfs` comes in.  
`tmpfs` is a **T**e**mp**orary **F**ile **S**ystem supported by most Unix-like operating systems. It allows you to mount a filesystem directly in RAM. That alone is powerful and has many use cases — but let’s stay focused on our original goal: eliminating unnecessary SSD writes and deletes.

We can mount a RAM-based filesystem, copy the base image (`win11-base.qcow2`) to it, and use it to run the VM:

```bash
sudo mkdir -p /mnt/ramdisk
sudo mount -t tmpfs -o size=32G tmpfs /mnt/ramdisk
sudo cp /path/to/base-image.qcow2 /mnt/ramdisk/base-image-clone.qcow2
```
If you test disk speeds inside the VM, you’ll be amazed — seriously.

## Can't download more RAM

At this point, we’ve solved our original problem. No more unnecessary I/O operations on the SSD — everything runs from RAM and disappears on reboot or when deleted manually from tmpfs.

But there’s an obvious limitation:
Not everyone has enough RAM to allocate ~20GB just for a disk image and run a VM on top of that.

## QCOW2 overlays

Thankfully, the QEMU team has already thought about this.
They’ve implemented overlay disk images.

An overlay disk image is created from a base image (called a backing file) and only stores changes (deltas) made after creation. This eliminates the need to store the full 20+ GB image in RAM just to run a disposable VM.

## Final solution

Here’s the complete solution to the problem:

**1. Create and mount a RAM filesystem using `tmpfs`:**

```bash
sudo mkdir -p /mnt/ramdisk
sudo mount -t tmpfs -o size=16G tmpfs /mnt/ramdisk
```
**2. Create an overlay image from the base image using `qemu-img`:**

```bash
qemu-img create -f qcow2 -b "/path/to/base-image.qcow2" -F qcow2 "/mnt/ramdisk/base-image-overlay.qcow2"
```
**3. Run the VM using the overlay image as its disk.**

**4. (Optional) Unmount the RAM disk afterwards:**

```bash
sudo umount /mnt/ramdisk
```

## ramvm

To simplify this entire workflow, I created a small script called `ramvm`.  
It automates the process of creating a RAM-backed overlay and running a disposable VM with minimal effort.

You can check out the source code here: <a target="_blank" href="https://codeberg.org/datuch/ramvm">https://codeberg.org/datuch/ramvm</a>

