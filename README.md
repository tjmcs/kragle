# kragle (v1.0)

This project contains the files, scripts, and documentation for the process used
to in the Automation 2.0 project to build out the **Automation Node**. There is
one automation node per **Pod** (where a pod is logical group of racks that are
connected to each other in the datacenter network), and that node is is the
node that contains all of the dependencies necessary to:

* **Discover** new nodes that are added to the pod. This process of automated
discovery is driven by the Hanlon instance that is running on the automation
node (and a default, discover-only policy that has been added to that Hanlon
instance).
* **Provision** new operating systems or hypervisors to those nodes. This
process of policy-based provisioning is driven using Ansible and Hanlon, with
Ansible creating the policies necessary to provision the right OS/hypervisor to
the right node at the right time.
* **Deploy** new platforms into the OS/hypervisor instances that were
provisioned to those nodes. The process of platform deployment is driven using
Ansible.
* **Configure** the infrastructure associated with the pod, the nodes in the
pod, and the platforms deployed to those nodes. The process of configuration at
the infrastructure, OS/hypervisor, and platform layer is driven using Ansible.

Since the automation node itself is meant to contain the tools that we normally
use to discover and provision an operating system to a node, we need a
semi-automated process that can be used to create a new automation node. In the
sections that follow, we describe just such a process.

## Building a 'boostrap' thumb drive

RedHat Enterprise Linux (RHEL) will be used as the base OS for our automation
nodes. Our automation nodes are actually Intel NUCs, but any server that
supports RHEL as an operating system and that is attached to the management
network of the pod should be useable for this purpose. To avoid dependencies on
other servers that may or may not exist in the datacenter yet (it can't be
turtles all the way down, after all), we install an operating system onto these
automation nodes using an unattended install from an ISO that has been burned to
USB thumb drive. In this section of document, we describe how to build that
thumb drive with a few basic Linux commands and a RHEL server ISO (which we
downloaded directly from RedHat).

### Step 1: Construct an editable copy of the RHEL-7.1 ISO

The first step in this process is to modify the standard RHEL ISO so that it
contains a couple of kickstart files and it is setup to perform an unattended
install using one of those kickstart files by default. The kickstart file that
is actually used will differ depending on whether the thumb drive is beind used
to perform an install via a legacy (isolinux) boot or an EFI boot (this is
because what we feel to be a more suitable, non-default, partitioning table is
used in our kickstart file and the requirements for a legacy boot system
differ from those for an EFI-boot system).

To modify the files we need to modify in the RHEL ISO, we first need to
unpack the RHEL ISO into a local directory (so that we can edit those
files). First, mount the RHEL ISO file locally (here, we're going to assume it's
mounted under the `/mnt/cdrom` directory, it may be different on your Linux
system):

```bash
mount -o loop rhel-server-7.1-x86_64-dvd.iso /mnt/cdrom

```

