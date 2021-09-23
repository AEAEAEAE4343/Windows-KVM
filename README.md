# Windows/KVM
### Before I begin

I need to say I don't know the specifics on Intel. I namely don't know what CPU model you need to use on unsupported CPU's. I couldn't find Comet Lake in the cpu-map from QEMU and I don't know the list of CPU features myself. If anyone can supply this information it would be appreciated.

### Install Windows 11 on unsupported hardware using KVM/Linux

Microsoft will be releasing their next version of Windows, rightfully named Windows 11, on October 5th this year. Unfortunately Microsoft has taken a bold move with the system requirements. To install Windows 11, or to upgrade from Windows 10, you'll be needing the following system requirements:

 - One of [these approved CPU's.](https://docs.microsoft.com/nl-nl/windows-hardware/design/minimum/windows-processor-requirements) Microsoft also states that you need at least 1Ghz and 2 cores, but that doesn't make sense as they have a predefined list of CPU's you can use already.
 - A version 2.0 TPM (Trusted Platform Module)
 - 4 GB of RAM
 - 64 GB of storage space
 - A GPU with DirectX 12 and WDDM 2.0
 - UEFI firmware

The first two requirements are the most troublesome. The oldest CPU in the compatibility list is from 2017 which means many older, but still capable, builds will be listed as incompatible, even though the hardware capabilities themselves might exceed some CPU's in the list.
The TPM is also a deal breaker for a lot of people. A TPM is something motherboard manufacturers need to incorporate into their hardware, but many older motherboards don't have these feature included (by default).

Unfortunately there's not much we can do. Windows 11 will not update if these requirements are not met, which puts the system at risk in terms of security. The only way to circumvent the requirements, is to lie to Windows about what hardware your PC has.

## QEMU/KVM

 **QEMU** is a free and open-source hypervisor. It emulates the machine's processor through dynamic binary translation and provides a set of different hardware and device models for the machine, enabling it to run a variety of guest operating systems. It could interoperate with Kernel-based Virtual Machine (KVM) to run virtual machines at near-native speed. 

> Source: Wikipedia

Using QEMU/KVM you manipulate what hardware Windows 11 sees when it's running on your PC. You don't have a TPM? Tell QEMU to add one. Your CPU doesn't meet the requirements? Make QEMU tell Windows that you have a 32-core AMD EPYC CPU!

## Linux

**Linux** is a family of open-source Unix-like operating systems based on the Linux kernel, an operating system kernel first released on September 17, 1991, by Linus Torvalds.

> Source: Wikipedia<br>
> It's Linux, not GNU/Linux. Stop crying about it. 

We use Linux because it's extremely configurable but not too expansive. We can start with Manjaro Minimal and we'll add stuff as we need it.

## What do I need?

Before committing to anything, make sure your PC will actually be able to run Windows 11 smoothly. If it can't run Windows 11 natively, it won't be able to run Windows 11 in a Virtual Machine either.

Your CPU must have at least the performance requirements (1 Ghz, 2 cores) met. Otherwise Windows 11 will run like crap on your PC.

You have to enable VT-x for Intel processors or SVM for AMD processors in your BIOS. **You can not use KVM without VT-x or SVM.** Please look it up on Google: `[Motherboard name] how to enable virtualization`

Also, you'll need a couple of hours. Installing and deploying a setup like this has a lot of steps and those steps take time. Also, you might need to troubleshoot some stuff, so count on it taking about 2-3 hours total. Note that you can pause at any moment, you're not playing an online match of CS:GO.

## Partitioning your disk

This step can vary. A lot. Unfortunately this step can also be pretty complicated depending on what you already have and what you want to do. You'll have to look up how to partition a computer yourself. Refrain from using Disk Manager in Windows, use DiskPart instead. If you really can't work with the DiskPart command-line, use a dedicated partition tool like EaseUS Partition Master, MiniTool Partition Wizard or similar tools.
As for the layout itself, for Windows/KVM I recommend the following based on what disk configuration you have:

---
 - **128/256GB SSD + 1TB or more HDD\***
I'd recommend you save the SSD for Windows. Yes, Linux will take longer to load but it'll be worth it in the end. For the HDD use the following configuration:

| Partition number: | 1           | 2              | 3                   |
|-------------------|-------------|----------------|---------------------|
| Type              | Linux Btrfs | Linux Swap     | NTFS Storage        |
| Size:             | 50GB        | 2-32GB (2*RAM) | The rest of the HDD |

---
 - **1TB or more HDD\***

| Partition number: | 1           | 2              | 3            | 4                   |
|-------------------|-------------|----------------|--------------|---------------------|
| Type              | Linux Btrfs | Linux Swap     | NTFS Windows | NTFS Storage        |
| Size:             | 50GB        | 2-32GB (2*RAM) | 120-250GB    | The rest of the HDD |
---
 - I can keep going with possible configurations, but the two above are most common so I'll keep it to that. Just make sure that you have:
	 - **Two** EFI system partitions (100MB FAT32 partitions), one for Linux and one for Windows
	 - A 50GB Linux Btrfs partition for storing Linux and your KVM configs
	 - Linux swap. Not necessarily needed but always handy for when you actually want to do something on Linux
	 - A partition to install Windows on. Note that you can also use a disk image on your storage partition. This guide includes instructions for both methods.
	 - (Optional) A partition to store your data. This should be all the left-over space.

**Before continuing,** format **both** EFI System Partitions. Otherwise they'll conflict with each other.
 > \*
 > (1): If you already have a NTFS storage partition, you can also shrink it by the size of the other partitions, and add the partitions at the end.

## Installing Manjaro XFCE Minimal

I'm not explaining this. This tutorial is about QEMU/KVM, not Linux. Just make sure your Linux installation is bootable without any boot medium. Make sure to use the last EFI system partition as the boot drive. Without manually installing through DISM, Windows installs its bootloader on the first ESP.

## Installing QEMU/KVM

Type the following in a Terminal window:
```
sudo pacman -Sy virt-manager qemu vde2 ebtables dnsmasq bridge-utils openbsd-netcat openssh
```
The command will install QEMU, virt-manager, some virtual networking stuff and SSH for troubleshooting. After that you want to enable the libvirt systemd service and start it:
```
sudo systemctl enable libvirtd.service
sudo systemctl start libvirtd.service
sudo systemctl enable sshd
sudo systemctl start sshd
```

## Determine your drive assignments

You'll need to know where Linux assigns your partitions. First you'll need to know what device names they got (eg. /dev/sda). You can do this by opening GParted under the System section of your start menu. When you have a list of device names run the command `ls -lh /dev/disk/by-id/`. Note down the device id symlinks instead. Device names might change after boot, device ids don't. 

Do this for all partitions you created.

## Creating the virtual machine

This is where the fun starts. Launch 'Virtual Machine Manager' from the Settings section in your launch ('start') menu. 

By default it'll probably only have a LXC connection configured, which won't work for us. Click on File -> Add Connection to add a connection to the QEMU/KVM hypervisor. If all goes well, the generated URI should be "qemu:///system". If so, press Connect.

Now we can continue with creating the VM. Select the button with the plus sign to start creation of the VM. If you want to boot from a pre-existing installation, select Manual install, otherwise select Local ISO and select your ISO.

When it asks you what operating system you want to install, select Windows 10. Note that you should first check if there's an option for Windows 11 available. It's just that at the time of writing it isn't, and Windows 10 is the next best option.

For RAM, assign 75% of your RAM. Most people will say this is way too much and that it'll mess with the host OS, which would be true if we chose a bigger distro. But on Manjaro Xfce Minimal it works fine. Though if you have less than 8GB of total RAM, you should probably do 50% as your VM might crash with more. You'll know you have too much assigned when the VM stops just as the Windows boot screen finishes.

For CPU I'd recommend all cores if you plan on only using Windows. If you plan on sometimes using both at the same time (like through SSH), I'd recommend leaving one or two cores unassigned for Linux to use and not interfere with Windows.

If you are going the **disk image route**, now is the time to create it.  First you'll need to mount your storage partition. This can be done through the terminal. I'd recommend you create a folder inside the /mnt folder (eg. /mnt/storage/) and mount it to there. Make sure to use sudo in your command to get the right permissions. See a tutorial [here](https://linuxhint.com/linux_mount_command/). Note that you'll also have to [make this mount permanent through fstab.](https://www.mvps.net/docs/how-to-permanently-mount-partitions-on-linux/)

Once mounted, you'll have to create a qcow2 image on the drive using `qemu-img`.

If you're going the **partition route**, you'll need to select custom storage. In the text box enter the device id of your windows partition you found earlier. 

## Set VM to UEFI and verify hypervisor

On the next page, select customize configuration and press Finish. Select UEFI (the one with secboot) for Firmware. Verify that KVM is listed as the hypervisor. If it isn't you'll need to go back all the way to Installing QEMU/KVM and verify that you've done everything.

## Patching CPU and TPM

If you have a unsupported CPU you'll have to edit the `<cpu>` section to reflect more modern hardware. Go back to the VM configuration and click on XML.
For AMD replace it with the following:

    <cpu mode="custom" match="exact" check="none">
      <model fallback="forbid">EPYC-Rome</model>
      <feature policy="disable" name="svm"/> 
      <!-- Set policy to "enable" if you want to use WSL or other virtualization tools inside the VM. -->
      <!-- Note that this requires you to enable nested virtualization in KVM. Look up how on Google. -->
      <topology sockets='1' cores='x' threads='x'/>
    </cpu>
For Intel replace it with the following:

    [CLASSIFIED] See top of README
Replace the x values with the number of cores and number of threads per core your CPU has.

Next we'll be adding a TPM. If you don't have one, run `sudo pacman -Sy swtpm` in a terminal first. This'll allow you to create a virtual TPM. In the hardware manager, select Add Hardware, and in the list select TPM. Select the CRB model, and if you have a TPM select passthrough. Add the TPM.

## Finishing the VM creation

You can now press Begin installation at the top. This'll create and boot up your Virtual Machine.

## Verifying Windows works

If you haven't installed Windows yet, you'll be greeted with Windows Setup. Install Windows as usual, just make sure you select the correct target partition.

If Windows boots up to the login screen/the Out-Of-Box Experience, you're set. Shutdown the VM. 

## PCI passthrough

**You can not passthrough an iGPU.**
**When you passthrough a GPU you have to also passthrough a USB controller, or you won't be able to send input to your VM**

To use PCI passthrough you first need to verify your IOMMU groups. Basically, when you want to passthrough a PCI device to your VM, you need to pass through the entire IOMMU group, otherwise the VM will fail to start. You should see a IOMMU group as a group of hardware that's dependent on each other. Download `testiommu.sh` from this repository and save it to your user folder. In a terminal type in the following commands:
```
chmod +x testiommu.sh
./testiommu.sh
```
This should display all of your IOMMU groups, and the devices within those groups. Make sure that Linux is not dependent on any hardware in the IOMMU group of the device you want to passthrough. If you are running a Single GPU and still want passthrough, check the next section.

Select the (i) icon on your VM screen to modify the hardware. Remove the following hardware based on what you'll passthrough:
 - **USB controller passthrough:**
 - Console 1
 - Channel Spice
 - Tablet
 - **GPU passthrough:**
 - Video QXL
 - Display Spice
 - **Motherboard audio passthrough:**
 - Sound ich9

Now we're going to configure some devices for PCI passthrough. Click on Add Hardware and select PCI host-device. You'll see a list of all the PCI device on your system. Select the device you want to pass through. **Repeat this step for all the devices in the IOMMU group!**

## Single GPU passthrough

I said earlier that you can't passthrough anything that Linux depends on. That is still true, but you can make Linux independent on the GPU by disabling the GPU driver. You can configure the VM to automatically do so but it involves some steps.

First of all, run the following commands in a terminal:
```
sudo wget 'https://raw.githubusercontent.com/PassthroughPOST/VFIO-Tools/master/libvirt_hooks/qemu' -O /etc/libvirt/hooks/qemu
sudo chmod +x /etc/libvirt/hooks/qemu
sudo mkdir -r /etc/libvirt/hooks/qemu.d/[Insert VM name]/prepare/begin/
sudo mkdir -r /etc/libvirt/hooks/qemu.d/[Insert VM name]/release/end/
```
---
Now you'll want to create your start script. Run:
```
sudo touch /etc/libvirt/hooks/qemu.d/[Insert VM name]/prepare/begin/start.sh
sudo chmod +x /etc/libvirt/hooks/qemu.d/[Insert VM name]/prepare/begin/start.sh
sudo nano /etc/libvirt/hooks/qemu.d/[Insert VM name]/prepare/begin/start.sh
```
Here you'll need to do the following:

 - Stop display manager
 - On GDM you need to kill all sessions manually
 - Unbind vtconsole
 - Detach the GPU from Linux
 - Load the VFIO module

Example script:
```
#!/bin/bash
# This is for potential troubleshooting later on
set -x

# Stop display-manager
systemctl stop display-manager.service

# Unbind vtconsoles
echo 0 > /sys/class/vtconsole/vtcon0/bind
# Sometimes there's a second vtconsole
# echo 0 > /sys/class/vtconsole/vtcon1/bind

# Sleep to avoid race condition, can be tuned to be shorter or longer depending on the system
sleep 3

# Detach the GPU
virsh nodedev-detach [Insert PCI name of GPU*]
virsh nodedev-detach [Insert PCI name of GPU audio*]

# Load VFIO
modprobe vfio-pci
```
Press Ctrl+X and type Y and then press Enter to save and quit Nano.

---
Now you'll want to create your stop script. Run:
```
sudo touch /etc/libvirt/hooks/qemu.d/[Insert VM name]/release/end/revert.sh
sudo chmod +x /etc/libvirt/hooks/qemu.d/[Insert VM name]/release/end/revert.sh
sudo nano /etc/libvirt/hooks/qemu.d/[Insert VM name]/release/end/revert.sh
```
Here you'll need to do the following:

 - Attach the GPU
 - Reload the drivers
 - Revind vtconsole
 - Start display-manager

Example script:
```
#!/bin/bash
# This is for potential troubleshooting later on
set -x

# Attach the GPU
virsh nodedev-reattach [Insert PCI name of GPU*]
virsh nodedev-reattach [Insert PCI name of GPU audio*]

# Reload drivers
# Use the bottom four when you have proprietary NVIDIA drivers and use nouveau if you have the preinstalled drivers.
modprobe nouveau
# modprobe nvidia_modeset
# modprobe nvidia_uvm
# modprobe nvidia_drm 
# nvidia-xconfig --query-gpu-info > /dev/null 2>&1

# Rebind vtconsoles
echo 1 > /sys/class/vtconsole/vtcon0/bind
# Sometimes there's a second vtconsole
# echo 1 > /sys/class/vtconsole/vtcon1/bind

# Start display-manager
systemctl start display-manager.service

```
Press Ctrl+X and type Y and then press Enter to save and quit Nano.

---
 > *: Look at the output from the testiommu script. Find the line of your device and look at the first few numbers. To convert this to the PCI name of your device do the following: 
 > xx:yy:z -> pci_0000_xx_yy_z
 
**Do not just copy these scripts.** They are just templates for you to make your own scripts.


## Fixing NVIDIA driver (Error 43)

NVIDIA is for some reason not a great fan of GPU passthrough. They have purposely broken their drivers in passthrough configurations. Luckily this can be easily fixed. Go back to the VM configuration and click on XML. Add the following under `<features>`:
```
<kvm>
  <hidden state="on"/>
</kvm>
<vmport state="off"/>
<ioapic driver="kvm"/>
```
This will make the GPU think it's running on real hardware, making the driver activate successfully.

## General troubleshooting

Try running the VM manually from a terminal (with single gpu passthrough you should use ssh from another pc):
```
sudo virsh start [insert vm name]
```
This will output anything that goes wrong to your terminal.

---
Undo the last steps you did until the VM boots again. This will help you determine what step you did incorrectly.

---
You can always ask for help on the Issues page.

# Credits

https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF
<br>Extensive guide on PCI passthrough

https://github.com/joeknock90/Single-GPU-Passthrough
<br>Very clean guide for passing through a single GPU.

https://github.com/PassthroughPOST/VFIO-Tools
<br>Useful libvirt hook that splits the script for each VM
