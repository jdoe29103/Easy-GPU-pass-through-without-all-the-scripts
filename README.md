# Easy-GPU-pass-through-without-all-the-scripts

I have yet to find a GPU pass-through guide that actually explains what you need to do and does not involve using a bunch of random OS and hardware specific scripts. These instructions are designed so that, if you need, you can search for OS or hardware specific information as it is impossible to account for every configuration in one guide.

GPUs, unlike most devices cannot easily be let go of and grabbed by other drivers. Therefore, GPU pass-through requires some extra configuration before configuring a virtual machine. This guide will be divided into two parts. Part 1, prepping the guest GPU to be used for pass-through (the "hard" part) and Part 2, setting up the virtual machine (the "easy" part).

This guide assumes you have a chipset with workable IOMMU groupings and virtualization options enabled in the BIOS of your system. This is different for every motherboard so you will need to reference your manual for that. The GPU you plan to pass needs to be in its own IOMMU group. If your GPU does not have its own IOMMU group try using a different PCI slot. If that doesn't work, your last hope is using kernel patches to bypass this issue, which is not covered in this guide. This guide also assumes you are on a Debian based system but will work with other distributions.

You can use this script to investigate your IOMMU groups if need be:
```
#!/bin/bash
shopt -s nullglob
for g in /sys/kernel/iommu_groups/*; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

# **Part 1: Binding guest GPU to the VFIO driver on boot**

The easiest way to to get around the nuances of passing through a GPU is to bind the guest GPU to the VFIO driver (the driver that lets QEMU/KVM use it) at boot so that the regular GPU driver doesn't get a chance to. You do not have to do this on boot but those solutions are more complex and less reliable.

1. First we need to get some identifying information about the GPU we are going to pass. Run `lspci -nn`. `lspci` lists all of the available PCI devices and the `-nn` argument shows us the PCI vendor and device codes which is what we are looking for. You can also use grep to narrow down the list e.g. `lspci -nn | grep "NVIDIA"` or `lspci -nn | grep "AMD`. You should get something like this:
```
03:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU116 [GeForce GTX 1660 SUPER] [10de:21c4] (rev a1)
03:00.1 Audio device [0403]: NVIDIA Corporation TU116 High Definition Audio Controller [10de:1aeb] (rev a1)
03:00.2 USB controller [0c03]: NVIDIA Corporation TU116 USB 3.1 Host Controller [10de:1aec] (rev a1)
03:00.3 Serial bus controller [0c80]: NVIDIA Corporation TU116 USB Type-C UCSI Controller [10de:1aed] (rev a1)
07:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA104 [GeForce RTX 3070 Ti] [10de:2482] (rev a1)
07:00.1 Audio device [0403]: NVIDIA Corporation GA104 High Definition Audio Controller [10de:228b] (rev a1)
```

2. From the list, find the card you want to pass. Assuming your GPU was made in the last 20 years, it should have at least a VGA controller and a sound card (Audio device). Some GPUs will have extra USB controllers as well. Copy the device codes (characters in brackets at the end) for every device associated with your GPU. e.g. `10de:2482` and `10de:228b` for the 3070 or `10de:21c4`, `10de:1aeb`,`10de:1aec`, and `10de:1aed` for the 1660.

3. Next, we need to set a kernel parameter that specifies which PCI devices will be handled by the VFIO PCI driver and gets passed to the kernel at boot. To do this, run `sudo nano /etc/default/grub` (or using any editor of your choice).

4. Find the line`GRUB_CMDLINE_LINUX_DEFAULT=`, and add the parameter `vfio-pci.ids=` followed by a comma separated list of all the device codes for your GPU. e.g. `GRUB_CMDLINE_LINUX_DEFAULT="vfio-pci.ids=10de:2482,10de:228b"`. Save and exit your editor.

5. Run `sudo grub-mkconfig -o /boot/grub/grub.cfg` to generate a new grub configuration file with the changes we just made.

6. Reboot.

7. Lastly, we need to tell the VFIO kernel module to bind our devices to the driver in our initial RAM filesystem. Run `sudo nano /etc/modprobe.d/vfio.conf` to create a new config.

8. Add the line `options vfio-pci ids=YOURDEVICECODESHERE` with your device codes in comma separated list like before - e.g. `options vfio-pci ids=10de:2482,10de:228b`.

9. Hit return and add the line `softdep nvidia pre: vfio-pci` if you are using the proprietary NVIDIA driver or `softdep nouveau pre: vfio-pci` if you are using the open source driver. This line prevents any NVIDIA drivers from grabbing the GPU before the VFIO driver. If you are using AMD, refrences those drivers instead. Save and exit your editor.

10. Run `sudo update-initramfs -c -k $(uname -r)` to update the initial RAM filesystem. The `-c` creates a new `initramfs` and the `-k $(uname -r)` specifies the currently running kernel. Make sure this gives no errors.

11. Reboot (See the pitfalls section if you have display issues after this).

12. Once you reboot, run `lspci -k | grep -E "vfio-pci|NVIDIA"`. This should return the same list from step one exept now you should see `Kernel driver in use: vfio-pc` under all the devices of your GPU. e.g.
```
07:00.0 VGA compatible controller: NVIDIA Corporation GA104 [GeForce RTX 3070 Ti] (rev a1)
	Subsystem: NVIDIA Corporation GA104 [GeForce RTX 3070 Ti]
	Kernel driver in use: vfio-pci
07:00.1 Audio device: NVIDIA Corporation GA104 High Definition Audio Controller (rev a1)
	Subsystem: NVIDIA Corporation GA104 High Definition Audio Controller
	Kernel driver in use: vfio-pci
```
If you did everything correctly your GPU is ready to be passed to a virtual machine.

# **Part 2: Configuring a virtual machine**

From this point on, all you really need to do is setup a virtual machine to fit your needs outside of a few specific parameters you need when passing through a GPU.

1. Install qemu and libvirt `sudo apt install qemu libvirt-daemon-system libvirt-clients bridge-utils virt-manager ovmf`

2. Add your user to the libvirt group to avoid permission issues `sudo useradd -g $USER libvirt` then `sudo useradd -g $USER libvirt-kvm`

3. When you create a new virtual machine make sure chipset is set to `Q35` (this is default) and firmware is set to `UEFI x86_64:/usr/share/OVMF/OVMF_CODE_4M.fd` everything else is personal preference.

4. To add the GPU for pass-through click on `Add Hardware` in the virtual hardware details section, go to `PCI Host Device` and add all of the associated devices for your GPU.

5. You will also need to install the [virtio drivers](https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md) inside the virtual machine in order to use the devices you pass through.

6. Dial

# **Pitfalls**

If you're like me and don't have one of the very few and expensive motherboards that have perfect IOMMU groupings you may find yourself having to use the first PCI slot for the guest GPU making it the "primary" GPU. If the primary GPU is grabbed by the vfio driver you may run into an issue where xorg won't start using the second GPU giving you no display. However, this is fixable.

1. Run `sudo nano /etc/X11/xorg.conf.d/10-gpu.conf` to create an Xorg config file.

2. Paste this block in the config:
```
Section "Device"
    Identifier     "whatever"
    BusID          "PCI:3:0:0"
    Driver         "nvidia"
EndSection
```
3. You will need to edit the `BusID` and `Driver` to the correct one for your GPU. This will tell Xorg to use the second GPU.
