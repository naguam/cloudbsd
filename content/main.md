+++
title = "From Linux to NetBSD, with SSH only"
date = "2024-07-12"
[taxonomies]
tags = ["Cloud","BSD","NetBSD","Linux","Debian","QEMU","SSH","Operating System","FR","EN"]
+++
# Swap the operating system of any remote Linux server by the one of your choice

*[Version française](@/main.fr.md)*

## Demo video

This video shows the installation of NetBSD in place of an existing Debian Linux.

<script src="https://asciinema.org/a/5rgoGoufRUXujCMvUiduYtJPr.js" id="asciicast-5rgoGoufRUXujCMvUiduYtJPr" async="true"></script>

## Disclaimer

This article is provided **as is**, **without any warranties or guarantees whatsoever**.

The author **cannot** be held liable for any damage that may occur if anything goes wrong. Only perform these actions on infrastructure you own or have permission to modify this way and ensure you understand all the implications.

If a provider have policies that forbid doing what's described here, just don't do it.

Also don't do this on providers located in countries that are under US Export Administration Regulations embargo and operating systems that are under these regulations especially regarding cryptography.

## Acknowledgements

Thanks to Marcan, prominent project lead of [Asahi Linux](https://asahilinux.org/) and member of [Fail0verflow](https://fail0verflow.com/blog/) whose repository's [takeover.sh](https://github.com/marcan/takeover.sh) refreshed my memory on some Linux mechanisms and sparked my curiosity to better understand how initrd and pivot_root work.

This article is using his work, on which I added emulation and some scripting to go beyond Linux's boundaries.

Thanks to the NetBSD people for continuing the work on [NetBSD](https://www.netbsd.org/).

Thanks to the active community on [UnitedBSD](https://www.unitedbsd.com/) for providing a "modern" user friendly place to get help and talk around the BSD operating systems.

## TL;DR

- Clone [Marcan's takeover.sh](https://github.com/marcan/takeover.sh) repository.
- **Read** the README.md file and understand every single word.
- Prepare the components:
    - A statically built version of busybox.
    - Compile the `fakeinit.c` statically.
    - The ISO (lightweight netboot prefered) of the system.
    ([NetBSD preconfigured to boot on serial output](https://cdn.netbsd.org/pub/NetBSD/NetBSD-10.0/amd64/installation/cdrom/boot-com.iso)).
    - Check out the scripts in this article.
- [Watch the video](#demo-video).

## Introduction

For a long time, I have been curious whether there were ways to swap out the operating systems of a remote server without having access to a BIOS/UEFI nor to a good cloud control panel with ISO/images loading capabilities, but only throughout a remote SSH access to an existing system.

I have been in various situations where I think this would have been useful.

I have setup Linux based NAS servers, for non-technical people, which I'm updating with apt or yum/dnf update every few weeks.
Uneluctably, after some time, the operating systems become unsupported and upgrading them may not be an option as this could lead to issues which can cause remote access problems, and sometimes just swapping the entire operating system for a specialised one would be preferable.

Another situation I see quite often, are very narrow and often outdated selection of operating systems on cheap cloud providers which don't allow custom images or ISO provisionning, and which consumer services might not be helpful. However, these providers are cheap, and this is why people come to them, they still provide useful service.

As a hobbyist not in a position to build a homelab, cheap cloud providers are a solution for tinkering with operating systems including BSD or for building a small multinode Kubernetes cluster. Indeed, if on the short term, people can do local virtual machines, this is not enough to replace a homelab, in terms of availability and because of the volatility of mobile devices' network.

I've had multiple ideas for that matter in the past:
- With one partition available, one can just format it and install an operating system (the Arch, Gentoo, Debootstrap way), configure the network, the remote access, the bootloader, reboot, format the first partition, transfer the files, update the fstab and bootloader, re-reboot, reformat the second partition and optionaly grow the first partition. The partition availability requirement of this option makes it rarely possible.
- Another option, is to use an operating system which has an unattended installation mechanism, preconfigure the network, and remote access settings to match the resulting environment, configure grub to chainload the ISO in RAM and proceed to the unattended installation (the installer needs to be able to reboot by itself at the end of the installation). Not sure how feasible this is in practice, this would require a lot of trials and errors, because this methods is completely headless with almost no chances of recovery if things don't go as intended.
- The method described in this article which is quite unusual but interesting. One limitation however is that it requires the remote's server existing operating system to be Linux based.

In this article, my goal is to provide some understanding on the concepts Marcan used in his script and allow people (mainly hobbyists) to run the operating system of their choice on any remote servers.

## The major obstacle

Swapping an operating system in-place brings a major problem: a system in operation is using its mounted root partition.
For very obvious reasons, Linux forbids any attempt to unmount partitions in use by any process as this could corrupt the data and put the system in an inconsistent state.

The most common example of this is when a user opens a terminal and changes the current directory to one in a mounted partition, and then tries to unmount the volume.
The system prevents this saying the volume is busy requiring the user to move out of that mountpoint to be able to unmount the volume.

To swap an operating system in operation, we need to gain control of the main volume, to be able to format it and install our new operating system while retaining remote access. Thanks to various mechanisms, while quite non-intuitive, this is feasible.

For a better understanding, the next section reminds some basics that will help understand the procedure.

## Basics on initial ramdisks

Even in my technical circle, very few people seem to really understand why initrd and initramfs are often needed and how they work.
Not pretending to be an expert on initrd and initramfs, I will try to explain for anyone the base principles. I would encourage experts to contact me if any major issue is uncovered to improve the below.

When booting, Linux must load the init process (PID 1), loaded from an executable which can be stored on a different partition.
Let's show the issue storing the init on a different partition is causing across the following example.

A common setup for a Linux laptop is to use luks2 disk encyption for the root partition, the kernel being loaded from an unencrypted boot partition. 
If the module `dm_crypt` is on the encrypted partition, the kernel has no way to load the module to decrypt the latter and thus, the kernel can't load the init from the very same partition either.

This where an initrd or an initramfs come into play to provide all the required tools the kernel needs to access the root partition.

The way initramfs works is slightly different from the initrd and will not be covered here.
Indeed, the focus of this section is to outline specific initrd operations to help to understand the rest of the article.

Initrd is the contraction of "initial ramdisk" which is quite self-descriptive. Loaded in RAM, it exposes a small filesystem which contains the necessary modules and scripts (contains busybox on Debian systems) to be used by the kernel to access and load the root filesystem.
The ramdisk contains some form of init because Linux requires a first process to be executed. This ramdisk init will load the various scripts and load the various modules required to load the root filesystem.

Once the root partition is unlocked, the actual init must replace the initrd's one. The unlocked root should also become the "real" root, as technically, the initrd is the root filesystem before its swap with the "real" one.

This is where `pivot_root` comes into play.

First let's avoid any confusion : the main difference between `chroot` and `pivot_root` is that `chroot` only changes the executed process' vision making it believe that the root is "elsewhere". `pivot_chroot` on the other end will actually swap mounting points.

This is illustrated by the following example which mostly describes what the initrd's init script does:

```sh
# Load dm_crypt module and decrypt rootfs
# Mount the "real" root at /targetroot, the current / being the ramdisk
mount /dev/mapper/decrypted_rootfs /targetroot
# Prepare the future ramdisk mountpoint
mkdir /targetroot/old_root
# Swap /targetroot to /, and the old / to /old_root
# pivot_root new_root put_old
pivot_root /targetroot /targetroot/old_root
# / is the real decrypted / and /old_root is now the ramdisk
# Now the initrd init replaces itself by the real init
# (chroot because the "old" initrd init is not aware of the pivot_root)
exec chroot / /bin/init
# The new real init can now unmount /old_root releasing the initrd memory
```

Now we have a good a grasp on these concepts, what's next will be easier to understand.

## The procedure

This replacement relies on Linux as mentionned in the introduction and will be illustrated by a Debian 12 virtual machine.

The target operating system can be any system that allows serial, SSH, or VNC installation.
Beware that the procedure requires public access to at least two TCP ports for SSH.

For the purpose of this article and because this might interest the BSD hobbyists, the new system will be a NetBSD 10.

Marcan's takeover script contains ouputs which makes its understanding quite straight forward.
Therefore, only the interesting sections of his script and the reason each steps are combined together will be outlined.

The procedure will be divided in three subsections.
The [video](#demo-video) helps the understanding.

### Prepare the system

To avoid using the root volume, we can use `pivot_root` to swap the existing `/` to an in-memory filesystem which we will call ramdisk from now on.
If all programs including the init are loaded from the ramdisk and no process are using the root volume anymore, we will be able to unmount it.
From there we can manipulate the root volume which includes installing a new operating system.

To achieve this, any tool used to install the future operating system must be present on the ramdisk.
For this reason this method requires quite some RAM. In the current demonstration it is possible to make it work on a virtual machine with only two gigabytes of RAM, however this takes into account that the NetBSD network ISO is only about 300MB and its installer can be run in a memory constrained (128MB) setting.

Indeed there must be enough memory to fit a ramdisk big enough to contain everything we need which includes, a base Linux system that can be chrooted, a SSH server, a QEMU emulator, efibootmgr in case of an UEFI system, the NetBSD network installation ISO. Also consider some memory margin to be able to run the NetBSD installer.

However if the future operating system is Ubuntu instead of NetBSD, Ubuntu's ISOs tend to be around 2GB for the server edition, so either consider a network installation (or provide the ISO throughout NFS) or target a server with large enough memory to fit such files in the ramdisk.

A good starting point is to actually prepare all the files and scripts on a local computer and only transfer them to the system to be swapped once everything is ready.

First clone [Marcan's takeover.sh](https://github.com/marcan/takeover.sh) repository and compile the `fakeinit.c` file.
This will be the new init once the pivot will be done.
My personal advice is to compile fakeinit statically if the libraries (libc) on the ramdisk do not correspond to the one used at compile time.

Here is the short fakeinit content

```c
#define _XOPEN_SOURCE 700
#include <signal.h>
#include <unistd.h>
#include <sys/wait.h>

int main()
{
	sigset_t set;
	int status, i;

	for (i = 0; i < 1024; i++)
		close(i);

	if (getpid() != 1) return 1;

	sigfillset(&set);
	sigprocmask(SIG_BLOCK, &set, 0);

	for (;;) wait(&status);
}
```

As we can see, it is minimalistic, not even providing a shutdown method.
Its only role is to close the file descriptors that might have been used by the previous init, again to avoid the rootfs to be considered as in use.
The rest is quite classic for an init, exiting when it is not PID 1, otherwise intercepting any signals, and wait endlessly.

Then also fetch a statically built busybox which contains the version of `chroot` and `pivot_root` we will use.
The static compilation is important because we need to ensure `pivot_root` and `chroot` which will be run from the ramdisk, wont try to load shared libraries from the root filesystem.
The rest of the binaries on the ramdisk wont have the problem as they will be run from inside a chroot.

A SSH server must be in the ramdisk because the existing SSH server will have to be stopped as it is using the root partition. To avoid losing control, a SSH server will be run from the ramdisk in the chroot on a different port first.

We need QEMU (and Seabios or EDK2 and efibootmgr depending on the host booting mode) installed in the ramdisk, otherwise we wont be able to install any operating system, but Linux ones.

The best place to start is from a base installation of a (minimal) Linux distribution, which already contains a SSH server, then install QEMU and all the relevant tools into it, and then rsync the root partition's content on the ramdisk.

To sum up, here is commented version of the `setup.sh` script I wrote and used in the demo video to prepare the swap.

```sh
#!/usr/bin/env bash

# Not in the script but present in the video. In case of Debian, run
# apt install --no-install-recommends qemu-system-x86 qemu-utils seabios rsync
# The --no-install-recommends helps to reduce the amount of storage used by these.
# Don't forget to install efibootmgr and edk2 in case of an UEFI system.

# Create mountpoint for the ramdisk
mkdir /takeover

# Create and mount the tmpfs in-memory filesystem
mount -t tmpfs -o size=1400M tmpfs /takeover
# 1400M was just enough to fit it all while keeping some megabytes
# available in memory for the new system's installation

# Sync the rootfs (containing SSH QEMU and Seabios)
# Ignoring the heavy non-required files and ignore virtual.special filesystems
rsync -avX \
        --exclude /media/* \
        --exclude /dev \
        --exclude /sys \
        --exclude /proc \
        --exclude /boot \
        --exclude /lost+found \
        --exclude /tmp \
        --exclude /vmlinuz \
        --exclude /vmlinuz.old \
        --exclude /initrd.img \
        --exclude /initrd.img.old \
        --exclude /var/log/journal \
        --exclude /var/lib/systemd \
        --exclude /var/tmp \
        --exclude /usr/share \
        --exclude /takeover \
        --exclude /swapfile \
        --exclude /root/takeover.sh \
        / /takeover

# As we ignored /usr/share for space, lets put back necessary files for QEMU
rsync -avX /usr/share/qemu /takeover/usr/share
rsync -avX /usr/share/seabios /takeover/usr/share

# Recreate mountpoints for future needed special file systems mounts
# Indeed we will have to remount them in the ramdisk.
mkdir /takeover/boot
mkdir /takeover/tmp
mkdir /takeover/sys
mkdir /takeover/proc
mkdir /takeover/dev
mkdir /takeover/var/tmp

# Copy Marcan's takeover script, busybox static binary,
# fakeinit binary and the NetBSD com boot ISO.
cp busybox /takeover/
cp takeover.sh /takeover/
cp fakeinit /takeover/
cp boot-com.iso /takeover/

# The NetBSD com ISO allows to directly have serial output
# directly in the terminal from QEMU without having to interrupt the bootloader
```

Once the scripts are ready, they can be transfered to the remote server.

Then QEMU and any tools you might need must be installed on the remote server, before running this script.

As the goal is to reduce the amount of process running on the remote system and that Debian 12 uses systemd the command `systemctl isolate rescue-ssh.target` allows to load a rescue systemd target which only keeps a minimal amount of processes but still keeps the SSH server running and the current session open.

### Take over the system

Once the system is ready we run Marcan's script which does what was detailed in the past sections.

Here are the most interesting parts of his script.

```sh
# Port for the new SSH server (must not conflict with the existing one on port 22)
PORT=80

# Setup a root password in the ramdisk to be able to connect to the new SSH server
./busybox chroot . /bin/passwd

# Ensure the kernel won't shout because of our manipulations
./busybox rm -f etc/mtab
./busybox ln -s /proc/mounts etc/mtab

# Create the dir for the pivot
./busybox mkdir -p old_root

# Before the next command, prepare the virtual (pseudo) filesystems
# Any Gentoo users that took the time to understand the handbooks
# understands that.

# Preserving the current pty (SSH session)
./busybox mount --bind /dev/pts dev/pts

# Ensure the pty is preserved while making the pivot
TTY="$(./busybox tty)"
exec <"$TO/$TTY" >"$TO/$TTY" 2>"$TO/$TTY"

# Putting the script that will be used to pivot at a spot that will
# be bind mounted in place of the current init's path and run by the
# "telinit u" command that will follow (re-exeing the init)
./busybox echo "Preparing init..."
./busybox cat >tmp/${OLD_INIT##*/} <<EOF
#!${TO}/busybox sh

exec <"${TO}/${TTY}" >"${TO}/${TTY}" 2>"${TO}/${TTY}"
cd "${TO}"

./busybox echo "Init takeover successful"
./busybox echo "Pivoting root..."
./busybox mount --make-rprivate /
./busybox pivot_root . old_root
./busybox echo "Chrooting and running init..."
exec ./busybox chroot . /fakeinit
EOF
./busybox chmod +x tmp/${OLD_INIT##*/}
# As we can see the script doing what's explained in this article
# The make-rprivate is to ensure / is not bind mounted :
# In theory rprivate it is the default mode,
# but systemd does changes it.

# Starting the new server that will listen on a different port (80 in the script)
./busybox chroot . /usr/bin/ssh-keygen -A
./busybox chroot . /usr/sbin/sshd -p $PORT -o PermitRootLogin=yes

# Bind mounting the script at the init's path
./busybox mount --bind tmp/${OLD_INIT##*/} ${OLD_INIT}

# Re-exec the init and done, the pivot to the ramdisk is ready
telinit u

```

After the takeover the user needs to connect to the new SSH server on the new port.
Then the user must kill any other process that was run from the old root.

Once it is done, the kernel should allow to run the following commands without an error. 

```sh
# vda is used in this example, can be any device
swapoff /dev/vda5
umount -R /old_root
```

Great, the server is now running entirely from the ramdisk and it is now possible to install any operating system on the actual storage device.

### Install the new operating system

From now on, it is the most risky part of the procedure, this is the last chance to stop before any non-reversible changes are made to the server. If so use `echo b > /proc/sysrq-trigger`, and then kill the certainly timed-out SSH client and reconnect to the server as if nothing happened.

What is also risky is that if any change is made on the disk and then the control is lost and non-recoverable, **the server will be lost**.

To ensure the disk will have an entirely new partition table we can wipe its content using `wipefs -a /dev/vda`. *vda* is the name of the internal disk during the whole manipulation. This needs to be adapted in scripts if necessary.

From now if instead of NetBSD a user wants to install a Linux distribution, the method for Arch or Gentoo is the same as their official documentation.

But we can go further: Use an emulator with the host's internal storage used as QEMU's internal storage. Thanks to this, we can install any operating system also compatible with QEMU.

Beware this method relies on nested virtualisation, especially if the remote server is a virtual machine. Pure-software emulation might also work despite being slow.

With NetBSD we will use the nographics option and get the serial output directly in the terminal, but one can find a way to expose the SSH of the installer, or even VNC from QEMU directly.

The installer might require network to retrieve the "dist-files" and this is the case in our demonstration as we used the network ISO to reduce the need for space on the ramdisk.

The basic NAT(ted) user slirpns network is good enough as long as at the end of the installation process, a network configuration matching the current host's network configuration is setup, otherwise connectivity will be lost. The setup of a SSH server in the guest before finishing the procedure is also necessary to allow access afterwards.

The following script disables ipv6 but this can be enabled later.
Slirpns by default only preconfigures ipv4 connectivity and the NetBSD installer specifically did not fall back to ipv4 in the preparation of this article.
Disabling the ipv6 for QEMU fixes the problem.

Ideally it is better to virtualise the same network interface as the host's to ensure the same interface naming is kept. If it is not the case, beware, the name matching the host NIC (network interface) is necessary for the host to have connectivity.

Same advice for disk naming, if the disk was partitionned using GPT during the installation, and the generated fstab is using UUIDs to identify partitions, thus this should not be a problem. However, in MBR mode operating systems such as NetBSD tends to put the disk names from `/dev` which can be a problem as in QEMU the internal drive migh have a different name that the one given for the hosts' disk controler. To work around that, verify that the fstab entries are matching the host's hardware naming expectations.

I wrote the following helper script, also used in the video. We can notice only 128MB is allocated to the NetBSD installer. Indeed the goal is to keep the total memory usage under 2GB of RAM as the ramdisk, the kernel and the few running processes are already using most of the memory.

*Change /dev/vda by another device if needed*

```sh
#!/usr/bin/env bash

#-cpu host \

kvm \
        -machine pc \
        -m 128M \
        -vga none \
        -device qxl \
        -nic user,model=e1000,ipv6=off \
        -device virtio-scsi-pci,id=scsi0 \
        -drive file=/dev/vda,if=none,id=rt,format=raw,cache=none,discard=unmap,aio=native \
        -device scsi-hd,drive=rt,bus=scsi0.0 \
        -cdrom /boot-com.iso \
        -nographic \
        -boot d \
        -name 'NetBSD 10'
```

Afterwards once the installation is finished, quit QEMU by using the shortcut `Ctrl-A X`.

In case of an UEFI installation, beware that the UEFI entry is not configured on the server's firmware but only in the temporary virtual machine.
If so, an entry must be created with `efibootmgr` before rebooting.

NetBSD does not support secure boot, thus don't do this method on remote server with secure boot enabled, or install operating systems which support secure boot instead.

Then as the `fakeinit` is not a complete init, the only way to reboot the system is by using the following sysrq trigger.

```sh
echo b > /proc/sysrq-trigger
```

Kill the current SSH client or wait for the timeout.
Also wait some time for the system to reboot.

If everything was configured correctly, SSH access to the new operating system should be possible. Otherwise something wrong happened and either a physical access to the server, or a cloud control panel access is needed to come back to restore a working system.

That's it, the operating system is swapped.

## Final note

Again, please take a look at [Marcan's takeover.sh](https://github.com/marcan/takeover.sh) repository.

I believe there is potential for tools to be developped to automate this.

For example someone could develop an Ansible role to automate the process and reduce the risk of human mistakes.

A Terraform module could also be developed to provision custom operating systems out of providers' selections for those having an provisionning API.

This article can make it look manual and complex but in practice it is a relatively simple three steps process.

This website used to be hosted on a cheap 2GB of RAM VPS running on [NetBSD](https://www.netbsd.org/) installed using this method.

Thanks for reading.

<!-- [![Powered my NetBSD Logo](/powered-by-NetBSD.png)](https://www.netbsd.org/) -->

*Contact me through private messages [on unitedbsd.com](https://www.unitedbsd.com/u/naguam) for any comments, requests or corrections.*
