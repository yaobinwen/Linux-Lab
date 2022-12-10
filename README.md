# Linux Lab

## Overview

This repository shows how to create a home lab to study Linux kernal and C/C++ libraries (and possibly everything else).

The home lab requires two machines:
- A physical machine with Ubuntu 18.04 (or later) installed. You need to prepare this machine by yourself. The physical machine is used to create the virtual machine that is used to build source code. See the section "Physical Machine" below for detailed setup steps.
- A virtual machine that builds the code of Linux kernel and everything else. This README file and the Ansible playbooks that are provided by this repository will help you set it up. See the section "Virtual Machine" below for detailed setup steps.

## Physical Machine

Follow the steps below to set up the physical machine:
- Install Ubuntu 18.04 (or later).
- Check out the repository [yaobinwen/work-env](https://github.com/yaobinwen/work-env).
- Run [`bootstrap.sh`](https://github.com/yaobinwen/work-env/blob/master/bootstrap.sh) to do bootstrap setup (mainly install Ansible).
- Under the directory `work-env/ansible`:
  - Run the playbook `virtualbox.yml` to install [VirtualBox](https://www.virtualbox.org/).
  - Run the playbook `vagrant.yml` to install [Vagrant](https://www.vagrantup.com/).
- Run `ansible/linux-lab.yml` to set up the Linux Lab work directory.
  - Note: If you want to build a specific version of Linux kernel, it is better to choose a Vagrant box that uses the same version to make it easier to find all the needed dependencies. For example, if you want to build Ubuntu Jammy's kernel, it's better to choose the Vagrant box `ubuntu/jammy64`. This may avoid the issue of VirtualBox shared folder.
- `cd` into the Linux Lab work directory.
  - This directory is mapped into the virtual machine at `/lab`.
- Refer to the section "Source Code" to check out the code of the Linux kernels or C/C++ libraries you want to build.
- Run `vagrant up` to start up the Linux Lab virtual machine.
- Run `vagrant ssh` to log into the Linux Lab virtual machine.

Now you can move on to the section "Virtual Machine".

## Virtual Machine (VM)

`/lab` is a shared folder between the physical machine and the virtual machine. Follow the steps below to build the code:
- `cd /lab`.
- `cd` into the working copy directory and follow the instructions to build and install the code inside the VM.
- In case the VM is screwed up somehow, you can just destroy and recreate the VM. All the built artifacts are also saved in the working copies on the host so you don't have to rebuild things again.

## Source Code

- Linux:
  - Homepage: [The Linux Kernel Archives](https://www.kernel.org/)
  - My GitHub fork: [yaobinwen/linux](https://github.com/yaobinwen/linux/)
    - As of 2022-12-11, I'm using the branch [`ywen/v5.10`](https://github.com/yaobinwen/linux/tree/ywen/v5.10) on which I updated `README` to include the build instructions.
- Linux (Ubuntu):
  - Homepage: [kernel.ubuntu.com](https://kernel.ubuntu.com/)
  - [GitWeb](https://kernel.ubuntu.com/git/) lists all the git repositories that can be cloned.
    - Note:
  - [Build Your Own Kernel](https://wiki.ubuntu.com/Kernel/BuildYourOwnKernel) has the instructions to build the kernel.
  - My GitHub fork: [yaobinwen/ubuntu-kernel-jammy](https://github.com/yaobinwen/ubuntu-kernel-jammy)
- `libc`:
  - See [`libc(7)`](https://manpages.ubuntu.com/manpages/jammy/man7/libc.7.html) to get an overview of what `libc` actually refers to.
    - Key takeaway: As of 2022-12-11, `libc` mostly refers to GNU C Library (i.e., `glibc`).
  - `glibc`:
    - Homepage: [The GNU C Library (glibc)](https://www.gnu.org/software/libc/)
    - My GitHub mirror: [yaobinwen/ubuntu-glibc](https://github.com/yaobinwen/ubuntu-glibc)

## Switching Kernels

If you want to switch back to another kernel:
- Remove all the files under `/boot` for that kernel version.
- Adjust the symbolic links `/boot/initrd.img` and `/boot/vmlinuz` to point to the desired kernel version.
- Remove the kernel modules under `/lib/modules` for that kernel version.
- Run `sudo update-grub`.
- Check the `menuentry` under the section `submenu 'Advanced options for Ubuntu'` in `/boot/grub/grub.cfg` to confirm the removed kernel version is no longer there. Also confirm the first `menuentry` points to the desired kernel version.
  - This method assumes that we always want to use the first available kernel version. If there are multiple kernel versions and you don't want to use the first one, you can update `GRUB_DEFAULT=0` in `/etc/default/grub` to be the desired index, or update it to something like `GRUB_DEFAULT='gnulinux-5.4.0-135-generic-advanced-52183739-668c-4ae6-b3bd-900912adcff5'` (the value can be found in the last column of `menuentry`). See the help info for `GRUB_DEFAULT`.
- Reboot the machine.

## Issues

### VirtualBox shared folders (Unresolved yet)

After installing a custom built Linux kernel, the shared folders may not be mounted correctly and `vagrant reload` may print the following error:

```
VirtualBox: mount.vboxsf: mounting failed with the error: No such device
```

This should be because the newly installed kernel may not have the VirtualBox modules added as suggested by [this answer](https://stackoverflow.com/a/30388070/630364):

```
modprobe -a vboxguest vboxsf vboxvideo
```

However, simply running the command above may not solve the problem because the modules `vboxguest` and `vboxsf` may not exist for the custom built kernel yet. If you look into `/lib/modules`, you'll see the old kernel modules (`5.4.0-135-generic` in my case):

```
vagrant@ywen-linux-lab:/lib/modules$ ll
total 16
drwxr-xr-x  4 root root 4096 Dec 11 18:51 ./
drwxr-xr-x 91 root root 4096 Dec 11 17:50 ../
drwxr-xr-x  3 root root 4096 Dec 11 18:51 5.10.0+/
drwxr-xr-x  5 root root 4096 Dec  2 13:14 5.4.0-135-generic/
```

And you can find the VirtualBox-related modules in the old kernel module directory:

```
vagrant@ywen-linux-lab:/lib/modules$ find ./5.4.0-135-generic/ -name "vb*" -type f
./5.4.0-135-generic/kernel/drivers/gpu/drm/vboxvideo/vboxvideo.ko
./5.4.0-135-generic/kernel/virtualbox-guest/vboxsf.ko
./5.4.0-135-generic/kernel/virtualbox-guest/vboxguest.ko
```

The module `vboxvideo` can also be found under the custom built kernel `5.10.0+`, but the other two can't be found there.

One thing you think you can try (but will fail) is to create a symbolic link inside `/lib/modules/5.10.0+` to point to the kernel modules that were built for the older kernel:

```
virtualbox-guest -> /lib/modules/5.4.0-135-generic/kernel/virtualbox-guest/
```

But this won't work and you will probably get the error of `Exec format error`:

```
vagrant@ywen-linux-lab:/lib/modules/5.10.0+/kernel$ sudo modprobe -a vboxguest vboxsf vboxvideo
modprobe: ERROR: could not insert 'vboxguest': Exec format error
modprobe: ERROR: could not insert 'vboxsf': Exec format error
```

I tried to mount [`VBoxGuestAdditions_7.0.4.iso`](http://download.virtualbox.org/virtualbox/7.0.4/VBoxGuestAdditions_7.0.4.iso) inside the VM to install the guest addition but I got the following errors:

```
vagrant@ywen-linux-lab:/media/vbox-guest-addition$ sudo ./VBoxLinuxAdditions.run
Verifying archive integrity...  100%   MD5 checksums are OK. All good.
Uncompressing VirtualBox 7.0.4 Guest Additions for Linux  100%
VirtualBox Guest Additions installer
VBoxControl: error: Could not contact the host system.  Make sure that you are running this
VBoxControl: error: application inside a VirtualBox guest system, and that you have sufficient
VBoxControl: error: user permissions.
This system appears to have a version of the VirtualBox Guest Additions
already installed.  If it is part of the operating system and kept up-to-date,
there is most likely no need to replace it.  If it is not up-to-date, you
should get a notification when you start the system.  If you wish to replace
it with this version, please do not continue with this installation now, but
instead remove the current version first, following the instructions for the
operating system.

If your system simply has the remains of a version of the Additions you could
not remove you should probably continue now, and these will be removed during
installation.

Do you wish to continue? [yes or no]
yes
touch: cannot touch '/var/lib/VBoxGuestAdditions/skip-5.4.0-135-generic': No such file or directory
/opt/VBoxGuestAdditions-7.0.4/bin/VBoxClient: error while loading shared libraries: libXt.so.6: cannot open shared object file: No such file or directory
/opt/VBoxGuestAdditions-7.0.4/bin/VBoxClient: error while loading shared libraries: libXt.so.6: cannot open shared object file: No such file or directory
VirtualBox Guest Additions: Starting.
VirtualBox Guest Additions: Setting up modules
VirtualBox Guest Additions: Building the VirtualBox Guest Additions kernel
modules.  This may take a while.
VirtualBox Guest Additions: To build modules for other installed kernels, run
VirtualBox Guest Additions:   /sbin/rcvboxadd quicksetup <version>
VirtualBox Guest Additions: or
VirtualBox Guest Additions:   /sbin/rcvboxadd quicksetup all
VirtualBox Guest Additions: Kernel headers not found for target kernel 5.10.0+.
Please install them and execute
  /sbin/rcvboxadd setup
modprobe vboxguest failed
The log file /var/log/vboxadd-setup.log may contain further information.
vagrant@ywen-linux-lab:/media/vbox-guest-addition$
```

There are a few errors I need to understand and solve:
- 1). Why did it report `Could not contact the host system`?
- 2). What caused `cannot touch '/var/lib/VBoxGuestAdditions/skip-5.4.0-135-generic': No such file or directory`?
- 3). How to get `libXt.so.6`?
- 4). How to solve the error `Kernel headers not found for target kernel 5.10.0+`? (It looks like I need to run some `make install` to install the kernel header files.)
