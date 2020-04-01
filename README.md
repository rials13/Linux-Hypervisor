# Linux-Hypervisor
This is a section of notes for setting up a linux host to run qemu/kvm/libvirt with both a windows and a macos vm with passthrough.  I know there are a lot of guides out there (all of my knowledge was from Googling how to do something) so this is as much a refference for me and to highliht a few things that some other guides don't include - all in one place.

**Step 0: Choose your OS.**
- I started with Manjaro because things worked on it with other guides but ended up on Mint because I switched after having trouble with the UEFI bios in Virt-manager (but also had trouble in Mint).
- It's worth noting that an ACS override is easier on Manjaro with the AUR.

-My config:
Mobo + processor: Gigabyte z97x-SOC-CF with i7-4790
- 16x Slot 1: MSI GTX970 - passed through to VMs
- 1x Slot: Empty
- 16x Slot 2: Empty
- 16x Slot 3: R9 290 - primary (original plan was to pass AMD card through but had trouble getting MacOS to work with this card)
- 16x Slot 4: Fresco Logic FL1100 USB 3.0 - passed through to VMs

- Corsair F120 for boot ssd and host OS
- Intel SSD for VM storage

This motherboard does not properly separate iommu groups so ACS override patch it required.

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
- use ```sudo dpkg -i <file-name.deb>``` on both files
- in /etc/default/grub GRUB_CMDLINE_LINUX_DEFAULT add ```pcie_acs_override=downstream```
- Update grub ```sudo update-grub```
- reboot and use```sudo uname -r``` to verify the new ACS patched kernel is running

**OPTIONAL Vulkan:** Because we want the host to be functional as well for everyday tasks and some gaming.
- With both Manjaro and Mint, Steam games on linux would occasionally not run due to not having vulkan natively installed (Dota Underlords). You may only encounter this issue with gfx cards that are older???
- in etc/default/grub GRUB_CMDLINE_LINUX_DEFAULT add ```radeon.cik_support=0 amdgpu.cik_support=1 radeon.si_support=0 amdgpu.si_support=1``` and ```sudo update-grub```
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
- Most of this was taken from Passthrough Post's guide: https://passthroughpo.st/new-and-improved-mac-os-tutorial-part-1-the-basics/
- Manjaro:
  - ```sudo pacman -S qemu python python-pip git```
  - ```pip install click request```
  - Install Virt-Manger
- Mint: (Note: on my Mint install, because I already had the files for MacOS VM, I skipped the pip installs.)
  - ```sudo apt-get install qemu python python-pip git```
  - ```sudo apt-get install python3-pip```
  - ```pip install click request```
  - On Ubuntu (didn't have this problem for Mint) I also had to install ```qemu-system-x86``` and ```qemu-utils```
  - ```sudo apt-get install virt-manager```
- Problems encountered with Virt-Manager:
  - Starting the network: ```sudo virsh net-start default```
    - I got the "firewalld backend" error. Installed ```ebtables``` and ```dnsmasq``` and it worked.
    - I had to add my username to the ```libvirt``` group in order to resolve some permissions issues. Also chown-ed the Intel SSD to my username. ```groups``` to check what group you are in.  ```Sudo usermod -a -G username libvirt``` Note: this only worked after a reboot following the install of virt-manager because libvirt was not a group yet
  - UEFI files: I tried making a Windows 7/10 vm and the UEFI was not found (not a problem in MacOS because Passthrough Post provides the UEFI files in their folder). This was a problem in both Mint and Manjaro.
    - ```/etc/libvirt/qemu.conf``` has an "nvram" section. Look at the section and confirm the path (note that this does not have to be uncommented it just describes the default behavior). My path was ```/usr/share/OVMF/...```
    - Looking at that directory, the only file was ```OVMF.fd``` nvram was looking for ```OVMF_CODE.fd``` and ```OVMF_VARS.fd```
    - Through a lot of Googling, I stumbled upon ```https://www.kraxel.org/repos/jenkins/edk2/?C=N;O=A``` and downloaded the ```edk2.git-ovmf-x64...rpm``` file.
    - Inside the package, there is a directory ```/ovmf/usr/share/edk2.git/ovmf-x64``` with the files ```OVMF_CODE-pure-efi.fd``` and ```OVMF_VARS-pure-efi.fd``` I renamed them to ```OVMF_CODE.fd``` and ```OVMF_VARS.fd``` and moved them to my ```/usr/share/OVMF/``` folder.



  - ```git clone https://github.com/foxlet/macOS-Simple-KVM.git```
  - ```cd macOS-Simple-KVM```
  - ```./jumpstart.sh (optional --high-sierra, --mojave)``` I did ```./jumpstart.sh --high-sierra``` on account of having an Nvidia graphics card.
  - ```qemu-img create -f qcow2 MyDisk.qcow2 XXG``` where XX is the amount in gigabytes you want the drive to be. (this is where I needed qemu-utils on Ubuntu)
  - Add these to bottom of ```basic.sh:``` 
  ```
    -drive id=SystemDisk,if=none,file=MyDisk.qcow2 \
    -device ide-hd,bus=sata.4,drive=SystemDisk \
  ```
