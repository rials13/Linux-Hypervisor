# Linux-Hypervisor
This is a section of notes for setting up a linux host to run qemu/kvm/libvirt with both a windows and a macos vm with passthrough.  I know there are a lot of guides out there (all of my knowledge was from Googling how to do something) so this is as much a refference for me and to highliht a few things that some other guides don't include - all in one place.

**Step 0: Choose your OS.**
- I started with Manjaro because things worked on it with other guides but ended up on Mint.
- It's worth noting that an ACS override is easier on Manjaro with the AUR.

-My config:
Mobo + processor: Gigabyte z97x-SOC-CF with i7-4790
- 16x Slot 1: MSI GTX970 - passed through to VMs
- 1x Slot: Empty
- 16x Slot 2: Empty
- 16x Slot 3: R9 290 - primary
- 16x Slot 4: Fresco Logic FL1100 USB 3.0 - passed through to VMs

This motherboard does not properly separate iommu groups so ACS override patch it required.
```sudo uname -r``` for checking what kernel is running

**Step 1: Enabling IOMMU** 
- in /etc/default/grub GRUB_CMDLINE_LINUX_DEFAULT add ```intel_iommu=on iommu=pt```
- Update grub ```sudo update-grub```
- in a file called ```iommu.sh```copy and paste from https://github.com/PassthroughPOST/Example-OSX-Virt-Manager/blob/master/iommu.sh

- Run it to check for IOMMU Groups ```./iommu.sh```
  - Record the GPU (and HDMI Audio) and USB card to Passthrough. example: ```10de:13c2``` from the GTX 970

**Manjaro:**
- ACS override patch linux-vfio AUR package
- in /etc/default/grub GRUB_CMDLINE_LINUX_DEFAULT add ```pcie_acs_override=downstream,multifunction```

**Mint:**
- Download (a version with a kernel higher than the one running) the headers and image file from https://queuecumber.gitlab.io/linux-acs-override/
- use ```sudo dpkg -i <file-name.deb>``` on both files then reboot
- in /etc/default/grub GRUB_CMDLINE_LINUX_DEFAULT add ```pcie_acs_override=downstream```
- Update grub ```sudo update-grub```
- reboot and use```sudo uname -r``` to verify the new ACS patched kernel is running

**Vulkan:** Because we want the host to be functional as well for everyday tasks and some gaming.
- With both Manjaro and Mint, Steam games on linux would occasionally not run due to not having vulkan natively installed (Dota Underlords). You may only encounter this issue with gfx cards that are fringe vulkan cards (Firepro M5100 and R9 290)
- in etc/default/grub GRUB_CMDLINE_LINUX_DEFAULT add ```radeon.cik_support=0 amdgpu.cik_support=1 radeon.si_support=0 amdgpu.si_support=1```
- add a PPA to your system ```sudo add-apt-repository ppa:oibaf/graphics-drivers```
- install from ppa ```sudo apt-get install libvulkan1 mesa-vulkan-drivers vulkan-utils```

**Step 2: Separating Passthrough Devices:**
- For Mnajaro, I used the the Arch Wiki, including the part about loading the vfio-pci early: https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Isolating_the_GPU
  - Rather than passing the the IDs in grub, I defined defined them in ```/etc/modprobe.d/vfio.conf``` only. 
- For Mint, I used the site: https://mathiashueber.com/windows-virtual-machine-gpu-passthrough-ubuntu/

- To summarize: 
  - In ```sudo nano /etc/initramfs-tools/modules``` add: ```vfio vfio_iommu vfio_virqfd frio_pci ids=abcd:1234,abcd:5678``` where abcd:1234... are the IDs you want to passthrough (determined from ```iommu.sh```)
  - In ```sudo nano /etc/modules``` add: ```vfio vfio_iommu_type1 vfio_pci ids=abcd:1234,abcd:5678```
  - Despite these changes, when running ```lspci -nnv``` the Nvidia GPU was still loading it's driver rather than the vfio one. In ```sudo nano /etc/modprobe.d/nvidia.conf``` add: ```softdep nouveau pre: vfio-pci softdep nvidia pre: vfio-pci softdep nvidia* pre: vfio-pci```
  - In ```sudo nano /etc/modprobe.d/vfio.conf``` add: ```options vfio-pci ids=abcd:1234,abcd:5678```
  - In ```sudo nano /etc/modprobe.d/kvm.conf``` add: ```options kvm ignore_msrs=1``` (needed for Win 10 after update 1803.
  - Update iniramfs ```sudo update-initramfs -u``` and reboot.
- Aside from the passed through gpu not displaying any output, using ```lspci -nnv``` will show that the vfio-pci is the kernel driver in use. However: the passed through USB card on my system works for the host until the VM is started.

**Step 3: Installing qemu/virt-manager**
