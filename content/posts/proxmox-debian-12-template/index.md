---
title: "Create Virtual Machines in Seconds"
description: "Simplify VM Creation with Proxmox and Debian 12 Templates"
date: 2023-07-06T20:45:45+03:00
draft: false
---

I was tired of going through the repetitive and time-consuming process of installing a new virtual machine from scratch every time I needed to test something in an isolated environment.
Sure, containers are an easy way to provide this isolation but there were some cases I wanted to have a VM in minutes.

In this post, we'll walk through the simple steps to create a reusable template using the latest Debian version (12 - Bookworm) and the Proxmox hypervisor.

By leveraging the magic of cloud-init and Proxmox templates, you can effortlessly clone and spin up virtual machines in a matter of seconds.

First of all, you will need to find the download link of the *qcow2* cloud image for your specific architecture from the official [Debian website](https://www.debian.org/distrib/). For the *amd64* architecture the download link is [this](https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.qcow2).

Download the cloud image in your Proxmox hypervisor inside any folder (we will later import it in a VM, so it will be copied automatically to the correct destination).

After this, create from the Proxmox WebUI a new VM. I prefer to give the ID 999 to my most used template (to be honest I only use one). I named the VM **debian-12-template**. Press next and on the OS tab select *Do not use any media*. Make sure that the Guest OS type is *Linux* and the Version is *6.x - 2.6 Kernel*.

On the *System* tab, the defaults should be fine. My defaults with Proxmox 8.0.3 were
- `i440fx` for Machine
- `SeaBIOS` for BIOS
- `VirtIO SCSI single` for SCSI Controller

On *Disks*, delete the prexisting drive and leave the system without any drive. We will insert our own later :)

After selecting your *CPU*, *Memory* and *Network* default settings (you will be able to change this per cloned VM, fill this with the most common settings just to save some clicks after cloning).

For the next step, you will need to know the name Storage names used in your Proxmox instance. If you don't know what I am talking about, you should use `local`, otherwise choose a storage that has `Disk Image` content.

Then, run

```bash
qm importdisk VM_ID debian-12-generic-amd64.qcow2 STORAGE_NAME
```

This will import the *qcow2* disk to the VM with the given ID and store it at the selected storage. It should take a couple of seconds.

After the disk is imported, it will be unused. You will need the *Hardware* page of the VM, double click on the unused disk and press Add.

Then, you will need to add a *cloud-init drive* by pressing Add on the same (*Hardware*) page. Select the storage you want (again local), leave the *Bus/Device* to `IDE` and press Add.

With this step done, go to the *Cloud-init* panel from the side and configure the default values you want.

Then, you should go to the *Options* panel, double click on the *Boot order* line, enable the scsi0 drive (the Debian drive) and move it on top of the boot order.

When you are ready, press on the More button on the top right and select the `Convert to template option`

Now, when you want to create a VM, you can easily clone this template and have a ready-to-go Debian 12 instance. You can configure and change the CPU cores and allocated memory per cloned instance. The thing that you shouldn't forget to do after cloning is to enlarge the Hard Disk of the cloned VM by navigating to the *Hardware* page, select the `scsi0` drive and from the top menu *Disk Actions* -> *Resize*. This will increase the disk size by the specified value.

In conclusion, by leveraging the power of templates and cloud-init, you can clone and deploy virtual machines in a matter of seconds.