next, use the `rsync` command to create a copy of the ISO file's contents in a
local directory (we're only showing partial output in this example):

```bash
$ rsync -avz /mnt/cdrom/ ./rhel-server-7.1-x86_64-customized
sending incremental file list
./
.discinfo
.treeinfo
EULA
GPL
RPM-GPG-KEY-redhat-beta
RPM-GPG-KEY-redhat-release
TRANS.TBL
media.repo
EFI/
EFI/BOOT/
 .
 .
 .
repodata/productid
repodata/repomd.xml

sent 3,851,410,330 bytes  received 139,172 bytes  19,209,723.20 bytes/sec
total size is 3,984,379,789  speedup is 1.03
$
```

Now that we have a copy of the contents of the RHEL ISO available locally,
we can proceed with the remastering process.

### Step 2: Modifying the standard RHEL-7.1 ISO to use a kickstart file by default

Adding support for use of a kickstart file during the install process (when
booting up and installing from the thumb drive that we're creating) is a
simple, but poorly documented process. This is especially true when you consider
that we need to support booting of both legacy (isolinux) and EFI-based systems
using the image we're building. The RHEL ISO supports both, so we'll need to
ensure that we maintain that support in the remastered ISO that we burn to our
thumb drive.

The first thing we need to do is to create a pair of kickstart files (one for
legacy/isolinux systems and one for EFI-based systems). While these kickstart
files only differ by a few lines (basically an extra partition is needed to
successfully boot and install RHEL to an EFI-based system), there really is no
simple way to combine them into a single kickstart file. As sucy, we have
constructed two separate kickstart files (which can be found in the
[kickstart-files](kickstart-files) subdirectory in this repository).

Once we've added those two kickstart files to our working directory (we place
them into a `kickstart` subdirectory), we can start modifying the contents of
our working directory so that the remastered ISO will make use of these
kickstart files during the boot and installation process. This requires
modifications to the `./rhel-server-7.1-x86_64-customized/isolinux/isolinux.cfg` and
`./rhel-server-7.1-x86_64-customized/EFI/BOOT/grub.cfg` files. In both cases,
we need to add an argument to ensure that one of the boot options presented by
the menus defined in those two files both provide an option that uses the kickstart
files we just added to our working directory.

It should be noted that we'll be making our changes to the second menu item
defined in both of these files (that's the default menu item for all RHEL
ISOs, so we want to maintain this behavior). Also, we've actually created a 'patch'
file that can be used to perform all of these changes in a single command so that
the process of modifying the stock RHEL ISO is as simple as possible. That patch
file can be found [here](patch-files/iso-mods.patch), and here's an example of
how you could use that patch file locally (assuming it is located in the same
directory as the working directory that we created with our `rsync` command, above):

```bash
$ cd rhel-server-7.1-x86_64-customized
$ patch -p1 < ../iso-mods.patch
patching file EFI/BOOT/grub.cfg
patching file isolinux/isolinux.cfg
patching file kickstart/ks-isolinux.cfg
patching file kickstart/ks-uefi.cfg
$ cd ..
$
```

### Step 3: Remastering the ISO from the working directory

Now that we've modified the ISO to contain (and use) our new kickstart files,
it's time to rebuild that image on our thumb drive. The process used here is
actually a multi-step process, with much of the complexity required being due
to our desire to support both legacy and EFI-based installs off of the thumb
drive we're creating. To get started, we need to convert the modified contents
of our working directory back into an ISO file. This is easily accomplished
using the `geniso` command:

```bash
$ genisoimage -untranslated-filenames -volid 'RHEL-7.1 Server.x86_64' -J -joliet-long -rational-rock -translation-table -input-charset utf-8 -x ./lost+found -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -eltorito-alt-boot -e images/efiboot.img -no-emul-boot -o rhel-server-7.1-x86_64-custom.iso -T ./rhel-server-7.1-x86_64-customized/ 2>&1 | tee t.t
Warning: creating filesystem that does not conform to ISO-9660.
Using pacemaker-libs000.1.12-22.el.rp for  ./rhel-server-7.1-x86_64-customized/addons/HighAvailability/pacemaker-libs-1.1.12-22.el7.i686.rpm (pacemaker-libs-1.1.12-22.el7.x86_64.rpm)
Using corosynclib-devel000.3.4-4.e.rp for  ./rhel-server-7.1-x86_64-customized/addons/HighAvailability/corosynclib-devel-2.3.4-4.el7.i686.rpm (corosynclib-devel-2.3.4-4.el7.x86_64.rpm)
 .
 .
 .
 99.20% done, estimate finish Wed Jun 10 12:42:40 2015
 99.46% done, estimate finish Wed Jun 10 12:42:40 2015
 99.71% done, estimate finish Wed Jun 10 12:42:40 2015
 99.97% done, estimate finish Wed Jun 10 12:42:40 2015
Total translation table size: 1740750
Total rockridge attributes bytes: 743454
Total directory bytes: 1335296
Path table size(bytes): 2518
Max brk space used 87a000
1950572 extents written (3809 MB)
$
```

The arguments to the `geniso` command shown above should be used as is if you
are hoping to have a working system when the resulting ISO file
(rhel-server-7.1-x86_64-custom.iso) is actually burned to your thumb drive.
Specifically, the volume ID specified in this command ('RHEL-7.1 Server.x86_64')
should be left unchanged since this string is used as a LABEL when identifying
where to find the kickstart files we added to the `/isolinux/isolinux.cfg` and
`/EFI/BOOT/grub.cfg` in this ISO (above).

Now that we have a remastered ISO, the next step is to make it bootable on a
UEFI-based system. This is accomplished using the `isohybrid` command, as is
shown here:

```bash
$ isohybrid -u rhel-server-7.1-x86_64-custom.iso
isohybrid: Warning: more than 1024 cylinders: 3810
isohybrid: Not all BIOSes will be able to boot this device
$
```

The resulting remastered, UEFI-bootable ISO file can then be "burned" to a
thumb drive using a simple `dd` command:

```bash
$ dd if=rhel-server-7.1-x86_64-custom.iso of=/dev/sdc bs=1024k
3810+0 records in
3810+0 records out
3995074560 bytes (4.0 GB) copied, 168.317 s, 23.7 MB/s
$
```

At this point, we're almost done. Unfortunately, the changes that we made
to the `/EFI/BOOT/grub.cfg` file in our ISO will not appear in the UEFI-boot
process without one more change (this is because the `isohybrid` command we ran,
above, created a new UEFI-bootable partition in our remastered ISO file, but
it reset the grub.cfg file in that partition to it's default behavior in the
process). To resolve this we simply have to mount the partition that was added,
then copy over the appropriate file from our working directory.

Of course, to mount the new partitions that were added to the thumb drive, by our
`dd` command (above), we need to eject it and re-mount the thumb drive (to force
the system to detect the new partitions). That is easily accomplished with the
following pair of commands (assuming your thumb drive has been inserted into an
appropriate USB port and was detected as `/dev/sdc`):

```bash
$ eject /dev/sdc
$ eject -t /dev/sdc
```

and then the appropriate file can be copied over to the new partition as follows:

```bash
$ mount /dev/sdc2 /mnt/usb
$ cp rhel-server-7.1-x86_64-customized/EFI/BOOT/grub.cfg /mnt/usb/EFI/BOOT/grub.cfg
$ umount /mnt/usb
$ eject /dev/sdc
```

You can now remove the thumb drive from your Linux system and use it to provision
a RHEL-7.1 OS any new automation nodes you might want to build.
