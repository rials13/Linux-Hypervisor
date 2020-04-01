# Linux-Hypervisor
This is a section of notes for setting up a linux host to run qemu/kvm/libvirt with both a windows and a macos vm with passthrough

**Step 1: Choose your OS.**
- I started with Manjaro because things worked on it with other guides but ended up on Mint.
- It's worth noting that an ACS override is easier on Manjaro with the AUR.

-My config:
Mobo + processor: Gigabyte z97x-SOC-CF with i7-4790
- 16x Slot 1: MSI GTX970 - passed through to VMs
- 16x Slot 3: R9 290 - primary
- 16x Slot 4: Fresco Logic FL1100 USB 3.0 - passed through to VMs

This motherboard does not properly separate iommu groups so ACS override patch it required.
```sudo uname -r``` for checking what kernel is running

**Enabling IOMMU** 
- in /etc/default/grub GRUB_CMDLINE_LINUX_DEFAULT add ```intel_iommu=on iommu=pt```
- in a file called ```iommu.sh```copy and paste from TODO:LINK TO PASSTHROUGH POST GITHUB FOR IOMMU.SH

- Run it to check for IOMMU Groups ```./iommu.sh```

**Manjaro:**
- ACS override patch linux-vfio AUR package
- in /etc/default/grub GRUB_CMDLINE_LINUX_DEFAULT add ```pcie_acs_override=downstream,multifunction```

**Mint:**
- Download (a version with a kernel higher than the one running) the headers and image file from https://queuecumber.gitlab.io/linux-acs-override/
- use ```sudo dpkg -i <file-name.deb>``` on both files then reboot
- in /etc/default/grub GRUB_CMDLINE_LINUX_DEFAULT add ```pcie_acs_override=downstream```
- Update grub ```sudo update-grub```
- reboot and use```sudo uname -r``` to verify the new ACS patched kernel is running

**Vulkan:**
- With both Manjaro and Mint, Steam games on linux would occasionally not run due to not having vulkan natively installed (Dota Underlords). You may only encounter this issue with gfx cards that are fringe vulkan cards (Firepro M5100 and R9 290)
- in etc/default/grub GRUB_CMDLINE_LINUX_DEFAULT add ```radeon.cik_support=0 amdgpu.cik_support=1 radeon.si_support=0 amdgpu.si_support=1```
- add a PPA to your system ```sudo add-apt-repository ppa:oibaf/graphics-drivers```
- install from ppa ```sudo apt-get install libvulkan1 mesa-vulkan-drivers vulkan-utils```


